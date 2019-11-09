# `@extend`

The `@extend` at-rule is probably the single most complicated feature in Sass.
While its semantics are straightforward to describe, the implementation involves
many interacting layers and a lot of intricate case analysis.

## Table of Contents

* [Definitions](#definitions)
  * [Extender](#extender)
  * [Target](#target)
  * [Extendee](#extendee)
  * [The `extend()` Function](#the-extend-function)
* [Semantics](#semantics)
  * [Limitations](#limitations)
  * [Specificity](#specificity)
    * [The First Law](#the-first-law)
    * [The Second Law](#the-second-law)

## Definition

These definitions provide names to the various selectors involved with a given
use of `@extend`:

```scss
.extender {
  @extend .target;
}

// ┌─ extendee
.target {
  // ...
}
```

### Extender

An `@extend` rule's *extender* is the [selector list][] for the style rule in
which the `@extend` rule appears.

[selector list]: https://drafts.csswg.org/selectors-4/#selector-list

### Target

An `@extend` rule's *target* is the [simple selector][] that's used as an
argument to `@extend`.

[simple selector]: https://drafts.csswg.org/selectors-4/#simple

### Extendee

An *extendee* is a selector list being modified by an `@extend` rule. It's only
defined within the scope of a single application of a given `@extend` rule. If
an extendee contains that rule's [target](#target), it will usually be modified
to include the rule's [extender](#extender) as well.

### The `extend()` Function

As a shorthand, we use the function notation `extend(extendee, target,
extender)` to refer to the result of extending `extendee` with `extender
{@extend target}` (much like the Sass function `selector-extend()`).

## Semantics

The `@extend` rule means that all elements matching the [extender](#extender)
should be styled as though they match the [target](#target) as well.

### Limitations

It's not possible for a preprocessor to guarantee the semantics of `@extend` in
full generality. There are three major exceptions where implementations are not
required to meet the full definition.

1. Implementations should not try to apply native browser styles that would
   apply to the target. For example, while it's legal to write `@extend table`,
   there's no good way to apply browsers' built-in table styles.

2. Second, when the extender and the extendee both contain multiple compound
   selectors separated by combinators, implementations are allowed to assume
   that the elements matched by the extender's compound selectors are not
   interleaved with those matched by the extendee's compound selectors.

   For example, consider `extend(.c .x, .x, .a .b)`. Implementations must
   generate the selectors `.a .c .b` and `.c .a .b`, because an element with
   `class="a"` may be either outside or inside one with `class="c"`. However,
   implementations are not required to generate the selector `.a.c .b` which
   would require HTML with `class="a c"`.

   This flexiblity is allowed because otherwise implementations would have to
   generate a combinatorial explosion of selectors, the vast majority of which
   would be extremely unlikely to match real HTML. This particular heuristic
   assumes that the extender and extendee were each written with self-contained
   HTML in mind, so that interwoven HTML is unlikely to come up.

3. Implementations are not required to apply the target's styles with the exact
   same specificity as the extender, because this isn't generally possible when
   complex extendees exist. However, implementations must respect certain
   guarantees about specificity; see below for details.

### Specificity

When modifying the extendee during extension, the implementation must provide
two guarantees about the result. These are known as the "laws of extend".


#### The First Law

The first law of `@extend` says that the specificity of the first generated
selector must be greater than or equal to that of the original extendee. For
example, `extend(a.foo, .foo, .a)` should generate `a.foo, a` even though
`a.foo` matches a subset of elements matched by `a`.

In most cases, the first generated selector will be identical to the extendee,
but it may need to be modified when dealing with the pseudo-selector `:not()`.
For example, `extend(:not(.foo), .foo, .bar)` should produce
`:not(.foo):not(.bar)`.

#### The Second Law

The second law of extend says that the specificity of a new selector to match a
given extender must be greater than or equal to the specificity of that
extender. For example, `extend(a, a, a.foo)` should produce `a, a.foo` even
though (again) `a.foo` matches a subset of elements matched by `a`.

This still leaves room for optimizations. For example,
`extend(.bar a, a, a.foo)` can just produce `.bar a` (omitting `.bar a.foo`).
This is allowed because `.bar a` matches a superset of the elements matched by
`.bar a.foo`, *and* the specificity of `.bar a` is equal to that of the extender
`a.foo`.
