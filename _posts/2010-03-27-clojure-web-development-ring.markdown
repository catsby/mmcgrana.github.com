---
layout: post
title: Clojure Web Development with Ring
---

# {{page.title}}

<span class="meta">March 27 2010</span>

This post provides an introduction Clojure web development with [Ring](https://github.com/mmcgrana/ring), a Clojure web application library. If you are just getting started with Clojure web development or if you are interested in the new Ring 0.2 release, this is a great place to start.

Before getting started, ensure that you have [Leiningen 1.1.0](https://github.com/technomancy/leiningen) installed. We will be using Leiningen via the `lein` command to manage dependencies. Check your Leiningen install with:

{% highlight sh %}
lein -v
{% endhighlight %}

Once you have Leiningen set up, create a new `ring-tutorial` project:

{% highlight sh %}
lein new ring-tutorial
cd ring-tutorial
{% endhighlight %}

Now edit the `project.clj` so that it looks like this:

{% highlight clj %}
(defproject ring-tutorial "1.0.0-SNAPSHOT"
  :description "An Ring tutorial project"
  :dependencies
    [[org.clojure/clojure "1.1.0"]
     [org.clojure/clojure-contrib "1.1.0"]
     [ring/ring-core "0.2.0"]
     [ring/ring-jetty-adapter "0.2.0"]]
  :dev-dependencies
    [[ring/ring-devel "0.2.0"]])
{% endhighlight %}

With the dependencies specified, pull them into the project with:

{% highlight sh %}
lein deps
{% endhighlight %}

Notice that we depended on `ring-core`, `ring-devel`, and `ring-jetty-adapter`. `ring-core` provides basic web application libraries that will be useful for all Ring projects. `ring-devel` contains debugging tools that will be useful locally but which won't need to be deployed with your application. `ring-jetty-adapter` contains the Ring bindings to Jetty, which will serve our tutorial app.

Let's start with a very simple "hello world" application. Put the following in `src/ring_tutorial/core.clj`:

{% highlight clj %}
(ns ring-tutorial.core
  (:use ring.adapter.jetty))

(defn handler [req]
  {:status  200
   :headers {"Content-Type" "text/html"}
   :body    "Hello World from Ring"})

(defn boot []
  (run-jetty #'handler {:port 8080}))
{% endhighlight %} 

To run the app, first start a REPL with:

{% highlight sh %}
lein repl
{% endhighlight %}

Then in the REPL, use the `boot` function defined above:

{% highlight clj %}
(use 'ring-tutorial.core)
(boot)
{% endhighlight %}

Now you can navigate to [`http://localhost:8080`](http://localhost:8080) and see your live Ring app.

Note that if you change the implementation of `ring-tutorial.core/handler` and want to see the effects, you'll need to CTR-C the REPL and re-`boot` the server. To avoid having to do this repeatedly, let's use `ring.middleware.reload` to automatically reload changes between requests. Update `src/ring_tutorial/core.clj` to the following:

{% highlight clj %}
(ns ring-tutorial.core
  (:use ring.adapter.jetty)
  (:use ring.middleware.reload))

(defn handler [req]
  {:status  200
   :headers {"Content-Type" "text/html"}
   :body    "Hello World from Ring"})

(def app
  (wrap-reload #'handler '(ring-tutorial.core)))

(defn boot []
  (run-jetty #'app {:port 8080}))
{% endhighlight %}

And boot the server as before. Now you should be able to change e.g. the `:body` of the response and see it reflected after refreshing the page.

While making quick changes to the application, you may introduce errors. For example, suppose you changed the definition of `handler` to the following:

{% highlight clj %}
(defn handler [req]
  ("oops"))
{% endhighlight %}

Now, refreshing the page will cause a stacktrace to show up your REPL. To make this stacktrace more readable, we can use `ring.middleware.stacktrace`. Update the definition of `app` to the following:

{% highlight clj %}
(def app
  (-> #'handler
    (wrap-reload '(ring-tutorial.core))
    (wrap-stacktrace)))
{% endhighlight %}

Also, update the `ns` declaration to include the new library:

{% highlight clj %}
(ns ring-tutorial.core
  (:use ring.adapter.jetty)
  (:use ring.middleware.reload)
  (:use ring.middleware.stacktrace))
{% endhighlight %}

Refresh the page a few times. Now stacktraces are displayed nicely in the browser instead of being sent to the REPL. Notice also that the reloader is still working; if you revert to the definition of `handler` to its original form, the "hello world" message will appear on subsequent refreshes.

To review, here is what we ended up with:

{% highlight clj %}
(ns ring-tutorial.core
  (:use ring.adapter.jetty)
  (:use ring.middleware.reload)
  (:use ring.middleware.stacktrace))

(defn handler [req]
  {:status  200
   :headers {"Content-Type" "text/html"}
   :body    "Hello World from Ring"})

(def app
  (-> #'handler
    (wrap-reload '(ring-tutorial.core))
    (wrap-stacktrace)))

(defn boot []
  (run-jetty #'app {:port 8080}))
{% endhighlight %}
   
There is a lot more to say about Ring and Clojure web development, but I hope this tutorial at least gets you started. If you'd like to learn more about Ring, visit the [Ring GitHub project](https://github.com/mmcgrana/ring), join the [Ring Google Group](http://groups.google.com/group/ring-clojure), check out the detailed [Ring namespace and function documentation](http://mmcgrana.github.com/ring/) available online, and read the [Ring spec](https://github.com/mmcgrana/ring/raw/master/SPEC).
