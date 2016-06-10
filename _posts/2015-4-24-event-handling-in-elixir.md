---
layout: post
title: Event Handling in elixir
date: 2015-4-24
categories:
  - programming
tags:
  - GenEvent
  - elixir
  - event handling
---

I was recently working on a project in elixir where I needed a way let other OTP applications subscribe to data updates from the application I was building. After asking around some I found GenEvent. I've used event handlers and event driven architecture with other languages, but I had not ventured into event handling with elixir yet.

The [getting started guide on elixir-lang.org](http://elixir-lang.org/getting-started/mix-otp/genevent.html), has a basic intro to using GenEvent. You start a manager, attach a handler and then send events to the manager using the `GenEvent.notify/2` or `GenEvent.sync_notify/2` functions.

### GenEvent

First we need to define our event handler, for this example we will define a simple handler that logs every event as an info message.

```elixir
defmodule MyEventHandler do
  use GenEvent
  require Logger

  def handle_event(event, parent) do
    Logger.info("event received: #{inspect event}")

    {:ok, parent}
  end
end
```

To use our event handler we first need to start an event manager process. In elixir this can be done by starting a GenEvent process with `start_link`

```elixir
iex(1)> {:ok, manager} = GenEvent.start_link
{:ok, #PID<0.55.0>}
```

Next we add our event handler `MyEventHandler` to the event manager we just started.

```elixir
iex(2)> GenEvent.add_handler(manager, MyEventHandler, self())
:ok
```

Finally we send an event to the manager for our event handler to process.

```elixir
iex(3)> GenEvent.notify(manager, {:my_event, "testing"})
:ok
iex(4)>
00:00:00.000 [info] event received: {:my_event, "testing"}
```

That gets us through the basics of generating and handling our own custom events. Its a good intro, but after implementing the basics I wanted more information.

Looking at the elixir docs I saw the `add_mon_handler/3` function. Since I'm writing this in elixir, it should be fault tolerant. And for my application this was a must.

## Fault tolerance

Adding the event handler using the `add_mon_handler/3` function will link it to the calling process. If an error occurs when the event handler is processing an event, a message will be sent from the event manager to the process that call `add_mon_handler/3`.  

I was curious how other people are using `GenEvent` with a monitored event handler. So I started searching github for `GenEvent.add_mon_handler`. I came across two common ways `add_mon_handler/3` is being used.

**Bare bones GenServer**
```elixir
defmodule MyEventHandlerWatcher do

  def start_link(manager) do
    GenServer.start_link(__MODULE__, manager, [])
  end

  def init(manager) do
    :ok = GenEvent.add_mon_handler(manager, MyEventHandler, self())

    {:ok, manager}
  end
end
```

In the event that `MyEventHandler` crashes, then `MyEventHandlerWatcher` will also crash. As long as `MyEventHandlerWatcher` is being supervised, then it will be restarted and re-add `MyEventHandler` to the event manager. This will give a basic level of fault tolerance and ensures our event handler is restarted in the event of an error.

This approach is terse, but to me obfuscates what is happening. When `MyEventHandler` crashes the event manager sends a message `{:gen_event_EXIT, handler, reason}` to `MyEventHandlerWatcher`. Since it does not handle this message, `MyEventHandlerWatcher` will also crash and then be restarted by its supervisor. A side effect of this is and large unnecessary error message for the `MyEventHandlerWatcher` crash.

Also this method only works if `MyEventHandlerWatcher` does not `use GenServer`. If it does then the `:gen_event_EXIT` message will be swallowed and `MyEventHandler` will not be re-added. This completely defeats the purpose of using `add_mon_handler` over `add_handler`.

**Handling the event handler crash explicitly**

Another approach, that I prefer, is to explicitly handle the
`:gen_event_EXIT` message and re-add `MyEventHanlder` to the event manager.

```elixir
defmodule MyEventHandlerWatcher do

  def start_link(manager) do
    GenServer.start_link(__MODULE__, manager, [])
  end

  def init(manager) do
    start_handler(manager)

    {:ok, manager}
  end

  def start_handler(manager) do
    :ok = GenEvent.add_mon_handler(manager, MyEventHandler, self())
  end

  def handle_info({:gen_event_EXIT, _handler, _reason}, manager) do
    start_handler(manager)

    {:ok, manager}
  end
end
```

If you would like to test these approaches yourself, I have each implemented together in my [gen event patterns][0] repo. Follow the directions in the `GenEventPatterns` moduledoc to work through the scenarios.

For more practical examples of `GenEvent` and `add_mon_handler` usage you can look at:

- [Elixir's Logger.Watcher][1]

- [Hedwig.Client][2]

- [My Magnectic Compass application HMC5883L][3]

Also I wrote a simple GenServer module for adding monitored event handlers [mon_handler](https://hex.pm/packages/mon_handler) [code](https://github.com/tattdcodemonkey/mon_handler)

[0]: https://github.com/TattdCodeMonkey/gen_event_patterns
[1]: https://github.com/elixir-lang/elixir/blob/master/lib/logger/lib/logger/watcher.ex
[2]: https://github.com/scrogson/hedwig/blob/master/lib/hedwig/client.ex#L160-L164
[3]: https://github.com/TattdCodeMonkey/hmc5883l/blob/master/lib/hmc5883l/event_handler.ex
