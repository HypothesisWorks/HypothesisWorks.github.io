---
layout: post
tags: python details technical
date: 2017-03-09 11:00
title: Hypothesis for Computer Science Researchers
published: true
author: drmaciver
---

I'm in the process of [trying to turn my work on Hypothesis into a PhD](http://www.drmaciver.com/2017/03/looking-into-starting-a-phd/)
and I realised that I don't have a good self-contained summary as to why researchers should care about it.

So this is that piece. I'll try to give a from scratch introduction to the why and what of Hypothesis. It's primarily
intended for potential PhD supervisors, but should be of general interest as well (especially if you
work in this field).

### Why should I care about Hypothesis from a research point of view?

The short version:

Hypothesis takes an existing effective style of testing (property-based testing) which has proven highly effective in practice
and makes it accessible to a much larger audience. It does so by combining several previously unconnected ideas from the existing
research literature on testing and verification, and combining them to produce a novel implementation that has proven very effective
in practice.

The long version is the rest of this article.

<!--more-->

### What is Hypothesis?

Hypothesis is an implementation of *property-based testing*, an idea that originated with a Haskell library called QuickCheck.

Property-based testing is a way to augment your unit tests with a source of structured random data that allows a tool to explore
the edge cases of your tests and attempt to find errors automatically. I've made a [longer and more formal discussion](http://hypothesis.works/articles/what-is-property-based-testing/)
of this definition in the past. 

An example of a property-based test using Hypothesis:

```python
  from hypothesis import given
  from hypothesis import strategies as st

  @given(st.lists(st.integers()))
  def test_sort_is_idempotent(ls):
    sort1 = sorted(ls)
    assert sorted(sort1) == sort1
```

When this test is run using a standard unit test runner such as py.test, Hypothesis will generate random lists of integers
and pass them to the test. The test sorts the integers, then sorts them again, and asserts that the two results are the same.

As long as the test passes for every input Hypothesis feeds it this will appear to be a normal test. If it fails however,
Hypothesis will then repeatedly rerun it with progressively simpler examples to try and find a minimal input that causes
the failure.

To see this, suppose we implemented the following rather broken implementation of sorted:

```python
  def sorted(ls):
      return list(reversed(ls))
```

Then on running we would see the following output:

```
      @given(st.lists(st.integers()))
      def test_sort_is_idempotent(ls):
        sort1 = sorted(ls)
  >     assert sorted(sort1) == sort1
  E     assert [0, 1] == [1, 0]
  E       At index 0 diff: 0 != 1
  E       Use -v to get the full diff

  sorting.py:12: AssertionError

  ---- Hypothesis ----

  Falsifying example: test_sort_is_idempotent(ls=[0, 1])
```

Hypothesis probably started with a much more complicated example (the test fails for essentially any list with more
than one element) and then successfully reduced it to the simplest possible example: A list with two distinct elements.

This example is found so quickly and reliably that it doesn't really matter, but another important feature is that
is that when an example fails it will replay the last failing example immediately rather than having to find it again.
This is particularly helpful during development, as it means that tests don't stop failing until the bug is fixed and
that when they fail again they fail quickly.

Tests can also draw more data as they execute:

```python
  @given(st.lists(st.integers(), min_size=1), st.data())
  def test_sort_is_idempotent(ls, data):
      ls.sort()
      i = data.draw(st.integers(0, len(ls) - 1))
      assert ls[i - 1] <= ls[i]
```

This fails because we've forgotten than `i` may be zero, and also about Python's negative indexing of lists:

```
      @given(st.lists(st.integers(), min_size=1), st.data())
      def test_sort_is_idempotent(ls, data):
          ls.sort()
          i = data.draw(st.integers(0, len(ls) - 1))
  >       assert ls[i - 1] <= ls[i]
  E       assert 1 <= 0

  sorting.py:15: AssertionError
    
  ---- Hypothesis ----

  Falsifying example: test_sort_is_idempotent(ls=[0, 1], data=data(...))
  Draw 1: 0
```

