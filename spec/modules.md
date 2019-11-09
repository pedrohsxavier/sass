# Modules

## Table of Contents

* [Definitions](#defintions)
  * [Member](#member)
  * [CSS Tree](#css-tree)
  * [Configuration](#configuration)
  * [Module](#module)
  * [Module Graph](#module-graph)
  * [Import Context](#import-context)
  * [Basename](#basename)
  * [Dirname](#dirname)
* [Syntax](#syntax)
* [Procedures](#procedures)
  * [Loading a Module](#loading-a-module)
  * [Loading a Source File](#loading-a-source-file)
  * [Resolving a `file:` URL](#resolving-a-file-url)
  * [Resolving a `file:` URL for Partials](#resolving-a-file-url-for-partials)

## Definitions

### Member

A *member* is a Sass construct that's defined either by the user or the
implementation and is identified by a Sass identifier. This currently includes
[variables](variables.md), mixins, and functions (but *not* placeholder
selectors). All members have definitions associated with them, whose specific
structure depends on the type of the given member.

> Each member type has its own namespace in Sass, so for example the mixin
> `name` doesn't conflict with the function `name` or the variable `$name`. 

### CSS Tree

A *CSS tree* is an abstract CSS syntax tree. It has multiple top-level CSS
statements like at-rules or style rules. The ordering of these statements is
significant. A CSS tree cannot contain any Sass-specific constructs, with the
notable exception of placeholder selectors.

An *empty CSS tree* contains no statements.

### Configuration

A *configuration* is a map from [variable](variables.md) names to SassScript
values. It's used when [executing](#executing-files) a [source file][] to
customize its execution. An *empty configuration* contains no entries.

[source file]: syntax.md#source-file

### Module

A *module* is a collection of various properties:

* A set of [members](#member) that contains at most one member of any given type
  and name.
  
  > For example, a module may not have two variables named `$name`, although it
  > may contain a function and a mixin with the same name or two functions with
  > different names.
  
  > The names (and mixin and function signatures) of a module's members are
  > static, and can be determined without executing its associated source file.
  > This means that any possible module for a given source file has the same
  > member names and signatures regardless of the context in which those modules
  > are loaded.

* A set of [extensions](???).

* A [CSS tree](???).

  > This tree is empty for [built-in modules](#built-in modules) and
  > user-defined modules that only define variables, functions, and mixins
  > without including any plain CSS rules.

* A set of references to other modules, known as the module's *dependencies*.

  > These are the modules referred to through [`@use` rules][] and/or
  > [`@forward` rules][]

  [`@use` rules]: at-rules/use.md
  [`@forward` rules]: at-rules/forward.md

* An optional [source file][].

  > Note that [built-in modules](#built-in-modules) *do not* have source files
  > associated with them.

* An absolute URL, known as the module's *canonical URL*. If the module has a
  source file, this must be the same as the source file's canonical URL.

Once a user-defined module has been returned by [Executing
Files](#executing-files), it is immutable except for its variable values.
[Built-in modules](#built-in-modules) are always immutable.

### Module Graph

The set of [modules](#module) loaded in the course of processing a stylesheet
can be construed as a [directed acyclic graph][] where the vertices are modules
and the edges are [`@use` rules][] and/or [`@forward` rules][]. We call this the
*module graph*.

[directed acyclic graph]: https://en.wikipedia.org/wiki/Directed_acyclic_graph

The module graph is not allowed to contain cycles because they make it
impossible to guarantee that all dependencies of a module are available before
that module is loaded. Although the names and APIs of a module's members can be
determined without [executing](#executing-files) it, Sass allows code to be
evaluated while loading a module, so those members may not behave correctly when
invoked before the module is executed.

### Import Context

An *import context* is a set of [members](#member) that contains at most one
member of any given type and name. It's always mutable.

> Import contexts serve as glue between the old [`@import` rule][] and the
> module system. It serves as a shared global namespace for stylesheets loaded
> using `@import` rules, while also preventing global names from leaking into or
> out of stylesheets loaded using [`@use` rules][] and/or [`@forward` rules][].

[`@import rule]: at-rules/import.md

### Basename

The *basename* of a URL is the final component of that URL's path.

### Dirname

The *dirname* of a URL is the prefix of that URL up to, but not including, the
beginning of its [basename](#basename).

## Syntax

The module system defines the following syntax for referring to names from other
modules:

<x><pre>
**PublicIdentifier**     ::= [\<ident-token>][] that doesn't begin with '-' or '_'
**NamespacedIdentifier** ::= Identifier | Identifier '.' PublicIdentifier
</pre></x>

[\<ident-token>]: https://drafts.csswg.org/css-syntax-3/#ident-token-diagram

No whitespace is allowed before or after the `'.'` in `NamespacedIdentifier`.

## Procedures

### Loading a Module

This algorithm takes a URL `url` and [configuration](#configuration) `config`
and returns a [module](#module):

* If `url`'s scheme is `sass`:

  * If `config` is not empty, throw an error.

  * If a [built-in module](???) exists with the exact given URL, return it.

  * Otherwise, throw an error.

* Let `file` be the result of [loading the source file](#loading-a-source-file)
  at `url`.

* If `file` is null, throw an error.

* If `file` has already been [executed](#executing-a-file):

  * If `config` is not empty, throw an error.

  * Otherwise, return the module that execution produced.

* If `file` is currently being executed, throw an error.

  > This disallows circular `@use`s, which ensures that modules can't be used
  > until they're fully initialized.

* Otherwise, return the result of [executing](#executing-files) `file` with
  `config` and a new [import context](#import-context).

  > For simplicity, the spec creates an import context for every module.
  > Implementations are encouraged to avoid eagerly allocating resources for
  > imports, though, to make use-cases only involving `@use` more efficient.

### Loading a Source File

This algorithm takes a string, `argument`, and returns either a [source file][]
or null.

* If Sass is currently [executing](#executing-files) a source file, and the
  scheme of that source file's canonical URL is `file`:

  * Let `root` be the current source file's canonical URL.

* Otherwise, let `root` be `null`.

* Let `bases` be a list beginning with `root` if it's non-null, followed by the
  absolute `file:` URLs of all import paths.

* For each `base` in `bases`:

  * Let `url` be the result of [parsing `argument` as a URL][] with `base` as
    the base URL.

    [parsing `argument` as a URL]: https://url.spec.whatwg.org/#concept-url-parser

    If this returns a failure, throw that failure.

  * If `url`'s scheme is not `file`, return null.

  * Let `resolved` be the result of [resolving `url`](#resolving-a-file-url).

  * If `resolved` is null:

    * Let `index` be [`dirname(url)`](#dirname) + `"index/"` +
      [`basename(url)`](#basename).

    * Set `resolved` to the result of [resolving
      `index`](#resolving-a-file-url).

  * If `resolved` is still null, continue to the next loop.

  * Let `text` be the contents of the file at `resolved`.

  * Let `ast` be:

    * The result of parsing `text` as SCSS if `resolved` ends in `.scss`.
    * The result of parsing `text` as the indented syntax if `resolved` ends in
      `.sass`.
    * The result of [parsing `text` as CSS][] if `resolved` ends in `.css`.

    [parsing `text` as CSS]: syntax.md#parsing-text-as-css

    > The algorithm for [resolving a `file:` URL](#resolving-a-file-url)
    > guarantees that `resolved` will have one of these extensions.

  * Return a source file with `ast` as its abstract syntax tree and `resolved`
    as its canonical URL.

* Return null.

### Resolving a `file:` URL

This algorithm takes a URL, `url`, whose scheme must be `file` and returns
either another URL that's guaranteed to point to a file on disk or null.

* If `url` ends in `.scss` or `.sass`, return the result of
  [resolving `url` for partials][resolving for partials].

* Let `sass` be the result of
  [resolving `url` + `".sass"` for partials][resolving for partials].

* Let `scss` be the result of
  [resolving `url` + `".scss"` for partials][resolving for partials].

* If neither `sass` nor `scss` are null, throw an error.

* If exactly one of `sass` and `scss` is null, return the other one.

* Return the result of
  [resolving `url` + `".css"` for partials][resolving for partials].

[resolving for partials]: #resolving-a-file-url-for-partials

### Resolving a `file:` URL for Partials

This algorithm takes a URL, `url`, whose scheme must be `file` and returns
either another URL that's guaranteed to point to a file on disk or null.

* If `url`'s [basename](#basename) begins with `"_"`:

  * If a file exists on disk at `url`, return `url`.

    Otherwise return null.

* Let `partial` be [`dirname(url)`](#dirname) + `"_"` +
  [`basename(url)`](#basename).

* If a file exists on disk at both `url` and `partial`, throw an error.

* If a file exists on disk at `url`, return `url`.

* If a file exists on disk at `partial`, return `partial`.

* Return null.
