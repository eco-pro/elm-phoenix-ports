# elm-phoenix-ports

An Elm client for [Phoenix](http://www.phoenixframework.org) Channels using ports.

This can be used to connect to Phoenix Channels using a very similar API to that of https://github.com/saschatimme/elm-phoenix.


It is compatible with Elm 0.19 as it does not use an Effects Manager and uses ports to communicate with the [phoenix.js](https://hexdocs.pm/phoenix/js/index.html) library.

## Getting started

### Migrating from saschatimme/elm-phoenix

In addition to the installation steps below please see [Migration Guide](migration-guide.md)

### Installation
This is not yet shipped as an Elm package as it requires installing js code that is shipped with the library.  Therefore
you need to install this with `npm` or similar direct from github.

In `package.json` add the dependency from github:
```
"dependencies": {
  ...
  "elm-phoenix-ports": "github:chazsconi/elm-phoenix-ports",
  ...
},

```
In `elm.json` add the source directories for both the library and the ports module.  The ports module is in a separate folder
so that the main code can subsequently be made into an Elm package (ports are not allowed in packages).
```
"source-directories": [
    "src",
    "node_modules/elm-phoenix-ports/src",
    "node_modules/elm-phoenix-ports/elm-ports"
],
```

### Connect your JS Elm launcher
You need to initialise the Elm Phoenix Ports JS code by passing your elm app object in your launcher.  For example

```javascript
import * as elmPhoenix from "elm-phoenix-ports"

var app = elmXingbox.Elm.MyElmApp.init({flags:{}});
elmPhoenix.init(app);
```

### Wiring up Elm

As there is no effects manager in Elm 0.19, the library has its own messages and model that you need to delegate in the `update` function.

Declare a new message with your existing messages and store `Phoenix`'s model in your own model

```elm
import Phoenix

type Msg
  = NewMsg Value
  | PhoenixMsg (Phoenix.Msg Msg)
  ...


type alias Model =
  { myModelField : String
  , phoenixModel : Phoenix.Model Msg ()
  }
```

Declare a `phoenixConfig` with your `PhoenixMsg`, the `socket` you want to connect to and the channels you want to join, along with what to do when
messages are received. The library code will open the socket connection and join the channels.

```elm
import Phoenix
import Phoenix.Socket as Socket
import Phoenix.Channel as Channel
import Phoenix.Config
import PhoenixPorts

phoenixConfig =
  Phoenix.Config.new PhoenixMsg PhoenixPorts.ports

-- N.B. Do not suffix the endpoint url with `/websocket` as this is automatically added by `phoenix.js`.
endpoint =
    "ws://localhost:4000/socket"

socket =
    Socket.init endpoint

-- channelModel here can be used to dynamically connect to channels.  See advanced usage.
channels channelModel =
  [ Channel.init "room:lobby"
        -- register an handler for messages with a "new_msg" event
        |> Channel.on "new_msg" NewMsg
  ]
```

Initialise the `phoenixModel`, delegate `PhoenixMsg` to `Phoenix` in your `update`, and set the subscriptions:

```elm
import Phoenix
import PhoenixPorts

init : Flags -> ( Model, Cmd Msg )
init flags =
    ( { myModelField = "Hello"
      , phoenixModel = Phoenix.new
      }, Cmd.none )

update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
    case msg of
        ...

        PhoenixMsg phoenixMsg ->
            let    
                ( phoenixModel, cmd ) =
                    Phoenix.update phoenixConfig socket channels () phoenixMsg model.phoenixModel
            in
            ( { model | phoenixModel = phoenixModel }, cmd )
           ...

        NewMsg value ->
          ...

subscriptions model =
    Phoenix.connect phoenixConfig
```

In order to push a message to a channel you can do this:
```elm
import Phoenix
import Phoenix.Push as Push
import Json.Encode as JE

pushMessage =
  let push =
    Push.init "room:lobby" "set_status"
          |> Push.withPayload (JE.string "away")
  in
  Phoenix.push phoenixConfig push

```

## Advanced usage
If you need to join or leave channels dynamically (e.g. in the example of a chat app the user can choose the rooms to join),
you need to define a `ChannelModel` which defines the parameters needed to build the list of channels.  Your `phoenixModel` must
be parameterised with this instead of `()`
e.g.

```elm
type alias Room = String

type alias ChannelModel =
  { rooms: List Room }

type alias Model =
  { myModelField : String
  , rooms : List Room
  , phoenixModel : Phoenix.Model Msg ChannelModel
  }
```

In the `update` create this channel model from your model and pass it
to `Phoenix.update`:
```elm
update msg model =
    case msg of
        ...
        PhoenixMsg phoenixMsg ->
            let
                channelModel = { rooms = model.rooms }
                ( phoenixModel, cmd ) =
                    Phoenix.update phoenixConfig socket channels channelModel phoenixMsg model.phoenixModel
            in
            ( { model | phoenixModel = phoenixModel }, cmd )
            ...
```
This can be then used to dynamically join channels.
e.g.
```elm
channels channelModel =
  [ Channel.init "room:lobby"
        |> Channel.on "new_msg" NewMsg
  ] ++ List.map Channel.init channelModel.rooms
```

## Documentation
Please see the old API [saschatimme/elm-phoenix](https://saschatimme.github.io/elm-phoenix) for the current API functions on `Channel` and `Push`

## Example app
Please see the example chat room app in the `example` folder.

## Limitations
* Not all features from the `saschatimme/elm-phoenix` are implemented.
* Only one socket can be connected to at a time.  If you need more than one you should be able
  to have multiple instances of the `phoenixConfig` and `phoenixModel` although this has not been tested.

## Design decisions

### Replacing Effects Manager

Because there is no Effects Manager available in Elm 0.19, a time subscription is used to check every few 100ms if the model that
is used to construct the channels has changed.  If it has, then the `channels` function is evaluated.
Any new channels are joined, and any old ones are left by making calls to ports.  The JS objects that represent the channels
are stored in the model along with their state. When `phoenix.js` receives an event this is passed back to Elm via a port, which
triggers events in the main app.

In this way, a similar API to `saschatimme/elm-phoenix` is retained, without consumers of the library having
to explicitly join and leave channels.

## Why have a Phoenix.Config

This encapsulates the parent `Msg` type and also the real ports module and is a convenience for passing as a
parameter to `Phoenix.push`.

An alternative would have been to put this information in the `Socket` however
often you will want to have your socket constructed dynamically from your model - e.g. adding a user token as
a connection parameter.  However the config should not need to be parameterised and thus is easier to use
without having to pass around the socket or model.


## Contributing
Contributions are welcome!

## Thanks
Thanks a lot to [Sascha Timme](https://github.com/saschatimme) for the original version of this library
which I used happily and extensively with Elm 0.18 for several years and has a great API design which I wanted to keep.

## Feedback
If you use the package in your project, it would be nice to know


## TODO
* Implement missing features (See TODO markers)
* Publish as a package
