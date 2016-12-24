---
layout: post
title: "Elixir Streams and Files"
comments: false
description: "An overview of Elixir streams"
keywords: "elixir streams"
---

I've recently undertaken [advent of code](https://www.adventofcode.com). If
you're unfamiliar, it's an annual event every December that posts a new code
challenge for 25 days leading up to Christmas. Usually these challenges have
a large text file as an input, with the goal being to write a function that
takes in that input, parses it, runs it through an algorithm, and outputs an
answer. Instead of copying the contents of the file and pasting it as a large
string, I prefer to read each line from the file and parse the contents that way.

This led to my first use of Elixir's stream module. According to Elixir's docs,
a Stream is:

> Any enumerable that generates items one by one during enumeration.

So what's the difference between `Enum` and `Stream`? Enums are *eager* loaded, where Streams are *lazy* loaded. This means that if I were to chain
a bunch of Enum functions together, all of those functions will be performed
right away, and return the final value. A Stream can perform the same functions, but the functions will not be performed until necessary.

Because Streams are lazy, it's often a good idea to use them for big sets of data. Let's check out some examples.

## Using Streams

{% highlight ruby %}
  iex(1)> list = [4, 8, 15, 16, 23, 42]
  [4, 8, 15, 16, 23, 42]
  iex> Stream.map(list, &(&1 + 1))
  #Stream<[enum: [4, 8, 15, 16, 23, 42],
   funs: [#Function<46.87278901/1 in Stream.map/2>]]>
{% endhighlight %}

We call `map` on the `Stream` module, passing a list of integers to be streamed, with a function to perform on each item in the list.
As you can see, the return value of this was a struct. We can see that it's storing our list under the key `enum` and the function to apply to it under the key `funs`. But I don't want a struct! I want my data! So I can go ahead and call `to_list` on the result.

{% highlight ruby %}
  iex> Stream.map(list, &(&1 + 1)) |> Enum.to_list  
  [5, 9, 16, 17, 24, 43]
{% endhighlight %}

This is the beginning of something that I like to use to read text files.

{% highlight ruby %}
  File.stream!(file, [:read, :utf8], :line)
  |> Enum.map(&String.trim/1)
  |> Enum.to_list
{% endhighlight %}

Here I create a stream from a file using Elixir's `File` module, use the Enumerable's map function to parse any new line characters from each line, then transform the resulting struct to a list. From here I can use any typical Enumerable function on the data that I need, as I recurse through the list and perform more complicated functions on each piece of data.

# Composable

Streams are composable, in the sense that we can *compose* several functions to them. We can do this with the `Enum` module, but Streams can be more performant when working with huge data sets. Let's say that I have a big list, and from that list I only want to take the last 5 items. One way to
do this is use `Enum.take/2` coupled with `Enum.reverse/1` like so:

{% highlight ruby %}
  list = [] # pretend there's a lot of data in here
  Enum.reverse(list) |> Enum.take(5)
{% endhighlight %}

If we have to reverse this entire list just to get those last 5 elements, we might have some performance issues. A stream could be beneficial here because it will only reverse those 5 elements. Once it's completed all functions passed to it, the job is done.

# Conclusion

All in all, streams are a great option to have in Elixir. I initially used them to open and read text files, however they have a multitude of different uses. The next time you're working with large/infinite data sets, consider using Streams.
