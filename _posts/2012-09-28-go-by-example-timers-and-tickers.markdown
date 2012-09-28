---
layout: post
title: "Go by Example: Timers and Tickers"
---

# {{page.title}}

<span class="meta">September 28 2012</span>

We often want to execute Go code at some point in the future, or
repeatedly at some interval. Go's built-in timer and ticker features
make both of these tasks easy. Let's look at some examples to see
how they work.


### Timers

Timers represent a single event in the future. You tell the timer
how long you want to wait, and it gives you a channel that will
be notified at that time. For example, to wait 2 seconds:

{% highlight go %}
package main

import "time"

func main() {
    timer := time.NewTimer(time.Second * 2)
    <- timer.C
    println("Timer expired")
}
{% endhighlight %}

The `<- timer.C` blocks on the timer's channel `C` until it sends a
value indicating that the timer expired. You can test this by putting
the code in `timers1.go` and running it with `time` and `go run`:

{% highlight console %}
$ time go run timers1.go
Timer expired
real  0m2.113s
{% endhighlight %}

If you just wanted to wait, you could have used `time.Sleep`. One
reason a timer may be useful is that you can cancel the timer before
it expires. For example, try running this in `timers2.go`:

{% highlight go %}
package main

import "time"

func main() {
    timer := time.NewTimer(time.Second)
    go func() {
        <- timer.C
        println("Timer expired")
    }()
    stop := timer.Stop()
    println("Timer cancelled:", stop)
}
{% endhighlight %}

{% highlight console %}
$ go run timers2.go
Timer cancelled: true
{% endhighlight %}

In this case we canceled the timer before it had a chance to expire
(`Stop()` would return false if we tried to cancel it after it
expired.)


### Tickers

Timers are for when you want to do something once in the future -
tickers are for when you want to do something repeatedly at regular
intervals. Here's a basic example:

{% highlight go %}
package main

import "time"
import "fmt"

func main() {
    ticker := time.NewTicker(time.Millisecond * 500)
    go func() {
        for t := range ticker.C {
            fmt.Println("Tick at", t)
        }
    }()
    time.Sleep(time.Millisecond * 1500)
    ticker.Stop()
    fmt.Println("Ticker stopped")
}
{% endhighlight %}

Try it out in `tickers.go`:

{% highlight console %}
$ go run tickers.go
Tick at 2012-09-22 15:58:40.912926 -0400 EDT
Tick at 2012-09-22 15:58:41.413892 -0400 EDT
Tick at 2012-09-22 15:58:41.913888 -0400 EDT
Ticker stopped
{% endhighlight %}

Tickers can be stopped just like timers using the `Stop()` method, as
shown at the bottom of the example.


### Power of Channels

A great feature of Go's timers and tickers is that they hook into
Go's built-in concurrency mechanism: channels. This allows timers
and tickers to interact seamlessly with other concurrent goroutines.
For example:

{% highlight go %}
package main

import "time"
import "fmt"

func main() {
    timeChan := time.NewTimer(time.Second).C
    
    tickChan := time.NewTicker(time.Millisecond * 400).C
    
    doneChan := make(chan bool)
    go func() {
        time.Sleep(time.Second * 2)
        doneChan <- true
    }()
    
    for {
        select {
        case <- timeChan:
            fmt.Println("Timer expired")
        case <- tickChan:
            fmt.Println("Ticker ticked")
        case <- doneChan:
            fmt.Println("Done")
            return
      }
    }
}
{% endhighlight %}

Try it out by running this code form `channels.go`:

{% highlight console %}
$ go run channels.go
Ticker ticked
Ticker ticked
Timer expired
Ticker ticked
Ticker ticked
Ticker ticked
Done
{% endhighlight %}

In this case we see the timer, ticker, and our own custom goroutine
all participating in the same `select` block. The combination of the
Go runtime, its channel feature, and timers/tickers use of channels
make this kind of cooperation very natural in Go.

You can learn more about timers, tickers, and other time-related Go
features in the [Go `time` package docs](http://golang.org/pkg/time/).


### More Go Examples

For more Go examples, follow [@gobyexample](https://twitter.com/gobyexample)
on Twitter. If there is something you'd like to see an example of,
let me know!

Also, check out our previous example: [Getting started with Go and Heroku](https://mmcgrana.github.com/...).


-----

Thanks to [Keith Rarick](http://xph.us/) and [Blake Mizerany](https://github.com/bmizerany)
for reading drafts of this post.
