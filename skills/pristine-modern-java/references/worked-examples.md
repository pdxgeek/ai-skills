# Worked examples

Real refactors, genericized. Each shows the shape to hunt for and the shape to produce,
with the reasoning that decided it. These all preserved behavior exactly, and that was
verified before committing, not assumed.

## 1. Guard plus orElse(null) into a single chain

Hunt for: `findOne(...).orElse(null)`, a null check, an early return, then work.

```java
// Before
EntityDTO dto = service.findOne(id).orElse(null);
if (dto == null) {
    return null;
}
if (!checker.matches(id, secret)) {
    return null;
}
return dto.getUserId();

// After
return service.findOne(id)
        .filter(entity -> checker.matches(id, secret))
        .map(EntityDTO::getUserId)
        .orElse(null);
```

A `filter` predicate that ignores the element is fine when it reads as an authorization
or precondition gate. The chain communicates "value flows through checks" better than
three exits.

## 2. Start the chain from the nullable input

Hunt for: a null guard on a parameter followed by a chain that could have absorbed it.

```java
// Before
if (accountHolderId == null) {
    return Optional.empty();
}
return Optional.ofNullable(api.createSession(accountHolderId, flag))
        .map(session -> new SessionDTO(accountHolderId, session.getClientSecret()));

// After
return Optional.ofNullable(accountHolderId)
        .map(holderId -> api.createSession(holderId, flag))
        .map(session -> new SessionDTO(accountHolderId, session.getClientSecret()));
```

`map` already turns a null mapper result into empty, so the intermediate call that "might
return null" needs no extra guard.

## 3. Keep a branch-specific log with or()

Hunt for: two early returns that would collapse into one chain except one of them logs.

```java
// Before
if (property == null) {
    return Optional.empty();
}
PayoutSettingDTO setting = property.getPayoutSetting();
if (setting == null) {
    log.warn("Payout setting missing for property {}", property.getId());
    return Optional.empty();
}
// ... proceed with setting

// After
return Optional.ofNullable(property)
        .flatMap(prop -> Optional.ofNullable(prop.getPayoutSetting()).or(() -> {
            log.warn("Payout setting missing for property {}", prop.getId());
            return Optional.empty();
        }))
        .flatMap(setting -> { /* proceed */ });
```

The two kinds of empty behaved differently (one warned), and that difference is part of
the behavior. Do not merge distinct empties silently; carry the distinction in `.or()`
or keep the guard.

## 4. Extract a helper to absorb try/catch (checked exceptions)

Hunt for: a chain conversion blocked because a callee throws a checked exception, which
cannot escape a lambda.

```java
// Before: forced out of a chain by the checked exception
EntityDTO dto = service.findOne(id).orElse(null);
if (dto == null) {
    return Optional.empty();
}
Customer customer;
try {
    customer = api.getCustomerOrCreate(dto, user);   // throws StripeException
} catch (StripeException e) {
    throw new BadRequestAlertException(e.getUserMessage(), ENTITY, "errorKey");
}
return buildSession(customer.getId());

// After
return service.findOne(id)
        .flatMap(dto -> buildSession(requireCustomer(dto).getId()));

private Customer requireCustomer(EntityDTO dto) {
    try {
        return api.getCustomerOrCreate(dto, resolveUser(dto));
    } catch (StripeException e) {
        throw new BadRequestAlertException(e.getUserMessage(), ENTITY, "errorKey");
    } catch (Exception e) {
        // The interface declares throws Exception, but the implementation only throws StripeException.
        throw new IllegalStateException(e);
    }
}
```

The helper translates the checked failure into the domain's runtime exception once, and
every caller gets to stay inline. Before writing the broad catch, read the callee: if its
`throws Exception` is wider than what the body can actually throw, say so in a comment
and treat the impossible arm as a defect (`IllegalStateException`), never as a swallowed
error. Do not silently broaden what a catch converts; that changes behavior.

## 5. Turn a pre-check into a filter

Hunt for: an if that returns empty before a terminal ofNullable.

```java
// Before
if (SecurityUtils.isCurrentUserInRole(ADMIN_AUTHORITIES)) {
    return Optional.empty();
}
return Optional.ofNullable(userService.getCurrentUser());

// After
return Optional.ofNullable(userService.getCurrentUser())
        .filter(caller -> !SecurityUtils.isCurrentUserInRole(ADMIN_AUTHORITIES));
```

Note the evaluation order flips: the lookup now runs before the role check. That was fine
here because the lookup is a cheap in-memory read. When the reordered call is expensive or
has side effects, keep the guard.

## 6. Existence checks belong in the query, not the stream

Hunt for: `!CollectionUtils.isEmpty(repository.findAllBy...(...))` or
`findAllBy...(...).size() > 0` or `.stream().anyMatch(...)` over a full fetch.

```java
// Before
boolean granted = !CollectionUtils.isEmpty(
        repository.findAllBySecretKeyAndApplicationIdAndActiveIsTrue(secret, appId));

// After: derived exists query on the repository
boolean granted = repository.existsBySecretKeyAndApplicationIdAndActiveIsTrue(secret, appId);
```

Spring Data list queries never return null, so the null half of `CollectionUtils.isEmpty`
was dead; and materializing rows to test existence pays for a fetch the database could
answer with EXISTS. The stream-side lesson generalizes: push filtering and existence into
the source when the source is a database.

## 7. A private record for an intermediate result

Hunt for: parallel local variables, or a chain that needs to carry two values.

```java
public enum Source { APPLICATION, APPLICATION_PAYMENT, DEFAULT_CLAIM }

private record Resolution(Source source, Long applicationId) { }

private Optional<Resolution> resolve(String secretKey) {
    return applicationRepository.findBySecret(secretKey)
            .map(app -> new Resolution(Source.APPLICATION, app.getId()))
            .or(() -> paymentRepository.findBySecret(secretKey)
                    .map(payment -> new Resolution(Source.APPLICATION_PAYMENT, payment.getApplicationId())))
            .or(() -> claimRepository.findBySecret(secretKey)
                    .map(claim -> new Resolution(Source.DEFAULT_CLAIM, claim.getApplicationId())));
}
```

The record names the pair, costs three lines, and lets the chain return one value instead
of contorting around two. Keep it private and nested when only one class cares.

## 8. min with a Comparator instead of sort-and-take-first

```java
// The deterministic "pick one" from a collection
return userRepository.findAllByPropertyId(propertyId).stream()
        .min(Comparator
                .comparing((User user) -> StringUtils.defaultString(user.getFirstName()), String.CASE_INSENSITIVE_ORDER)
                .thenComparing(user -> StringUtils.defaultString(user.getLastName()), String.CASE_INSENSITIVE_ORDER)
                .thenComparing(User::getId))
        .or(this::fallbackUser)
        .map(user -> StringUtils.defaultIfBlank(user.getEmail(), user.getLogin()))
        .orElse(null);
```

`min` with a composed comparator replaces sort-then-get(0), never throws on empty, and the
tie-breaking `thenComparing(User::getId)` is what makes the pick stable across runs. The
`orElse(null)` at the end is a boundary concession: the consumer is a legacy API that takes
a nullable value. Inside your own code, keep the Optional.

## 9. Delete redundant arms instead of chaining them

Not every cleanup is a conversion. One authorization check ORed a local repository lookup
with a call to the central checker; the local lookup queried a strict subset of what the
checker already covered, so `local || checker` reduced to `checker` in every legitimate
state. The fix was deletion, and the chain got simpler for free. Before converting a
compound condition, ask whether one side subsumes the other.
