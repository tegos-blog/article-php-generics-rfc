# PHP Generics Already Exist — They're Just Hidden in PHPDoc

![PHP Generics Already Exist — They're Just Hidden in PHPDoc](assets/poster.jpg)

Every Laravel dev has already written PHP generics — we just hid them inside `@template` and `Collection<…>` comments and pretended it didn't count. A new RFC wants to make them real syntax.

I grepped a Laravel project I work on and found 143 generic-PHPDoc annotations and zero of my own `@template`. Here's the quick theory, why PHP has fought generics for a decade, and what the Bound-Erased Generic Types RFC actually changes.

## What's Inside

- A 30-second refresh: what a generic is (`Box<T>`, parametric polymorphism) and where the idea came from
- Reified vs erased generics — the difference that decides everything
- Why PHP keeps stalling: expensive runtime checks, ~6 analyzers with no shared spec, RFCs since 2014
- The new native syntax — bounds, defaults, variance, optional turbofish `::<…>`
- A real audit: 143 generic annotations, 0 own `@template`, all consumed via doc blocks
- Where the RFC stands (Under Discussion, v0.22) and why I want it shipped in 2026

## 📎 Read Full

[PHP Generics Already Exist — They're Just Hidden in PHPDoc](https://dev.to/tegos/PLACEHOLDER)
