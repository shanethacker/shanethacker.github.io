---
layout: post
title: Calling Erlang from Elixir
category: programming
tags: [elixir]
---

Lately I've been working with Elixir, a new-ish language based on the Erlang VM (BEAM). Elixir is designed to take advantage of Erlang's parallel processing power while presenting the developer with less intimidating syntax.

I'm not going to spend any time here trying to sell Elixir. [Elixir-lang.org](http://elixir-lang.org/) does that well, and much like with Ruby on Rails, Elixir has the [Phoenix Framework](http://www.phoenixframework.org/) leading the way to greater adoption. I will say I got it right off the bat better than any other functional language I've seen.

I've been doing programming exercises from [Exercism.io](http://exercism.io/), which is a great resource for a lot of different languages. While doing the Gigasecond exercise, I had to drop down into the Erlang weeds for some time/date functionality. (Erlang and Elixir relate much as Java and Jython do.)

This exercise calculates the date one billion seconds after a given date. While there might be some Elixir-native libraries that can do this sort of calculation by now, I decided to use Erlang's [Calendar module](http://erlang.org/doc/man/calendar.html) to transform the data.

```elixir
def from({year, month, day}) do
  :calendar.datetime_to_gregorian_seconds($${{year, month, day}, {0, 0, 0}}$$)
  |> + 1000000000
  |> :calendar.gregorian_seconds_to_datetime
  |> elem(0)
end
```

Going through it step-by-step:

The function is called `from/1` -- defined by the test module Exercism supplied -- and accepts a tuple as an argument with the year, month, and day.

(You'll see a lot of *function/n* notation in Elixir. It means the *function* takes *n* arguments. You can have functions with the same name that take different numbers of arguments...but it's more complex than that. Elixir does pattern matching to determine which function to call. We'll talk about that in the future.)

In order to call an Erlang function, you have to call its module using the syntax for an atom, an Elixir constant with a value of itself. In this case, calling the Calendar module is done with `:calendar`, and invoking the function is through the oft-used dot notation.

We'll get a new value from the passed-in date -- Elixir variables are generally intended to be immutable -- by passing the date into the `:calendar.datetime_to_gregorian_seconds/1` function, which should return the number of seconds since year 0.

In this case, Erlang's `:calendar.datetime_to_gregorian_seconds/1` function takes one argument, a datetime, which is represented as a tuple consisting of a date tuple `{year, month, day}` and time tuple `{hour, minute, second}`. We only care about the date, and not the time, so we pass in {0, 0, 0} as the time.

Next, we pipe the return value of the Erlang function, using the pipe operator `|>`, to the next step, which returns a new value with 1000000000 added to the previous value. So, the number of seconds since year 0 plus 1 billion. And yes, the pipe operator is very similar to the Unix version.

Passing that new value to another function in the Erlang calendar module: `:calendar.gregorian_seconds_to_datetime/1`, which takes in an integer value of seconds and converts it back to a datetime calculated from year 0.

We get back a datetime -- which, remember, is a tuple of a date tuple and time tuple -- but we don't care about the time. So, we use `elem/2` to get back the 0th element of the datetime tuple.

But wait? elem/2? Elem takes a tuple as the first argument, and the index position as the second. Since we are piping the datetime tuple into the elem/2 function, the tuple ends up as the first argument even when not explicitly written out, and the supplied 0 becomes the second argument.

What we get back is the date tuple, and since that is the last value before the end, `from/1` returns that date tuple to the calling function.

I mostly wrote this out to remind myself that Erlang modules are available as atoms in Elixir, but I like it as an example of piping through a series of data transformations as well.

I found this very helpful: [Calling Erlang code from Elixir: a tale of hubris, strings, and how to read the documentation](http://nickcanzoneri.com/elixir/erlang/2015/08/03/calling-erlang-code-from-elixir.html).

The nice thing about Elixir is that there are a few other ways to do this same thing, and as near as I can tell, they all work about as efficiently. So, you can write code in the way that you find easiest to read.

Stay tuned for more adventures in Elixir.