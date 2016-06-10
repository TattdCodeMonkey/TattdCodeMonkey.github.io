---
layout: post
title: Fun with Tasks
date: 2015-10-31
categories:
  - programming
tags:
  - tasks
  - elixir
  - functional programming
---

Recently a friend was working on an extreme version of a word counting exercise in elixir. He did the original exercise with [exercism.io](http://exercism.io/) and then expanded it to count all the words in a 65k line text file of War and Peace.

He got it working well but it was taking about 119 seconds to process the file. We talked about it and it seemed like a great use case for elixir Tasks. I hadn't used Tasks much yet, so the next time I had some free time I tried to refactor his solutions to use Tasks and hopefully improve the 119 seconds processing time.

The first thing to do was make sure we could count the words in a single line of text asynchronously. The original code for this was:

```elixir
def count(string) do
  string
  |> String.strip
  |> String.downcase
  |> String.split
  |> reduce
end

defp reduce(word_list) do
  Enum.reduce word_list, %{}, fn (i, acc) ->
    Dict.update acc, i, 1, &(&1 + 1)
  end
end
```

Most of this is good, but instead of updating a map locally I moved the word count to a HashDict stored in an Agent. The new count function looks like this

```elixir
def count_words(line) do
  line
  |> String.strip
  |> String.downcase
  |> String.split
  |> Enum.map(fn word -> WordsAgent.increment_word(word) end)
end
```

We take the line of text, strip it, downcase it then split it. with the resulting array we make calls to the WordsAgent.increment_word function that will add one to the HashDict value for the word passed

That will work well enough. There are short comings in this code that we can address later if we want the most accurate counts. But for now lets look at how we can use Tasks to run this function concurrently.

```elixir
def task_counting(line, {outstanding_tasks, tasks}) do
  task_pid = Task.async(fn -> count_words(line) end)
  {outstanding_tasks + 1, [task_pid | tasks]}
end
```

This is where things get interesting, so this function will create a Task and run the count_words function with the line of text given in the first argument. The second argument is a tuple with the current number of tasks that have been created and a list of the PIDs for those tasks. We use that tuple to add our new task to that list and count.

This function is used by a reduce but there is higher order function to throttle the number of tasks we create at any given time.

```elixir
def task_counting(line, {outstanding_tasks, tasks})
  when outstanding_tasks >= @max_current_line_tasks do

  wait_for_tasks(tasks)

  task_counting(line, {0, []})
end
...
def wait_for_tasks(tasks) do
  tasks
  |> Enum.reverse
  |> Enum.map(&(Task.await(&1)))

  :ok
end
```

Once we reach our max outstanding tasks, stop and wait for all of them to complete before creating new ones. This will not only throttle the current running tasks but the reading of the file as well.

Finally the function that calls the above code is as follows

```elixir
def count(file_path) do
  file_path
  |> File.stream!
  |> Enum.reduce({0,[]},&task_counting/2)
  |> summarize
end
```

We take the path to the text file, create a file stream and reduce over it with the task_counting function to concurrently count the words in each line of text. And at the end we call summarize which returns a string with some basic metrics.

Finally the entry function that calls count with a Task

```elixir
def process_file_async(file_path) do
  WordsAgent.clear_words
  IO.puts "Starting to count words in file"
  Task.async(fn -> count(file_path) end)
  |> Task.await(:infinity)
  |> IO.puts
end
```

This was a first pass and got the processing time down below 6 seconds. I tried different values for @max_current_line_tasks (10, 100, 1000) and for my quad-core MBP 20 yielded the best results.

I'm aware this solution is FAR from perfect. It was a fun exercise and cutting the total processing time down to 5-6 seconds from 119 seconds was pretty cool. I think there are still some improvements to be made here. Some for accuracy and maybe some for faster processing, like creating a task to wait_for_tasks. But overall I'm happy with this as an excuse to have some fun with elixir Tasks.

All the code I wrote for this can be found here: [https://github.com/TattdCodeMonkey/werd_counter/tree/stream](https://github.com/TattdCodeMonkey/werd_counter/tree/stream)

```elixir
defmodule WerdCounter do
  @max_current_line_tasks 20
  @print_top_words 5

  def process_file_async(file_path) do
    WordsAgent.clear_words
    IO.puts "Starting to count words in file"
    Task.async(fn -> count(file_path) end)
    |> Task.await(:infinity)
    |> IO.puts
  end

  def count(file_path) do
    file_path
    |> File.stream!
    |> Enum.reduce({0,[]},&task_counting/2)
    |> summarize
  end

  def task_counting(line, {outstanding_tasks, tasks})
    when outstanding_tasks >= @max_current_line_tasks do

    wait_for_tasks(tasks)

    task_counting(line, {0, []})
  end
  def task_counting(line, {outstanding_tasks, tasks}) do
    task_pid = Task.async(fn -> count_words(line) end)
    {outstanding_tasks + 1, [task_pid | tasks]}
  end

  def wait_for_tasks(tasks) do
    tasks
    |> Enum.reverse
    |> Enum.map(&(Task.await(&1)))

    :ok
  end

  def count_words(line) do
    line
    |> String.strip
    |> String.downcase
    |> String.split
    |> Enum.map(fn word -> WordsAgent.increment_word(word) end)
  end

  def summarize({_, tasks}) do
    wait_for_tasks(tasks)

    WordsAgent.sort_desc
    |> Enum.take(@print_top_words)
    |> Enum.reduce({1, "\nCount complete, Top #{@print_top_words} words by usage:\n"},
        fn
          {word, count}, {index, summary} -> {index + 1, summary <> "#{index}: #{word} - #{count}\n"}
        end
      )
    |> (fn {_, summary} -> summary end).()
  end
end

defmodule WordsAgent do
  defp name, do: __MODULE__

  def start_link, do: Agent.start_link(fn -> HashDict.new end, name: name())

  def get, do: Agent.get(name(), &(&1))

  def increment_word(word) do
    Agent.update(
      name,
      fn val ->
        Dict.update(val, word, 1, &(&1 + 1))
      end
    )
  end

  def clear_words do
    Agent.update(name, fn _ -> HashDict.new end)
  end

  def sort_desc do
    current = get
    sorted = Enum.sort(current, fn {_, v1}, {_, v2} -> v1 > v2 end)
    Agent.update(name, fn _ -> sorted end)
    sorted
  end
end
```
