+++
date = "2016-02-28T19:33:20-06:00"
title = "Release: elm-http-extra 5.0.0"
linktitle = "Release: elm-http-extra 5.0.0"
weight = 1
description = "elm-http-extra 5.0.0 is out, now with even more flexibility for handling response bodies!"
tags = [
  "elm",
  "elm-http-extra",
  "release",
  "open-source"
]
+++

- http://package.elm-lang.org/packages/lukewestby/elm-http-extra/5.0.0
- https://github.com/lukewestby/elm-http-extra


<iframe src="http://ghbtns.com/github-btn.html?user=lukewestby&repo=elm-http-extra&type=star&count=true" frameborder="0" scrolling="0" height="20px"></iframe><br/><iframe src="http://ghbtns.com/github-btn.html?user=lukewestby&repo=elm-http-extra&type=fork&count=true" frameborder="0" scrolling="0" height="20px"></iframe>

I've just released elm-http-extra 5.0.0, and along with the links I wanted to
share the reasons for the changes and discuss some of the challenges that I
faced addressing them.

> A huge thank you to [Fred Yankowski](https://twitter.com/fredcy) for using
  elm-http-extra, finding a serious flaw in the design, talking through
  it with me on the [Elm Slack](http://elmlang.herokuapp.com), and
  generally being an awesome member of the Elm community!

### The problem

Prior to this release, `Http.Extra.send` accepted three arguments: a
`Json.Decode.Decoder` for the value expected on success, another
`Json.Decode.Decoder` for the value expected on server error, and a
`RequestBuilder` to pipe with `(|>)` and kick off the request. (For more
information on this API design, see [Chainable APIs with the forward apply operator](/chainable-apis-with-forward-apply)). This is a significant
improvement over elm-http because it lets you deal with the body of a response
in the case of a server error. You might be dealing with an API that returns an
object like the following on error:

```json
{
  "message": "There was an error",
  "status": 500,
  "success": false
}
```

With elm-http the process for accessing this data as a record is fairly
laborious, and with elm-http-extra it is easy and also _required_. I thought
this was a pretty clever design until Fred told me about how his API worked. On
a 404 error, his API returned the following body:

```
"404 page not found"
```

The `Task` failed as expected, but instead of failing with a
`BadResponse String` and providing access to the message, it always failed with
`UnexpectedPayload` and an error message `"Unexpected token p”`, even though
the error body was being decoded with `Json.Decode.string`. The problem is that
`"404 page not found"` is not valid JSON. `"\"404 page not found\""` with the
included quotes _is_ valid JSON. The type of value being accepted from the body
needed to be lifted one level higher.

### The solution

I spent some time thinking about this and decided that the signature for `send`
should remain roughly the same, but include some polymorphism for how bodies are
handled. The first attempt I made was to create a tagged union which could
express either a plain text string body or a JSON body and contain a decoder:

```elm
type BodyReader a
  = StringReader
  | JsonReader (Json.Decode.Decoder a)
```

This might raise an immediate red flag in the experienced Elm programmer's mind,
as this encoding of the transform as a union type implicitly expresses the
output type of a function on that type. Ultimately I couldn't get a rework using
this method to compile without writing one function for each combination of
`BodyReader` instances for error and success cases.

So I scrapped that plan and went one step outward: generalizing the idea of what
a `BodyReader` does at the function level. In this case, `BodyReader` is just a
function interface:

```elm
type alias BodyReader a =
  Http.Value -> Result String a
```

This interface bears a lot of resemblance to `Json.Decode.decodeString`, in
which a string value goes in and a `Result String a` comes out. Each
`BodyReader` is allows to take on any kind of `Http.Value` and optionally fail.
This has a two-fold benefit.

1. Even though `Text` is the only supported `Http.Value` type right now, this
generic function interface leaves room for extensibility once more values are
introduced.
2. Since we only need to write a function, readers can get even more specific
than just accessing the string value contained by `Text`. `jsonReader` does
this, and anyone can do the same by `Result.map`-ing the output of
`stringReader`.

So there it is! Now, instead of having to pass a `Json.Decode.Decoder` and
always getting a parsing error, use `stringReader` to get a string, or continue
decoding JSON with `jsonReader`.

### Other changes

[Max Goldstein](https://twitter.com/maxgoldst) pointed out that the signature of
`withHeader` accepting a `(String, String)` added more visual noise than was
necessary. He is absolutely right, and now `withHeader` accepts two string, one
for the key and one for the value. See the example in the next section for a
demo!

### New README example

In this example, we expect a successful response to be JSON array of strings,
like:

```json
["hello", "world", "this", "is", "the", "best", "json", "ever"]
```

and an error response might have a body which just includes text, such as the
following for a 404 error:

```json
Not Found.
```

We'll use `HttpExtra.jsonReader` and a `Json.Decode.Decoder` to parse the
successful response body and `HttpExtra.stringReader` to accept a string
body on error without trying to parse JSON.

```elm
import Time
import Http.Extra as HttpExtra exposing (..)
import Json.Decode as Json


itemsDecoder : Json.Decoder (List String)
itemsDecoder =
  Json.list Json.string


addItem : String -> Task (HttpExtra.Error String) (HttpExtra.Response (List String))
addItem item =
  HttpExtra.post "http://example.com/api/items"
    |> withBody (Http.string "{ \"item\": \"" ++ item ++ "\" }")
    |> withHeader "Content-Type" "application/json"
    |> withTimeout (10 * Time.second)
    |> withCredentials
    |> send (jsonReader itemsDecoder) stringReader
```

### Contributing to elm-http-extra

I'm always happy to receive any feedback and ideas about additional features or
anything at all! Any input and pull requests are very welcome and encouraged. If
you'd like to help or have ideas, get in touch with me at [@luke_dot_js on
Twitter](https://twitter.com/luke_dot_js) or @luke in the [Elm Slack](http://elmlang.herokuapp.com)!
