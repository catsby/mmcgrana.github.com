---
layout: post
title: Getting Started with Go on Heroku
---

# {{page.title}}

<span class="meta">September 25 2012</span>

_If you're interested in Go, be sure to check out [Go by Example](https://gobyexample.com)._

[Go](http://golang.org/) is a general-purpose language for building
simple, reliable, and fast software. It's fun to write and a good
fit for many use cases including web apps, network services, and
command-line tools.

This quickstart will show you how to run Go apps and deploy them to
the [Heroku](http://www.heroku.com) cloud platform.

We'll start with installing Go and running a hello world program.
Then we'll set up a Go environment, build a Go web app, and deploy
that app to Heroku.


### Go Install

To install Go, visit the [official downloads page](http://code.google.com/p/go/downloads/list)
and follow the link for your OS.

To test your install, open a new terminal window and try the `go`
command:

{% highlight console %}
$ go version
go version go1.0.3
{% endhighlight %}

Now let's write our first Go program.


### Hello World

Here's our hello world program. Put the code in `hello.go`:

{% highlight go %}
package main

import "fmt"

func main() {
    fmt.Println("hello, world")
}
{% endhighlight %}

And try it with `go run`:

{% highlight console %}
$ go run hello.go
hello, world
{% endhighlight %}

Using `go run` is great for running small test programs. To
pre-compile Go apps you'll want to setup a proper Go environment.
We'll look at that next.


### Go Environment

Go tools expect your code to be arranged according to Go conventions.
Following these conventions makes Go easier to use. The full details
are on [golang.org](http://golang.org/doc/code.html), but here's the
quick version.

Make a directory for your Go code:

{% highlight console %}
$ mkdir -p $HOME/go/src
{% endhighlight %}

Tell the Go tools where this is with a `GOPATH` environment
variable sourced in your `.bash_profile` (or `.bash_login`, etc.):

{% highlight console %}
$ echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
$ echo 'export PATH=$PATH:$GOPATH/bin'  >> $HOME/.bash_profile
{% endhighlight %}

Now open a new terminal window and try:

{% highlight console %}
$ echo $GOPATH
/Users/you/go
{% endhighlight %}

If you see something `echo`'d for `$GOPATH` like above then you're all
set.

With this in place let's try building a Go web app.


### Go Web App

Our Go web app will be an HTTP version of the hello world we saw
above. Start by creating a directory for your app:

{% highlight console %}
$ mkdir $GOPATH/src/demoapp
$ cd $GOPATH/src/demoapp
{% endhighlight %}

Here is the code for the app. Put it in `web.go`:

{% highlight go %}
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    http.HandleFunc("/", hello)
    fmt.Println("listening...")
    err := http.ListenAndServe(":"+os.Getenv("PORT"), nil)
    if err != nil {
      panic(err)
    }
}

func hello(res http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(res, "hello, world")
}
{% endhighlight %}

Build this code into an executable binary with `go get`:

{% highlight console %}
$ go get
{% endhighlight %}

Check for the new binary with `which`:

{% highlight console %}
$ which demoapp
/Users/you/go/bin/demoap
{% endhighlight %}

Now try running the program:

{% highlight console %}
$ PORT=5000 demoapp
listening...
{% endhighlight %}

And making an HTTP request to it:

{% highlight console %}
$ curl -i http://127.0.0.1:5000/
HTTP/1.1 200 OK
Date: Sat, 22 Sep 2012 15:30:33 GMT
Transfer-Encoding: chunked
Content-Type: text/plain; charset=utf-8

hello, world
{% endhighlight %}

Great, it worked! Now let's try deploying this app to Heroku so that
we can see it running live.


### Heroku Setup

In order to deploy to Heroku you'll need a Heroku user account.
[Signup is free and instant](https://api.heroku.com/signup).

You'll also need a Heroku command-line client. Get it by installing
the [Heroku Toolbelt](https://toolbelt.heroku.com/) if you haven't
already.

If this is your first time using Heroku, login and upload your
SSH key:

{% highlight console %}
$ heroku login
Enter your Heroku credentials.
Email: you@example.com
Password:
Uploading ssh public key /Users/you/.ssh/id_rsa.pub
{% endhighlight %}

With that in place, let's prepare our app for deploying to Heroku.


### Git and Procfile

In order to deploy to Heroku we'll need the app stored in Git:

{% highlight console %}
$ git init
$ git add -A .
$ git commit -m code
{% endhighlight %}

We'll also need a `Procfile` to tell Heroku what command to run
for the `web` process in our app:

{% highlight console %}
$ echo 'web: demoapp' > Procfile
{% endhighlight %}

Finally, we need to tell Heroku the directory the app usually runs
in, since this local context won't be available to Heroku:

{% highlight console %}
$ echo 'demoapp' > .godir
{% endhighlight %}

Add these new files to git:

{% highlight console %}
$ git add -A .
$ git commit -m metadata
{% endhighlight %}

Now we're ready to ship this to Heroku.


### Heroku Deploy

Create a new Heroku app, telling it to use the [Go Heroku Buildpack](https://github.com/kr/heroku-buildpack-go.git)
to build your Go code:

{% highlight console %}
$ heroku create -b https://github.com/kr/heroku-buildpack-go.git
Creating ancient-temple-243... done, stack is cedar
Git remote heroku added
{% endhighlight %}

Push the code to Heroku:

{% highlight console %}
$ git push heroku master
-----> Heroku receiving push
-----> Fetching custom git buildpack... done
-----> Go app detected
-----> Installing Go 1.0.2... done
       Installing Virtualenv... done
       Installing Mercurial... done
       Installing Bazaar... done
-----> Running: go get ./...
-----> Discovering process types
       Procfile declares types -> web
-----> Compiled slug size: 1.0MB
-----> Launching... done, v4
       http://ancient-temple-243.herokuapp.com deployed to Heroku
{% endhighlight %}

Your app should be up and running. Visit it with:

{% highlight console %}
$ heroku open
{% endhighlight %}

If everything went OK, you should have 1 web process up:

{% highlight console %}
$ heroku ps
=== web: `demoapp`
web.1: up 2012/09/22 12:29:08 (~ 1m ago)
{% endhighlight %}

You can also check your logs to see watch as requests come in:

{% highlight console %}
$ heroku logs --tail
heroku[api]: Deploy 75a0292 by you@example.com
heroku[slugc]: Slug compilation finished
heroku[web.1]: Starting process with command `demoapp`
app[web.1]: listening...
heroku[web.1]: State changed from starting to up
heroku[router]: GET ancient-temple-243.herokuapp.com/ ...
{% endhighlight %}

That's it - you now have a running Go app on Heroku!

-----

Thanks to fellow Herokai [Keith Rarick](http://xph.us/), [Blake Mizerany](https://github.com/bmizerany),
and [Ryan Smith](http://ryandotsmith.heroku.com/) for
their work on bringing Go to Heroku, on the the Go
buildpack, and for the [original quickstart](https://gist.github.com/299535bbf56bf3016cba)
on which this article is based.
