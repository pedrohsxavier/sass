# `@forward`

The `@forward` rule loads a [module][] from a URL and adds its members to the
public API of the current module without making them available to use within the
current stylesheet.

[module]: ../modules.md#module

## Table of Contents

* [Syntax](#syntax)

## Syntax

The grammar for the `@forward` rule is as follows:

<x><pre>
**ForwardRule** ::= '@forward' QuotedString AsClause? (ShowClause | HideClause)?
**AsClause**    ::= 'as' Identifier '*'
**ShowClause**  ::= 'show' MemberName (',' MemberName)*
**HideClause**  ::= 'hide' MemberName (',' MemberName)*
**MemberName**  ::= '$'? Identifier
</pre></x>

`@forward` rules must be at the top level of the document, and must come before
any rules other than `@charset` or `@use`. The `QuotedString`'s contents, known
as the rule's *URL*, must be a [valid URL string][] (for non-[special][] base
URL). No whitespace is allowed after `$` in `MemberName`, or before `*` in
`AsClause`.

[valid URL string]: https://url.spec.whatwg.org/#valid-url-string
[special]: https://url.spec.whatwg.org/#special-scheme
