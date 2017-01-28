---
layout: post
title: Pub/sub with gproc vs GenEvent in elixir
date: 2016-2-17
categories:
  - programming
tags:
  - gproc
  - GenEvent
  - pub/sub
  - elixir
  - functional programming
---

The library [gproc](https://github.com/uwiger/gproc) was recently recommended to me for registering processes in my elixir app. While I was researching the library I came across [this in the readme](https://github.com/uwiger/gproc#use-case-pubsub-patterns):

> An interesting application of gproc is building publish/subscribe patterns.
>
> Example:

```erlang
subscribe(EventType) ->
     %% Gproc notation: {p, l, Name} means {(p)roperty, (l)ocal, Name}
     gproc:reg({p, l, {?MODULE, EventType}}).

 notify(EventType, Msg) ->
     Key = {?MODULE, EventType},
     gproc:send({p, l, Key}, {self(), Key, Msg}).>
```

While doing some further research I came across a [blog post with an elixir example](http://bbhoss.io/easy-pub-sub-event-dispatch-with-gproc-and-elixir/). These peaked my interest and got me thinking about using `gproc` for pub/sub in a project, instead of `GenEvent`.

Below is an example of a bare-bones implementation using `GenEvent` similar to how I've been using it in projects. Then a comparable example using `gproc`.

### Example `GenEvent` usage

```elixir
# supervisor.ex
defmodule EventSup do
  use Supervisor

  def start_link, do: Supervisor.start_link(__MODULE__, [], [])

  def init(_) do
    supervise(
      [
        worker(GenEvent, [[name: :event_manager]], [id: :event_manager]),
        worker(EventMonitor, [:event_manager])
      ],
      [strategy: :one_for_one]
    )
  end
end
# event_monitor.ex
defmodule EventMonitor do
  use GenServer

  def start_link(mgr), do: GenServer.start(__MODULE__, [mgr], [])

  def init(mgr) do
    :ok = add_handler(mgr)
    {:ok, mgr}
  end

  def handle_info({:gen_event_EXIT, _handler, _reason}, mgr) do
    :ok = add_handler(mgr)
    {:noreply, mgr}
  end

  def add_handler(mgr) do
    GenEvent.add_mon_handler(mgr, EventHandler, [])
  end
end

# event_handler.ex
defmodule EventHandler do
  use GenEvent
  require Logger

  def init(_), do: {:ok, {}}

  def handle_event(event, state) do
    Logger.info "received event #{inspect event}"

    {:ok, state}
  end
end

# send event
iex>GenEvent.notify(:event_manager, {:event, "stuff"})
```

### Example `gproc` pub/sub equivalent

```elixir
# supervisor.ex
defmodule EventSup do
  use Supervisor

  def start_link, do: Supervisor.start_link(__MODULE__, [], [])

  def init(_) do
    supervise(
      [
        worker(EventHandler, [:event_manager])
      ],
      [strategy: :one_for_one]
    )
  end
end

# event_handler.ex
defmodule EventHandler do
  use GenServer
  require Logger

  def start_link(topic), do: GenServer.start_link(__MODULE__, topic, [])

  def init(topic) do
    :gproc.reg({:p, :l, topic})

    {:ok, topic}
  end

  def handle_info(msg, topic) do
    Logger.info "received message #{inspect msg}"

    {:noreply, topic}
  end
end

# send message
iex>:gproc.send({:p, :l, :event_manager}, {:message, "stuff"})
```

Between these two I'm favoring `gproc`. Only needing to add one `GenServer` to my supervision tree to receive events from another module or OTP application is really nice. The ability to register more than just atoms as property names is good too. I'm not crazy about the api, but I could easily wrap that if I'm going to be using throughout a large project.

I'm not totally sold on `gproc` just yet though. I still have some other pub/sub solutions to research. I found [elixir_pubsub](https://github.com/simonewebdesign/elixir_pubsub) and [erlbus](https://github.com/cabol/erlbus). Which could both fit the same use case.

There is also the [Phoenix.PubSub](https://github.com/phoenixframework/phoenix_pubsub) now that its being broken out into a separate module. This is the option I'm most interested in as it matures with the next version of Phoenix.
