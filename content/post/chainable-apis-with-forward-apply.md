+++
date = "2016-02-27T11:36:18-06:00"
title = "Chainable APIs with the forward apply operator"
linktitle = "Chainable APIs with the forward apply operator"
weight = 1
description = "What do we do when configuration objects grow large or hard to understand? In this article we'll examine some cases where this happens and how to use the forward apply operator with some basic patterns create beautifully readable configuration builders."
tags = [
  "elm"
]
+++


The `(|>)` operator is, in my opinion, one of the most elegant features of the
Elm language. It lets us read function applications in the order the actually
happen and this readability gain can have a hugely positive influence on our API
and application design. While this operator is generally useful for expressing
data transformations, it has a particularly nice fit for building large or
high-complexity configuration objects more expressively.

Consider as a primary
candidate the low-level request interface in `elm-http`. Let's say we want to
send a `PATCH` request to a cross-origin endpoint and we need to include a
cookie for authentication and we only want to wait 5 seconds for the request to
complete. Doing so requires two configuration objects:

```elm
import Http

request : Http.Request
request =
  { verb = "PATCH"
  , headers =
    [ ("Origin", "http://lukewestby.com")
    , ("Access-Control-Request-Method", "PATCH")
    , ("Content-Type", "application/json")
    ]
  , url = "https://exampleapi.com/items/4"
  , body = Http.string (encodeItemUpdate itemUpdate)
  }

settings : Http.Settings
settings =
  { Http.defaultSettings
  | timeout = 5 * Time.second
  , withCredentials = True
  }

result : Task Http.RawError Http.Response
result =
  Http.send settings request
```

 This is a pretty obnoxious amount of work to send a request with parameters
that are not too far outside the norm. This is, of course, as it should be as
`elm-http` is intended to be a low-level interface to `XMLHttpRequest` in Elm
and not anything fancy or expressive. But as humans we want something that is
easy to read and understand. This is the intention of
[elm-http-extra](https://github.com/lukewestby/elm-http-extra), and it will
serve as an excellent example of using `(|>)` to build configuration as needed
instead of supplying it all at once.
A roughly equivalent request using `elm-http-extra` looks like the following:

```elm
import Http.Extra as HttpExtra exposing (..)

result : Task (HttpExtra.Error ApiError) (HttpExtra.Response Item)
result =
  HttpExtra.patch "https://exampleapi.com/items/4"
    |> withHeader ("Origin", "http://lukewestby.com")
    |> withHeader ("Access-Control-Request-Method", "PATCH")
    |> withHeader ("Content-Type", "application/json")
    |> withStringBody (encodeItemUpdate itemUpdate)
    |> withTimeout (5 * Time.second)
    |> withCredentials
    |> send decodeItem decodeApiError
```

The advantages of the above are as follows:

- We don't need to know anything about the underlying configuration objects
- We only need to learn or remember the functions which are relevant to the
  the task at hand
- We can easily remove the final `send` and express the whole configuration
  building process as a separate function that is easy to test by doing
  equality comparison on the output.
- We can combine existing operators into more high-level ones that are totally
  reusable.

As an example of the final point, we can extract the headers out into a separate
function that expresses the header needs of every API request and even use the
contents of the request configuration builder record to help us out:

```elm
import Http.Extra as HttpExtra exposing (..)

withApiHeaders : HttpExtra.RequestBuilder -> HttpExtra.RequestBuilder
withApiHeaders builder =
  let
    verb =
      .verb (HttpExtra.toRequest builder)
  in
    builder
      |> withHeader ("Origin", "http://lukewestby.com")
      |> withHeader ("Access-Control-Request-Method", verb)
      |> withHeader ("Content-Type", "application/json")

result : Task (HttpExtra.Error ApiError) (HttpExtra.Response Item)
result =
  HttpExtra.patch "https://exampleapi.com/items/4"
    |> withApiHeaders
    |> withStringBody (encodeItemUpdate itemUpdate)
    |> withTimeout (5 * Time.second)
    |> withCredentials
    |> send decodeItem decodeApiError
```

Using the `(|>)` operator in this way allows us to build totally extensible DSL-
like APIs without all of the complexity of an actual DSL because everything is
just a function with a very specific type of interface. This even allows us to
abstract actual DSLs in a much nicer and easier-to-understand way. Consider
regular expressions, the confusing DSL to end them all. Using this technique
with the
[elm-verbal-expressions](https://github.com/verbalexpressions/elm-verbal-expressions)
package we can express a `Regex` using a human-readable, chainable interface.
Instead of constantly re-learning regular expressions to perform a simple task
like match a correctly formatted url, we can write the following:

```elm
import Regex exposing (Regex)
import VerbalExpressions as Verex exposing (..)

tester : Regex
tester =
  VerbalExpressions.verex
    |> startOfLine
    |> followedBy "http"
    |> possibly "s"
    |> followedBy "://"
    |> possibly "www."
    |> anythingBut " "
    |> endOfLine
    |> toRegex
```

Every call in the pipeline above is just a function which operates on some
configuration builder which we can ultimately transform into a result which
would have been harder to obtain on its own. In the case of `elm-http` it was a
`Task a b` for the request result which took a lot of configuration, and in
this case it is a `Regex` which would have required a hard-to-understand
regular expression string.

Let's try and generalize a few things about this pattern. The process of using a
chainable API with `(|>)` involves three stages, _initialize_, _build_, and
_compile_. During the initialize stage we start with generate a base object of a
particular type, called the _builder_. It could be a totally new record type or
a union type that wraps existing configuration types. In the case of
`elm-http-extra` it is a union type which holds an `Http.Request` and an
`Http.Settings` together. The value obtained during the initialize step should
always be valid for compilation straight away, which means several
initialization functions might be required.

This is the case with `elm-http-extra` since every request must have at least a
verb and a url. The case is much simpler with `elm-verbal-expressions`, since we
can start every `VerbalExpression` from a single base value. In this case we
just expose the one initial value and take advantage of immutability in Elm to
allow chaining from the basic case.

During the build step we make use of the `(|>)` operator by enforcing a specific
signature for the functions in our API. Every function must accept a builder as
its final argument and return a builder as its return value. Any additional
arguments should precede the builder. This data-last signature is a common style
in functional programming as it also makes function composition easy.

The compile step allows to actually make use of the builder. Compilation
functions can have any number of purposes, like extraction of data for testing
or actually running a `Task`. For example, in `elm-http-extra` there are three
compilation functions - `toRequest`, `toSettings`, and `send`. The first two
allow us to extract and inspect information from the builder whereas `send`
actually creates a `Task`. Compilation functions that do not create a `Task` can
be used during the build step as well since they are pure.

### Summary

As the amount of configuration or mental overhead to perform a particular
operation increases it can be useful to dissociate the various components of
that configuration into a series of smaller, step-wise operations. Such an
effort is best expressed through functions that play to the strengths of the
`(|>)` operator. As examples, `elm-http` requires much configuration for many
common request cases, and regular expressions require a lot of research and
preparation to use correctly. Through the use of chainable APIs and then `(|>)`,
`elm-http-extra` and `elm-verbal-expressions` make those processes easy to
read and maintainable. The success of this API design in these cases can be
easily applied to many similar use cases by simply following the initialize-
build-compile pattern.
