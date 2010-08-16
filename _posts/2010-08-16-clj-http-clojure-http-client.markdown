---
layout: post
title: Introducing clj-http, a Clojure HTTP Client
---

# {{page.title}}

<span class="meta">August 16 2010</span>

[`clj-http`](http://github.com/clj-sys/clj-http) is a new Clojure HTTP client library inspired by Ring and designed for simplicity, robustness, extensibility, and testability.

For basic use cases, it provides a high-level interface:

{% highlight clj %}
(require '[clj-http.client :as client])

(client/get "http://rest-test.heroku.com/")
=> {:status 200
    :headers {"date" "Sun, 01 Aug 2010 07:03:49 GMT"
              "cache-control" "private, max-age=0, must-revalidate"
              "content-type" "text/html; charset=utf-8"
              ...}
    :body "GET http://rest-test.heroku.com/"}
{% endhighlight %}

This interface accepts various options. For example:

{% highlight clj %}
(:body (client/post "http://rest-test.heroku.com/"
         {:body "body data" :content-type "text/plain"}))
=> "PUT http://rest-test.heroku.com/ with a 9 byte payload,
    content type text/plain; charset=UTF-8"
{% endhighlight %}

The library also provides a more general interface for use in building domain-specific client libraries:

{% highlight clj %}
(defn api-request [method path body]
  (:body
    (client/request
      {:method method
       :url (str "http://rest-test.heroku.com" path)
       :content-type "text/plain"
       :body body})))

(api-request :post "/resource" "data")
=> "POST http://rest-test.heroku.com/resource with a 4 byte payload,
    content type text/plain; charset=UTF-8"
{% endhighlight %}

`clj-http` uses a similar architecture to the [Ring](http://github.com/mmcgrana/ring) Clojure HTTP server library. It defines a simple interface abstracting the HTTP request/response cycle and implements all additional features through this interface. For example, this is how `clj-http.client/request` is defined:

{% highlight clj %}
(def request
  (-> #'core/request
    wrap-redirects
    wrap-exceptions
    wrap-decompression
    wrap-input-coercion
    wrap-output-coercion
    wrap-query-params
    wrap-basic-auth
    wrap-accept
    wrap-accept-encoding
    wrap-content-type
    wrap-method
    wrap-url))
{% endhighlight %}

`clj-http` is still in the early stages of development, but is ready for testing by interested Clojure users. Give it a shot by including it in your applications with:

{% highlight clj %}
:dependencies
  [[clj-http "0.1.0"] ...]
{% endhighlight %}

You can learn more about the library on its [project page](http://github.com/clj-sys/clj-http). I look forward to your hearing your thoughts, suggestions, and criticisms. Let me know what you think in the comments below or on the [Clojure mailing list](http://groups.google.com/group/clojure).

If you are interested in Clojure HTTP libraries in general, you should also check out [`clojure-http-client`](http://github.com/technomancy/clojure-http-client), [`clj-apache-http`](http://github.com/rnewman/clj-apache-http), and [`ahc-clj`](http://github.com/neotyk/ahc-clj).

Thanks to the [Bay Area Clojure Users Group](http://www.meetup.com/The-Bay-Area-Clojure-User-Group/) for the initial Ring-style-HTTP-client suggestion, to Daniel Janus for [another nudge](http://groups.google.com/group/ring-clojure/browse_thread/thread/c86557f0cd0b7a02) in this direction, to [Adam Wiggins](http://adam.heroku.com/) for the inspiration of [rest-client](http://rdoc.info/projects/archiloque/rest-client) and the `rest-test.heroku.com` site, and to [Bradford Cross](http://measuringmeasures.com/) for design and implementation collaboration. 
