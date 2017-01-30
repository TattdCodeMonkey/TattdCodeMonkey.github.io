---
layout: post
title: ??
date: 2017-1-XX
categories:
  - programming
tags:
  - SQS
  - AWS
  - GenStage
  - elixir
  - functional programming
---

When I first saw `GenStage` I thought it would be a great fit for processing messages from an AWS SQS queue. This is something I've done a bit of in other languages, but elixir and particularly `GenStage` gives a nice abstraction for this type of problem. For the purpose of this post I'm going to ignore the actual processing of the messages from SQS queue (consumer `GenStage`) and only focus on writing a `GenStage` producer that gets data from a SQS queue.

Before we start we will want to add some dependencies to our project:

```elixir
def deps do
  [
    {:ex_aws, "~> 1.1.0"},
    {:poison, ">= 1.2.0"},
    {:hackney, "~> 1.6"},
    {:sweet_xml, "~> 0.6"},
    {:gen_stage, "~> 0.11.0"},
  ]
end
```

- `ex_aws`: we'll use this to interact with AWS and specifically the SQS queue
- `poison`, `hackney` & `sweet_xml`: These are optional dependencies for `ex_aws` that make our lives easier
- `gen_stage`: the thing we are here for 😃

You will also need to configure `ex_aws` and you should look at the [Getting Started section](https://hexdocs.pm/ex_aws/ExAws.html#module-getting-started) of the `ex_aws` docs.

SQS will default to `us-east-1` region, but this and other setting can be changed in the config.

So now lets get started writing our `GenStage` producer.

```elixir
defmodule SQSProducer do
  use GenStage

  def start_link(sqs_queue_name, opts \\ []) do
    GenStage.start_link(__MODULE__, sqs_queue_name, opts)
  end
```

This is a pretty bare-bones `start_link` definition, we pass the module and the SQS queue name that we will be getting messages from. Then we also take options for the `GenStage` process.

Next we need out `init` function that will setup the state of our process as a map with the SQS queue name.

```elixir
  def init(sqs_queue_name) do
    {:producer, %{queue: queue_name}}
  end
```

Now we can start on our `GenStage` implementation. The simplest and most naive approach would be to directly get messages from SQS every time we receive demand.

```elixir
def handle_demand(incoming_demand, state) do
  aws_resp = ExAws.SQS.receive_message(
    state.queue,
    max_number_of_messages: min(state.demand, incoming_demand)
  )
  |> ExAws.request

  messages = case aws_resp do
    {:ok, resp} ->
      resp.body.messages
    {:error, reason} ->
      []
  end

  {:noreply, messages, state}
end
```

For production use we would not just ignore errors like we are doing here, we would also likely refactor the getting of message to its own function. But what else is wrong here?

- What happens if our SQS queue is empty?
- What happens if SQS call does error?

With the default `GenStage` consumer events are asked for on start up and then after handling received events. If we return no events from our `handle_demand/2` function, then we would never call SQS again to satisfy out demand. This would only work if our SQS queue had more messages than we were asking for and AWS was always available.

So lets try to improve this by separating getting messages from SQS from the `handle_demand/2` function. But first we will need to add our outstanding demand to the process state.

```elixir
  def init(sqs_queue_name) do
    {:producer, %{queue: queue_name, demand: 0}}
  end
```

Now we need to update the state when we receive demand.

```elixir
def handle_demand(incoming_demand, state) do
  new_demand = state.demand + incoming_demand
  {:noreply, messages, %{state| demand: new_demand}}
end
```

But we still need to get messages from SQS, so lets send ourselves a message to do that.

```elixir
def handle_demand(incoming_demand, state) do
  new_demand = state.demand + incoming_demand

  Process.send(self(), :get_messages, [])

  {:noreply, messages, %{state| demand: new_demand}}
end
```

Then we need to handle the message.

```elixir
def handle_info(:get_messages, state) do
  aws_resp = ExAws.SQS.receive_message(
    state.queue,
    max_number_of_messages: min(state.demand, 10)
  )
  |> ExAws.request

  messages = case aws_resp do
    {:ok, resp} ->
      resp.body.messages
    {:error, reason} ->
      []
  end

  new_demand = state.demand - Enum.count(messages)

  {:noreply, messages, %{state| demand: new_demand}}
end
```

This is better, but still doesn't fix our issues when SQS has no messages or errors. Lets add some code to keep getting messages until our demand = 0.

```elixir
def handle_info(:get_messages, state) do
  aws_resp = ExAws.SQS.receive_message(
    state.queue,
    max_number_of_messages: min(state.demand, 10)
  )
  |> ExAws.request

  messages = case aws_resp do
    {:ok, resp} ->
      resp.body.messages
    {:error, reason} ->
      []
  end

  new_demand = state.demand - Enum.count(messages)

  cond do
    new_demand == 0 -> :ok
    true -> Process.send(self(), :get_messages, [])
  end

  {:noreply, messages, %{state| demand: new_demand}}
end
```

Thats good, but if SQS does return 0 messages then we are going to be constantly making requesting until we get messages and our demand is not zero. So lets add some additional logic for that case.

```elixir
def handle_info(:get_messages, state) do
  aws_resp = ExAws.SQS.receive_message(
    state.queue,
    max_number_of_messages: min(state.demand, 10)
  )
  |> ExAws.request

  messages = case aws_resp do
    {:ok, resp} ->
      resp.body.messages
    {:error, reason} ->
      []
  end

  num_messages_received = Enum.count(messages)
  new_demand = state.demand - num_messages_received

  cond do
    new_demand == 0 -> :ok
    num_messages_received == 0 ->
      Process.send_after(self(), :get_messages, 200)
    true ->
      Process.send(self(), :get_messages, [])
  end

  {:noreply, messages, %{state| demand: new_demand}}
end
```

We could improve this more by adding some back-off if we repeatedly get 0 messages back from SQS, but this will do for now.

Now we have a new problem, when we have multiple consumers we will send ourselves multiple `:get_messages` messages. Again flooding SQS, so lets fix that by only sending the message when demand is 0.

```elixir
def handle_demand(incoming_demand, %{demand: 0} = state) do
  new_demand = state.demand + incoming_demand

  Process.send(self(), :get_messages, [])

  {:noreply, messages, %{state| demand: new_demand}}
end
def handle_demand(incoming_demand, %{demand: 0} = state) do
  new_demand = state.demand + incoming_demand

  {:noreply, messages, %{state| demand: new_demand}}
end
```

And thats pretty much it for a basic implementation. There are several things we would want to improve for production. Refactor things into smaller functions, handle errors etc.

```elixir
defmodule SQSProducer do
  use GenStage

  def start_link(queue_name, opts \\ []) do
    GenStage.start_link(__MODULE__, queue_name, opts)
  end

  def init(queue_name) do
    state = %{
      demand: 0,
      queue: queue_name
    }

    {:producer, state}
  end

  def handle_demand(incoming_demand, %{demand: 0} = state) do
    new_demand = state.demand + incoming_demand

    Process.send(self(), :get_messages, [])

    {:noreply, messages, %{state| demand: new_demand}}
  end
  def handle_demand(incoming_demand, %{demand: 0} = state) do
    new_demand = state.demand + incoming_demand

    {:noreply, messages, %{state| demand: new_demand}}
  end

  def handle_info(:get_messages, state) do
    aws_resp = ExAws.SQS.receive_message(
      state.queue,
      max_number_of_messages: min(state.demand, 10)
    )
    |> ExAws.request

    messages = case aws_resp do
      {:ok, resp} ->
        resp.body.messages
      {:error, reason} ->
        # You probably want to handle errors differently than this.
        []
    end

    num_messages_received = Enum.count(messages)
    new_demand = max(state.demand - num_messages_received, 0)

    cond do
      new_demand == 0 -> :ok
      num_messages_received == 0 ->
        Process.send_after(self(), :get_messages, 200)
      true ->
        Process.send(self(), :get_messages, [])
    end

    {:noreply, messages, %{state| demand: new_demand}}
  end
end
```
