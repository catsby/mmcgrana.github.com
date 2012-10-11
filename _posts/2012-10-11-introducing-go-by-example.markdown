---
layout: post
title: Introducing Go by Example
---

# {{page.title}}

<span class="meta">October 11 2012</span>

Today we're introducing [Go by Example](https://gobyexample.com),
a hands-on guide to [Go](http://golang.org). Go by Example
helps you learn and write Go through annotated example programs.

The site already has several examples, including:

* [Arrays](https://gobyexample.com/arrays), [Slices](https://gobyexample.com/slices), and [Maps](https://gobyexample.com/maps)
* [Goroutines](https://gobyexample.com/goroutines), [Channels](https://gobyexample.com/channels), and [Select](https://gobyexample.com/select)
* [Timers](https://gobyexample.com/timers) and [Tickers](https://gobyexample.com/tickers)
* [Sorting](https://gobyexample.com/sorting) and [Sorting by Functions](https://gobyexample.com/sorting-by-functions)
* [Environment Variables](https://gobyexample.com/environment-variables) and [Signals](https://gobyexample.com/signals)
* [Spawning Processes](https://gobyexample.com/spawning-processes) and [Exec'ing Processes](https://gobyexample.com/execing-processes)
* [SHA1 Hashes](https://gobyexample.com/sha1-hashes) and [Line Filters](https://gobyexample.com/line-filters)

Read on to learn more about why and how we made Go by
Example and what's coming next.


## Example-Based Approach

Programmers learn to program by reading and writing
programs. Go by Example therefore tries to give you:

* Explanations paired with code you're reading - letting
  code speak for itself when it's clear and providing
  context when it's needed.
* Code that enables you to solve problems you have when
  writing your own programs.

We have more than 15 published examples and [70 more](https://github.com/mmcgrana/gobyexample/tree/master/examples)
in development.


## Open

In order to best enable developers to access, use, and
re-mix the content, Go by Example is released under a
liberal [Creative Commons license](https://github.com/mmcgrana/gobyexample#license).

The source code is [available on Github](https://github.com/mmcgrana/gobyexample)
and we've already merged a pull request from a helpful reader
of the alpha version.


## Go Everywhere

The Go by Example site is derived almost entirely from
`.go` source files (code + comments), rendered to HTML
with a [Go toolchain](https://github.com/mmcgrana/gobyexample/tree/master/tools),
and served online with a [Go web server](https://github.com/mmcgrana/gobyexample-server).

For example, here is the [line filter Go source](https://github.com/mmcgrana/gobyexample/blob/master/examples/line-filters/line-filters.go) and
corresponding [rendered page](https://gobyexample.com/line-filters).


## Designed

Go by Example strives to provide the best experience
possible for developers learning Go:

* Code and docs are typeset to ~60 characters per column
  to maximize legibility.
* Minimal styling keeps the focus on the docs and code.
* Code is highlighted with the excellent [Pygments](http://pygments.org/)
  library.
* The site renders beautifully on both laptops and [iPads](http://f.cl.ly/items/1F1u2I0Y1t1S1e1F3O0l/gobyexample-line-filter-ipad.png).

We hope you find Go by Example a joy to read.


## What's Next

Go by Example is an early experiment. We're interested
in learning how developers find it useful, evolving the
model, and shipping more examples that benefit developers.

If there's an example you'd like to see or if you have
any other feedback, let us know at [@gobyexample](https://twitter.com/gobyexample)
or email the author at `mmcgrana@gmail.com`.

We're excited to share Go by Example and hope that you
[check it out](https://gobyexample.com)

To learn about new examples and other updates on the site,
follow [@gobyexample](https://twitter.com/gobyexample).
