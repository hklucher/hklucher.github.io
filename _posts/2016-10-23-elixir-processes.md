---
layout: post
title: "State your Process: Storing State Through Processes in Elixir"
comments: false
description: "Storing state in elixir with processes."
keywords: "elixir state functional processes"
---

I've been dabbling a lot in [Elixir](http://elixir-lang.org/) lately. I've been
enjoying it, but I think there's a pretty sizable learning curve, especially
coming from an object oriented language such as Ruby. One thing that tripped me up
was the concept of state.

Elixir is a [functional](https://en.wikipedia.org/wiki/Functional_programming) language.
A key idea of functional programming is that your data is *immutable*, meaning
that your data cannot change state. This is in stark contrast to Ruby, where
a big part of the language is building and instantiating your own classes, in which
you change and manipulate internal state.

To exemplify this point, let's think about how we would achieve a task in a functional
language vs an object oriented language. We'll use the classic FizzBuzz. If you've
never heard of it, you can check out the rules [here](https://en.wikipedia.org/wiki/Fizz_buzz).

First, in Ruby:

{% highlight ruby %}
  class FizzBuzz
    attr_accessor :counter, :results

    def initialize
      @counter = 0
      @results = []
    end

    def upto(n)
      counter.upto(n) do |i|
         if i % 3 == 0 && i % 5 == 0
           results << "FizzBuzz"
         elsif i % 3 == 0
           results << "Fizz"
         elsif i % 5 == 0
           results << "Buzz"
         else
           results << i
         end
      end
      self.counter = n
    end     
  end
{% endhighlight %}

While not super eloquent, it allows us to keep track of and update our state
via `counter` and `results`.

Okay, so how do we achieve the same thing in Elixir? That is how, do we keep track
of `counter` and `results` after the logical iteration?

**Processes**, that's how!

{% highlight ruby %}
  defmodule FizzBuzz do
    def new do
      spawn fn -> state(0, []) end
    end

    def set_counter(pid, value) do
      send(pid, {:set, value, self()})
      receive do x -> x end
    end

    def increment(pid) do
      send(pid, {:increment, self()})
      receive do x -> x end
    end

    def upto(pid, n) do
      current_count = get_counter(pid)
      results = Enum.map(current_count..n, fn(i) -> fizz_buzz_for(i) end)
      set_counter(self, n)

      send(pid, {:set_counter, n, self()})
      receive do x -> x end

      send(pid, {:upto, results, self()})
      receive do x -> x end
    end

    def get_counter(pid) do
      send(pid, {:get, self()})
      receive do x -> x end
    end

    def get_results(pid) do
      send(pid, {:get_results, self()})
      receive do x -> x end
    end

    defp fizz_buzz_for(i) when rem(i, 3) == 0 and rem(i, 5) == 0, do: "FizzBuzz"
    defp fizz_buzz_for(i) when rem(i, 3) == 0, do: "Fizz"
    defp fizz_buzz_for(i) when rem(i, 5) == 0, do: "Buzz"
    defp fizz_buzz_for(i), do: i

    defp state(n, list) do
      receive do
        {:increment, from} ->
          send(from, n + 1)
          state(n + 1, list)
        {:get_counter, from} ->
          send(from, n)
          state(n, list)
        {:set_counter, value, from} ->
          send(from, :ok)
          state(value, list)
        {:get_results, from} ->
          send(from, {:ok, list})
          state(n, list)
        {:upto, results, from} ->
          new_results = list ++ results
          send(from, {:ok, new_results})
          state(n, list ++ results)
      end
    end
  end
{% endhighlight %}

A smidge more complicated, right? Let's walk through it. First, let's demonstrate how the API
would function.

{% highlight ruby %}
iex(1)> f = FizzBuzz.new
#PID<0.87.0>
iex(2)> FizzBuzz.get_counter(f)
0
iex(3)> FizzBuzz.increment(f)
1
iex(4)> FizzBuzz.upto(f, 15)
{:ok,
 [1, 2, "Fizz", 4, "Buzz", "Fizz", 7, 8, "Fizz", "Buzz", 11, "Fizz", 13, 14,
  "FizzBuzz"]}
iex(5)> FizzBuzz.get_results(f)
{:ok,
 [1, 2, "Fizz", 4, "Buzz", "Fizz", 7, 8, "Fizz", "Buzz", 11, "Fizz", 13, 14,
  "FizzBuzz"]}
{% endhighlight %}

We call `FizzBuzz.new` and match `f` to it. This returns `#PID<0.87.0>`, which is a
process ID. This is because we're using the spawn function in new to *spawn* a
new process. This process runs the function `state`, which, for all intents and
purposes, is an ininite recursive loop.

Within the `state` function, we start a receive block, which will receive
messages sent to that PID. Because our logic
depends upon *sending* messages to that process, we always pass in our PID (f)
to each function call. Once told to send, the receive block pattern matches to
whichever function was called, and updates our processes state according to the
matched function.

Notice that when we alter our state (for example, the `upto` function), we update
our data by sending a message to `state` with whatever number was passed in as
the updated counter, and by sending the `results` variable to the spawned function.
Once we pattern match to the sent list, we concatenate those results, and call
`state` recursively with our updated data.

### Opinions
Obviously the Elixir example is a lot longer and more complicated than the Ruby example.
My opinion is that isn't necessarily a bad thing - state, while making things conceptually
easier to understand, can become a hindrance. After all, I can't count all the times
I've had to track down a rogue method call in a Ruby code base that was altering the
structure of a class in an unanticipated way.

As I said earlier, Elixir is a *functional* language, one that isn't necessarily made
to store state. At it's core, functional programming is designed to churn date through
function after function, eventually churning out your desired structure at the end.
However, as I've demonstrated here, that's not the only way to accomplish things!