Simplification and example saving work as normal for data drawn in this way.

Hypothesis also has a form of [model based testing](http://hypothesis.works/articles/rule-based-stateful-testing/),
in which you specify a set of valid operations on your API and it attempts to generate whole programs using those
operations and find a simple one that breaks.

### How is Hypothesis innovative?

From an end user point of view, Hypothesis adds several important things:

* It exists at all and people use it. Historically this sort of testing has been found mostly within the functional
  programming community, and attempts to make it work in other languages have not seen much success or widespread
  adoption. Some of this is due to novel implementation details in Hypothesis, and some is due to design decisions
  making it "feel" like normal testing instead of formal methods.
* Specifying data generators is much easier than in traditional QuickCheck methods, and you get a great deal more
  functionality "for free" when you do. This is similar to [test.check](https://github.com/clojure/test.check) for
  Clojure, but adopts a different and more effective strategy.
* The fact that arbitrary examples can be saved and replayed significantly improves the development work-flow. Other
  implementations of property-based testing either don't do this at all, only save the seed, or rely on being able to
  serialize the generated objects (which can break invariants when reading them back in).
* The fact that you can generate additional data within the test is often extremely useful, and seems to be unique
  to Hypothesis in this category of testing tool.

These have worked together well to fairly effectively bring property based testing "to the masses", and Hypothesis
has started to see increasingly widespread use within the Python community, and is being actively used in the
development of tools and libraries, as well as in the development of both CPython and pypy, the two major implementations
of Python.

Much of this was made possible by Hypothesis's novel implementation.

From an implementation point of view, the novel feature of Hypothesis is this: Unlike other implementations of
property-based testing, it does not need to understand the structure of the data it is generating at all (it sometimes
has to make guesses about it, but its correctness is not dependent on the accuracy of those guesses).

Hypothesis is divided into three logically distinct parts:

1. A core engine called *Conjecture*, which can be thought of as an interactive fuzzer for lightly structured byte streams.
2. A strategy library, which is designed to take Conjecture's output and turn it into arbitrary values representable
   in the programming language.
3. An interface to external test runners that takes tests built on top of the strategy library and runs them using Conjecture
   (in Python this mostly just consists of exposing a function that the test runners can pick up, but in the
   [Java Prototype](http://github.com/HypothesisWorks/hypothesis-java)

Conjecture is essentially the interesting part of Hypothesis's implementation and is what supports most of its functionality:
Generation, shrinking, and serialization are all built into the core engine, so implementations of strategies do not require
any awareness of these features to be correct. They simply repeatedly ask the Conjecture engine for blocks of bytes, which
it duly provides, and they return the desired result. I have written previously
[more details on how Conjecture and Hypothesis relate](http://hypothesis.works/articles/how-hypothesis-works/).

### What prior art is it based on?

I've done a fair bit of general reading of the literature in the course of working on Hypothesis.

The two main papers on which Hypothesis is based are: 

* [QuickCheck: a lightweight tool for random testing of Haskell programs](https://dl.acm.org/citation.cfm?id=351266) essentially
  started the entire field of property-based testing. Hypothesis began life as a QuickCheck implementation, and its user facing
  API continues to be heavily based on QuickCheck, even though the implementation has diverged very heavily from it.
* [EXPLODE: a lightweight, general system for finding serious storage system errors](https://dl.acm.org/citation.cfm?id=1298469)
  provided the key idea on which the Conjecture engine is based - instead of doing static data generation separate from the tests,
  provide tests with an interactive primitive from which they can draw data.

Additionally, the following are major design inspirations in the Conjecture engine, although their designs are not currently
used directly:

* [American Fuzzy Lop](http://lcamtuf.coredump.cx/afl/) is an excellent security-oriented fuzzer, although one without much
  academic connections. I've learned a fair bit about the design of fuzzers from it. For a variety of pragmatic reasons I don't
  currently use its most important innovation (branch coverage metrics as a tool for corpus discovery), but I've successfully
  prototyped implementations of that on top of Hypothesis which work pretty well.
* [Swarm Testing](https://dl.acm.org/citation.cfm?id=2336763) drove a lot of the early designs of Hypothesis's data generation.
  It is currently not explicitly present in the Conjecture implementation, but some of what Conjecture does to induce deliberate
  correlations in data is inspired by it.

### What are some interesting research directions?

I have a large number of possible directions that my work on Hypothesis could be taken - far more than I can practically
work on, so a lot of the most important work I would be doing as part of a PhD is prioritizing and figuring out which ones
are the most promising.

The following are what I think of as the four most interesting directions to go next:

#### More structured byte streams

My current immediate research focus on Hypothesis is to replace the core Conjecture primitive with a more structured one
that bears a stronger resemblance to its origins in EXPLODE. This is designed to address a number of practical problems
that Hypothesis users currently experience, but it also opens up a number of other novel abstractions that can be built
on top of the core engine.

The idea is to pare down the interface so that when calling in to Conjecture you simply draw a single byte, specifying
a range of possible valid bytes. This gives Conjecture much more fine-grained information to work with, which opens up
a number of additional features and abstractions that can be built on top of it.

From this primitive you can then rebuild arbitrary weighted samplers that shrink correctly (using a variation of the
Alias Method), and arbitrary grammars (probably using Boltzmann Samplers or similar).

#### Glass box testing

Currently Conjecture treats the tests it calls as a black box and does not use coverage information. The primary
reason for this so far has been pragmatics - all of the techniques I've found have worked well when tests are allowed
to run for minutes or hours, but the current design focus of Hypothesis assumes tests have seconds to run at most,
which limits the utility of these methods and means they haven't been a priority so far.

But in principle this should be an extremely profitable line of attack, even with that limitation, and I would like
to explore it further.

The main idea would be to add a notion of "tags" to the core Conjecture engine which could be used to guide the
search. Coverage would be one source of tags, but others are possible. For example, my previous work on
[Schroedinteger](https://github.com/DRMacIver/schroedinteger) implements what is essentially a form of lightweight
[Concolic testing](https://en.wikipedia.org/wiki/Concolic_testing) that would be another possibly interesting source
of information to use.

#### Making the Conjecture engine smarter

A thing I've looked into in the past is the possible use of grammar inference to improve shrinking and data generation.

At the time the obstacle I ran into was that the algorithm I was using -
[an optimized variation of L\* search](https://dl.acm.org/citation.cfm?id=73047) - did not get good performance in
practice on the problems I tried it on.

[Synthesizing Program Input Grammars](https://arxiv.org/abs/1608.01723) promises to lift this restriction by providing
much better grammar inference in practical scenarios that are quite closely related to this problem domain, so I would
like to revisit this and see if it can prove useful.

#### Other testing abstractions

Despite Hypothesis primarily being a library for property based testing, the core Conjecture engine actually has
very little to do with property-based testing and is a more powerful low-level testing abstraction. It would be interesting
to see how far that could be taken - the existing stateful/model-based testing is one partial step in that direction,
but it could also potentially be used more directly for other things. e.g. in tandem with some of the above features
it could be used for low-level fuzzing of binaries, or using it to drive thread scheduling.

The nice thing about the Conjecture separation is that because it is so self-contained, it can be used as the core
building block on which other tools can be rebuilt and gain a lot of its major features for free.

I don't currently have any concrete plans in this direction, but it seems likely there are some interesting possibilities
here that will emerge after more review of the testing literature.

### What should you do with this information?

It depends who you are.

* If I'm already talking to you because you're a potential PhD supervisor, tell me what about this interests you and
  ask me lots of questions.
* If you're a potential PhD supervisor who I'm *not* already talking to but you'd like me to, please let me know!
* If you're somebody else, it's rather up to you. Feel free to send me papers, questions, etc.

Whoever you are, if you found this document interesting I'd love to hear from you. Drop me an email at
[david@drmaciver.com](mailto:david@drmaciver.com).
