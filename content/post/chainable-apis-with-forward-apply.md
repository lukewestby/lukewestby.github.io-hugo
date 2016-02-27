+++
date = "2016-02-27T11:36:18-06:00"
title = "Chainable APIs with the forward apply operator"
linktitle = "Chainable APIs with the forward apply operator"
weight = 1
description = "Lorem ipsum dolor sit amet, consectetur adipisicing elit. Earum similique, ipsum officia amet blanditiis provident ratione nihil ipsam dolorem repellat."
tags = [
  "go",
  "golang",
  "hugo",
  "development"
]
+++

The `(|>)` operator is, in my opinion, one of the most elegant features of the
Elm language. It lets us read function applications in the order the actually
happen and this readability gain can have a hugely positive influence on our API
and application design. While this operator is generally useful for expressing
data transformations, it has a particularly nice fit for building large or
high-complexity configuration objects more expressively. Consider as a primary
candidate the low-level request interface in `elm-http`. Let's say we want to
send a `PATCH` request to a cross-origin endpoint and we need to include a
cookie for authentication and we only want to wait 5 seconds for the request to
complete. Doing so requires two configuration objects:

```elm
item : Item
item =
  { {- ... -} }


request : Http.Request
request =
  { verb = "PATCH"
  , headers =
    [ ("Origin", "http://lukewestby.com")
    , ("Access-Control-Requesst-Method", "PATCH")
    , ("Content-Type", "application/json")
    ]
  , url = "https://exampleapi.com/items/4"
  , body = Http.string (encodeItem item)
  }


settings : Http.Settings
settings =
  { Http.defaultSettings
  | timeout = 5 * Time.second
  , withCredentials = True
  }
```
