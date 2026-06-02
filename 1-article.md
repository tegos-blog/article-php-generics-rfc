Every Laravel dev has written PHP generics. You just wrote them inside a comment and pretended it didn't count.

```php
/** @return Collection<CartItemDTO> */
```

That line is a generic type. PHPStan reads it, your IDE reads it, but PHP itself shrugs and ignores it. A new RFC, [Bound-Erased Generic Types](https://wiki.php.net/rfc/bound_erased_generic_types), wants to turn those comments into real syntax. Let's start with what a generic even is, then look at why PHP has fought this for a decade.

## A 30-Second Theory Refresh

A **generic** is a type with a hole in it. Instead of writing `IntBox`, `StringBox`, and `UserBox`, you write `Box<T>` once and decide `T` later. The formal name is **parametric polymorphism**.

```php
// Without generics: one class per type, copy-paste forever
final class IntBox { public function get(): int { /* ... */ } }

// With generics: one class, T decided at use site
final class Box<T> { public function get(): T { /* ... */ } }
```

The idea isn't new. Java, C#, and TypeScript all have it. PHP and JS skipped it for years, which is exactly why static analyzers grew up to fill the gap.

There are two ways to ship generics. **Reified** keeps the type at runtime, so `Box<int>` and `Box<string>` stay distinct (C#). **Erased** uses it only for static analysis and throws it away before execution (Java, TypeScript). This RFC picks erasure. Hold that thought.

## Why It's So Hard to Add to PHP

Generics have been on the PHP internals agenda since **January 2014**. Multiple RFCs, multiple implementations, none merged. Why?

- **Runtime type checks are expensive.** Reified generics mean PHP must store and verify type arguments on every instance. Nikita Popov built a prototype and hit superlinear type-checking cost and heavy per-instance memory. PHP is request-per-process — you pay that on every request.
- **No shared spec.** PHPStan, Psalm, Mago, and PhpStorm have spent years guessing at the same `@template` syntax, each a little differently. There's no official standard to follow, so an edge case that works in one tool can quietly break in another.
- **A decade of stalls.** The 2016 reified RFC sat in draft for ten years. The 2024 reified continuation stalled on cross-file inference. Each attempt paid the syntax cost and shipped no syntax.

## The Solution: Erase, Don't Reify

The RFC, by Seifeddine Gmati (author of the Mago analyzer), adds native syntax: `class Box<T>`, bounds `<T : Animal>`, defaults `<K = string>`, variance `<+T>` / `<-T>`, and an optional turbofish `::<…>`.

The key word is **bound-erased**. At runtime `Box<int>` and `Box<string>` are the same class — the type argument is erased down to its bound (or `mixed`). Checking happens statically, like Java or TypeScript. Reified was rejected on purpose: as the RFC puts it, *"we don't need runtime type checks."* Generics are a static-analysis tool, not a runtime one.

So your comment becomes the signature, and old code keeps working — the turbofish is optional everywhere:

```php
// Today: the type parameter lives in a doc block
/**
 * @template T
 */
interface Repository
{
    /** @return Collection<T> */
    public function all(): Collection;
}

// With the RFC: the same intent, but the language reads it
interface Repository<T>
{
    public function all(): Collection<T>;
}
```

## It's Not New — Just Look at Your Own Code

I grepped a Laravel 10 project I work on (PHP 8.1, Larastan level 5). The result: **143 generic-PHPDoc annotations and 0 of my own `@template`**. So the project *consumes* generics everywhere (mostly `Collection<…>`) but never declares them. Here's a real action from it:

```php
final class CartIndexAction implements Actionable
{
    /** @return Collection<CartItemDTO> */
    public function handle(CartItemParamDTO $param, int $userId): Collection
    {
        return $this->cartItemService->getForUser($userId, $param->itemUuids)->values();
    }
}
```

The native return type is just `Collection`. The useful part, `CartItemDTO`, lives in a comment. Drop it and autocomplete dies and Larastan stops catching wrong-type bugs. It's worse in repositories, where you babysit the analyzer by hand:

```php
public function getUserCarts(int $userId): EloquentCollection
{
    /** @var EloquentCollection<Cart> $carts */
    $carts = Cart::query()->where('user_id', $userId)->get();

    return $carts;
}
```

That `/** @var */` exists only to feed the analyzer. And it's not just my code: Laravel itself ships generic annotations across its core classes and IDE-helper stubs. Every `Collection` and query `Builder` you touch is already generic in the doc blocks. The RFC authors counted [200k+ files](https://wiki.php.net/rfc/bound_erased_generic_types) on GitHub using `@template`. Adopting this RFC doesn't *introduce* generics. They've been here for a decade. It formalizes what's already there.

## I Want This in 2026

Heads up: the RFC is still *Under Discussion* (v0.22), with an [open php-src PR](https://github.com/php/php-src/pull/21969) that works today. The blocker isn't the code. It's a small group on internals who don't love the erased approach.

We've babysat doc blocks and worked around analyzer quirks for years because generics are worth it. The implementation is ready, the syntax is proven, and the demand has been there for a decade. PHP developers already write generics every day. The only question left is whether they keep living in comments, or finally become part of the language. I know which one I want in 2026.

## TL;DR

- A generic is a type with a hole — `Box<T>` instead of one class per type. Old idea (Java, C#, TS).
- PHP generics already exist in `@template` / `<…>` PHPDoc your analyzer reads. 200k+ files on GitHub use them.
- Adding them natively is hard: reified runtime checks are expensive, and ~6 analyzers disagree with no shared spec. On the agenda since 2014.
- The RFC picks **bound erasure**: checked statically, erased at runtime, `Box<int>` ≡ `Box<string>`, zero runtime cost.
- Old code keeps working; the turbofish `::<…>` is optional everywhere.

💡 Watch Brent Roose's [video walkthrough](https://www.youtube.com/watch?v=4wpW98S2xJQ) and read the [RFC](https://wiki.php.net/rfc/bound_erased_generic_types) before the vote.

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Laravel, after the happy path.**
