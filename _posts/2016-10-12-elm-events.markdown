---
layout: post
title:  "Html events in elm"
date:   2016-10-12 18:52:20 +0300
categories: elm
---

Elm supports html events with [elm-lang/html](http://package.elm-lang.org/packages/elm-lang/html/latest) package.

Using it we can create a simple application, which will just increase click counter with each click.

I assume you're familiar with typical elm application structure, but if not, you can read about it [here](https://guide.elm-lang.org/architecture/).

{% highlight elm %}

import Html exposing (Html, div, text, label)
import Html.Attributes exposing (style)
import Html.Events exposing (onClick)
import Html.App as App

main : Program Never
main =
  App.program
    { init = init
    , view = view
    , update = update
    , subscriptions = \_ -> Sub.none
    }

init : ( Model, Cmd Msg )
init =
  { labelClicked = 0
  , divClicked = 0
  } ! []

type alias Model =
  { labelClicked : Int
  , divClicked : Int
  }

view : Model -> Html Msg
view model =
  div
    []
    [ div
      [ style [("width", "200px"), ("height", "50px"), ("background-color", "red")]
      , onClick ClickDiv
      ]
      [ label
        [ onClick ClickLabel ]
        [ text <| "Label text" ]
      ]
    , div [] [ text <| "Label clicked: " ++ toString(model.labelClicked) ]
    , div [] [ text <| "Div clicked:" ++ toString(model.divClicked) ]
    ]

type Msg
  = ClickLabel
  | ClickDiv

update msg model =
  case msg of
    ClickLabel ->
      { model | labelClicked = model.labelClicked + 1 } ! []

    ClickDiv ->
      { model | divClicked = model.divClicked + 1 } ! []

{% endhighlight %}

After several clicks it looks like this:

![this]({{ site.url }}/assets/1.png)

So, basically with each `label` or `div` click we send an appropriate message (`ClickLabel` or `ClickDiv`),
increment counter in the model and let elm update the view.

The same way works the rest of mouse events.

## Checking click target

But the problem with this solution is that clicking on a `label` will also increase `div` click counter (you would get the same issue, using pure javascript).
To avoid such behavior we would check the `target` in the `event` object in the javascript. To achieve the same in elm we can use the [custom event handler](http://package.elm-lang.org/packages/elm-lang/html/latest/Html-Events#on):

{% highlight elm %}

onClickDiv : (String -> Msg) -> Attribute Msg
onClickDiv message =
  on "click" (Json.map message <| Json.at ["target", "tagName"] Json.string)

{% endhighlight %}

We should add a string parameter to `ClickDiv` message since it will receive a `tagName` now:

{% highlight elm %}

type Msg
  = ClickLabel
  | ClickDiv String

{% endhighlight %}

and the last change is in the `update` function:

{% highlight elm %}

ClickDiv tagName ->
  if tagName == "DIV" then
    { model | divClicked = model.divClicked + 1 } ! []
  else
    model ! []

{% endhighlight %}

Now label and div click events are handled separately:

![]({{ site.url }}/assets/2.png)


## Handling click with "shift" button pressed

Now, what if we need to add a `shift` click event? In javascript, one possible solution is to check `event.shiftKey` value. We can do the same in elm.

First, we need to define an `Event` and `EventTarget` record:

{% highlight elm %}

type alias ClickEvent =
  { shiftKey : Bool
  , target : EventTarget
  }

type alias EventTarget =
  { tagName : String
  }

{% endhighlight %}

Next, we should modify a decoder, used in our custom event handler:

{% highlight elm %}

onClickRow : (Event -> msg) -> Attribute msg
onClickRow message =
  on "click" (Json.map message clickDecoder)

clickDecoder : Json.Decoder ClickEvent
clickDecoder =
  Json.object2 ClickEvent
    ("shiftKey" := Json.bool)
    ("target" := targetDecoder)

targetDecoder : Json.Decoder EventTarget
targetDecoder =
  Json.object1 EventTarget
    ("tagName" := Json.string)

{% endhighlight %}

Again, we need to change a message to receive our `ClickEvent`:

{% highlight elm %}

type Msg
  = ClickLabel
  | ClickDiv ClickEvent

{% endhighlight %}

and `update` function:

{% highlight elm %}

ClickDiv e ->
  if e.target.tagName == "DIV" && e.shiftKey then
    { model | divClicked = model.divClicked + 1 } ! []
  else
    model ! []

{% endhighlight %}

So, this way we will increment div click counter only if we clicked, holding a shift key.
The same way you can decode any other values from the event object.

Below is the full listing:

{% highlight elm %}
import Html exposing (Html, Attribute, div, text, label)
import Html.Attributes exposing (style)
import Html.Events exposing (on, onClick)
import Html.App as App
import Json.Decode as Json exposing ((:=))

main : Program Never
main =
  App.program
    { init = init
    , view = view
    , update = update
    , subscriptions = \_ -> Sub.none
    }

init : ( Model, Cmd Msg )
init =
  { labelClicked = 0
  , divClicked = 0
  } ! []

type alias Model =
  { labelClicked : Int
  , divClicked : Int
  }

view : Model -> Html Msg
view model =
  div
    []
    [ div
      [ style [("width", "200px"), ("height", "50px"), ("background-color", "red")]
      , onClickDiv ClickDiv
      ]
      [ label
        [ onClick ClickLabel ]
        [ text <| "Label text" ]
      ]
    , div [] [ text <| "Label clicked: " ++ toString(model.labelClicked) ]
    , div [] [ text <| "Div clicked:" ++ toString(model.divClicked) ]
    ]

type Msg
  = ClickLabel
  | ClickDiv ClickEvent

type alias ClickEvent =
  { shiftKey : Bool
  , target : EventTarget
  }

type alias EventTarget =
  { tagName : String
  }

update msg model =
  case msg of
    ClickLabel ->
      { model | labelClicked = model.labelClicked + 1 } ! []

    ClickDiv e ->
      if e.target.tagName == "DIV" && e.shiftKey then
        { model | divClicked = model.divClicked + 1 } ! []
      else
        model ! []

onClickDiv : (ClickEvent -> msg) -> Attribute msg
onClickDiv message =
  on "click" (Json.map message clickDecoder)

clickDecoder : Json.Decoder ClickEvent
clickDecoder =
  Json.object2 ClickEvent
    ("shiftKey" := Json.bool)
    ("target" := targetDecoder)

targetDecoder : Json.Decoder EventTarget
targetDecoder =
  Json.object1 EventTarget
    ("tagName" := Json.string)

{% endhighlight %}

## Sending additional data with click event

Pretty often it is required to send additional data with event object. Since each event should send a message, we will update our click message to receive required data. In this case it will be element `id` (but you can send any data you want of course).
First, lets update our message to receive `id` as a string:

{% highlight elm %}

type Msg
  = ClickLabel
  | ClickDiv String ClickEvent

{% endhighlight %}

Next, lets create several elements with different `id`:

{% highlight elm %}

view : Model -> Html Msg
view model =
  div
    []
    [ div
      [ style [("width", "200px"), ("height", "50px"), ("background-color", "red")]
      , onClickDiv <| ClickDiv ""
      ]
      [ label
        [ onClick ClickLabel ]
        [ text <| "Label text" ]
      ]
    , div [ onClickDiv <| ClickDiv "foo" ] [ text <| "Label clicked: " ++ toString(model.labelClicked) ]
    , div [ onClickDiv <| ClickDiv "bar" ] [ text <| "Div clicked:" ++ toString(model.divClicked) ]
    , div [] [ text <| "Div Id: " ++ model.divId ]
    ]

{% endhighlight %}

and the last part is model update:

{% highlight elm %}

type alias Model =
  { labelClicked : Int
  , divClicked : Int
  , divId : String
  }

{% endhighlight %}

Now with each shift + click on "Label clicked" or "Div clicked" element you will see updated "Div id" element.
Such approach is pretty useful for lists of the same type data, so you can identificate list item which was clicked.
