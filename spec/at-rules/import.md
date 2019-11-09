# Import

Imports are the fundamental mechanism for sharing code in Sass. This spec
describes the algorithm for [handling an `@import`
rule](#handling-an-import-rule), with the exception that it doesn't cover in
detail the behavior of user- or implementation-defined importers. An importer
should be considered to effectively replace the algorithm for [loading an
import](#loading-an-import), possibly with another algorithm that calls the
algorithm listed below to handle filesystem imports.

This spec also defines the algorithm for [loading an entrypoint
path](#loading-an-entrypoint-path), which defines how a Sass implementation
should compile a file passed on the command line or through a programming
language API.

## Semantics

To evaluate an `@import` rule:

* For each of that rule's arguments:

  * If any of the following are true, the argument is considered "plain CSS":

    * The imported URL begins with `http://` or `https://`.
    * The imported URL ends with `.css`.
    * The imported URL is syntactically defined as a `url()`.
    * The argument has a media query and/or a supports query.

    > Note that this means that imports that explicitly end with `.css` are
    > treated as plain CSS `@import` rules, rather than importing stylesheets as
    > CSS.

  * If the argument is "plain CSS":

    * Evaluate any interpolation it contains.

    * Add an `@import` with the evaluated string, media query, and/or supports
      query to the CSS AST.

  * Otherwise, let `stylesheet` be the result of
    [loading the imported string](#loading-an-import).

    If this returns null, throw an error.

  * If an AST with the same [canonical URL][] as `stylesheet` is currently being
    evaluated, throw an error.

    [canonical URL]: #canonical-url-of-a-stylesheet

  * Evaluate `stylesheet` in the global scope.
