---
layout: post
title: "Go by Example: Sort by Function"
---

# {{page.title}}

<span class="meta">September 29 2012</span>

_If you're interested in Go, be sure to check out [Go by Example](https://gobyexample.com)._

Sorting a slice by a function is a bit tricker in Go than you
may be used to in other languages. Let's look at some examples of
sorting in Go to see how it works.


### Natural Sort

Before we try sorting by arbitrary functions, let's look at natural
order sorting. Here's an example of sorting strings alphabetically:

{% highlight go %}
package main

import "sort"
import "fmt"

func main() {
  fruits := []string{"peach", "banana", "kiwi"}
  sort.Strings(fruits)
  fmt.Println(fruits)
}
{% endhighlight %}

We can put this code in `sort1.go` and run it with `go run`:

{% highlight console %}
$ go run sort1.go
[banana kiwi peach]
{% endhighlight %}

With that baseline, let's look at custom sorting.


### Custom Sort

Sometimes we'll want to sort a collection by something other than
its natural order. For example, suppose we wanted to sort our
fruit strings by their length instead of alphabetically. Here's
an example of this in Go:

{% highlight go %}
package main

import "sort"
import "fmt"

type ByLength []string

func (s ByLength) Len() int {
    return len(s)
}
func (s ByLength) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}
func (s ByLength) Less(i, j int) bool {
    return len(s[i]) < len(s[j])
}

func main() {
    fruits := []string{"peach", "banana", "kiwi"}
    sort.Sort(ByLength(fruits))
    fmt.Println(fruits)
}
{% endhighlight %}

Try this out from `sort2.go`:

{% highlight console %}
$ go run sort2.go
[kiwi peach banana]
{% endhighlight %}

In order to sort by a custom function in Go, we need a corresponding
type. Here we've created a `ByLength` type that is just an alias for
same `[]string` type we sorted in the previous example.

We implement `sort.Interface` on our type so we can use the `sort`
package's generic `Sort` function. These interface methods are the
`Len`, `Less`, and `Swap` functions above `main` in the example.

Typically `Len` and `Swap` will take the form above, and `Less` is
where we put our custom function to determine sort order. In our
case we wanted to sort in order of increasing string length, so
our `Less` function used `len(s[i])` and `len(s[j])`.

With all of this in place, we can cast our original `fruits`
collection to `ByLength`, and then use `sort.Sort` to sort that
collection with our custom logic.

By following this same pattern of creating a custom type,
implementing the three `Interface` methods on that type, and then
calling `sort.Sort` on a collection of that custom type, we can
sort Go slices by arbitrary functions.


-----

Thanks to [Keith Rarick](http://xph.us/) and [Blake Mizerany](https://github.com/bmizerany)
for reading drafts of this post.
