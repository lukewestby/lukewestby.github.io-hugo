+++
date = "2016-11-25T21:32:28.939Z"
title = "Runtime Expectations as Package Constraint Dimensions"
linktitle = "Runtime Expectations as Package Constraint Dimensions"
weight = 1
description = "I don't think there's enough conversation around what it fully means for a package or library for a given programming language to be compatible with the environment of its users. Package versions are the obvious primary designator of compatibility, and I'm grateful for the work that has gone into Semantic Versioning to formalize them. But maybe there's more formalization to be done."
tags = [
  "versions",
  "compatibility",
  "env",
  "package managers"
]
+++

Take a look at this shortened version of the package.json file for knex,
a SQL query builder and driver for popular SQL databases in JavaScript.

> Before proceeding I want to take a second and let you know that knex is one
of my favorite libraries. I absolutely recommend it and want to make sure it's
clear that none of this post should be taken as criticism of knex or its
author, [Tim Griesser](https://twitter.com/tgriesser). I chose it (a) because it happens to be a good example of something that
could benefit from dealing with the larger issue in question and (b) so that
people who read this will know that it exists and check it out.

```json
{
  "name": "knex",
  "version": "0.12.6",
  "description": "A batteries-included SQL query & schema builder for Postgres, MySQL and SQLite3 and the Browser",
  "main": "knex.js",
  "dependencies": {...},
  "devDependencies": {...},
  "buildDependencies": [...],
  "scripts": {...},
  "bin": {
    "knex": "./bin/cli.js"
  },
  "repository": {
    "type": "git",
    "url": "git://github.com/tgriesser/knex.git"
  },
  "keywords": [...],
  "author": {
    "name": "Tim Griesser",
    "web": "https://github.com/tgriesser"
  },
  "browser": {
    "./lib/migrate/index.js": "./lib/util/noop.js",
    "./lib/bin/cli.js": "./lib/util/noop.js",
    "./lib/seed/index.js": "./lib/util/noop.js",
    "mssql": false,
    "mysql": false,
    "mysql2": false,
    "mariasql": false,
    "pg": false,
    "pg-query-stream": false,
    "oracle": false,
    "strong-oracle": false,
    "sqlite3": false,
    "oracledb": false
  },
  "files": [...],
  "license": "MIT"
}
```

After reading through this file, what can you explain about where this library
is meant to be used? We installed it from NPM, and the description and
dependencies (redacted) mention things like SQLite, so we can assume it's meant
for node. From there we can guess that it works in all versions of node, because
there is no `engines` field in the package.json describing which versions. We
can also learn that knex runs in the browser because it is mentioned in the
description, and we have the `browser` field, which, while
[standardized](https://github.com/defunctzombie/package-browser-field-spec) on
its own, is a separate effort from those governing the rest of the package.json
format. We also don't know which browsers this library will work in. If we
read the documentation for knex, we learn that we can use it to work with
browsers that support Web SQL, which is now [no longer an active specification,
not fully supported, and may disappear from future browser versions](http://caniuse.com/#feat=sql-storage).

Anyway, between the package.json, the library's documentation, and documentation
for different platforms mentioned in the docs we can figure out where we can
reasonably expect to see our lives improved by using knex. So what's the
problem?

**Tools like npm and webpack aren't going to go read the knex docs, nor are they
going to read caniuse.com and MDN and figure out if knex will even work in the
environments we might be targeting.**

So why don't we just write all this stuff down somewhere for our tools to read?
We do this with semantic versioning and it totally works! We agree that we
should stop writing versions for people and start writing them for machines so
that our tools can let us know before we even attempt to use them whether our
libraries are compatible with each other. Why not do the same with
where we want them to run?

This idea found its way into my head almost a year ago when I began using
[Elm](http://elm-lang.org). I was interested in expanding the Elm platform to
Electron so that folks could get the benefits of Elm in their desktop apps.
This seemed to fit into the larger narrative at the time of how Elm was going
to support the web platform in general, and I asked a question. Elm's creator,
[Evan Czaplicki](https://twitter.com/czaplic), gave a really interesting answer that I've been thinking about
ever since.

> Luke: Will the idea of a set of platform bindings for "the web platform"
extend to similarly embedded bindings for other environments where Elm might
thrive as a compile-to-js language? The specific example I mean to reference is
Electron, in my opinion an incredible use-case for Elm, which ships with
embedded APIs that augment the chromium wrapper and are not implementable in
Elm. I've begun working on these bindings at https://github.com/elm-electron and
I'm not sure from the above what the future status of this project might be
since it doesn't quite fit into one side or the other (web platform vs.
rewritable in Elm).

> Evan: Luke, I agree that that is an important thing to
support. Other examples include elm-webgl and node. Basically other "platforms"
in my mind. This gets us into a world where a certain package may only work on
certain "platforms", so it is not enough to solve constraints, we need to label
each package with the platforms it needs. So I'd like to get one platform sorted
out and see what that looks like before building in language support for many
platforms. I think those are two separate and difficult problems, and if I try
both at once, I'm probably going to have a bad time. If there is more to say,
let's discuss it in a separate thread.

> [Source](https://groups.google.com/d/msg/elm-dev/1JW6wknkDIo/rrnCs3kqCQAJ)

This thread is now very outdated and much of the greater context
has since changed. But the idea has stuck with me and I do think there
is more to say, but not just with regard to Elm. To me this is best exemplified
by a different feature of Elm: enforcement of constraints on things from types
in code to package versions prior to execution.

That is to say: if a thing isn't going to work, and my tools can find out about
that before I try and do said thing, I want them to tell me right then. We get
this with Elm's type system, and we get it with Elm's package manager when it
comes to package versions. We can't run broken code because the compiler tells
us, not the runtime. And we can't install packages in a broken way because the
package manager tells us, not the runtime. In my opinion, this is one of the
most important design features of the Elm platform and an enormous source of
productivity. Just let the compiler guide you to correct code!

So why can I install `elm-lang/mouse`, use it in an Elm program, compile that
program, run it in node, and see lots of errors at runtime? Well, because Elm
isn't meant for node. It just happens to compile to JS and I can run JS in node.
But more to the point, there's nothing built into either node or elm-package to
say "this code isn't meant to work here".

And now back to knex. Why can I install knex, add it to a webpack bundle to do
Web SQL stuff, run it in Firefox, and watch it fail at runtime? It's not knex's
fault that Firefox doesn't support Web SQL, nor is it necessarily the
responsibility of the maintainer to clarify and document it. The knex library
doesn't depend on Web SQL, but you can use it that way.

I think the problem is that we don't offer enough information to our tools to
let them catch our mistakes. We can't ask the maintainer of knex to add code to
detect when it's being used in a browser that doesn't support Web SQL. And we
also can't ask the maintainers of webpack to develop NLP features so that
webpack can read our libraries' docs and determine if they will run. What we
need is to meet in the middle. It should be no more difficult for a library
author to declare compatibility than doing just that. Additional dimensions
beyond package version, in the spirit of semantic versioning, that make it clear
where something is meant to be used.

Before proposing anything I'll set the constraint that I'm really only thinking
about solutions for JavaScript and things that use JavaScript as a VM. My reason
is that these are the things I work on, and I don't know enough about other
stuff to know if this is even a problem or how it should be solved there. Let's
just think about JS.

Step one could be a standard way of expressing a fully qualified environment.
This includes dimensions like browser vs. server. It includes things like which
browser vendor and browser version. It includes things like which server runtime
and which runtime version. It includes a way to register all of these things so
that if one day there ended up being a server runtime with the same name as a
browser there would be no conflicts. Lastly, it also includes augmented and
combined derivations of things like browser and server, so that things like
Electron can declare compatibility with the embedded node or Chromium version
as well as their own APIs.

Step two could be to determine where this information needs to be declared.
Does it go in the package.json or elm-package.json? Does it go in the webpack
config? Does it get embedded in transpilation results or directly in source code
like a `"use"` pragma? Wherever they go, they need to be in a place where our
package managers can use them in their constraint solvers and our preprocessors
can use them in their error reporting. Maybe putting them somewhere accessible
by VMs could be asking too much, but I don't work on browsers or standards
committees at the moment so I don't really know.

Semantic versioning does a great job of helping our tools understand the _what_
of our library combinations. Let's not stop there. Let's figure out a good way
of adding more dimensions than just version, to deal with the _where_.
Share your thoughts with me!
