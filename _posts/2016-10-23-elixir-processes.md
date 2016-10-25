---
layout: post
title: "State your Process: Storing State Through Processes in Elixir"
comments: false
description: "Storing state in elixir with processes."
keywords: "elixir state functional processes"
---

As everyone who talks to me about is probably sick of hearing by now, I've been
dabbling a lot in [Elixir](http://elixir-lang.org/) lately. I've been enjoying it, but
I think there's a pretty sizeable learning curve, especially coming from an object
oriented language such as Ruby. One thing that tripped me up is the concept of state.

Elixir is a [functional](https://en.wikipedia.org/wiki/Functional_programming) language.
A key idea of functional programming is that your data is *immutable*, meaning
that your data cannot change. This is a stark contrast compared to Ruby, where
a big part of the language is building and instantiating your own classes, of which
you change and manipulate.

To exemplify this point, let's think about how we would do the same task in a
functional and object oriented approach. We'll use the classic FizzBuzz. If you've
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

While not the most eloquently written, it allows us to keep track of and update
of our state via `counter` and `results`.

Okay, so how do we achieve the same thing in Elixir? That is how, do we keep track
of a counter and results after the logical iteration?

**Processes**, that's how!

%{ highlight ruby %}
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
        {:get, from} ->
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
%{ endhighlight %}
