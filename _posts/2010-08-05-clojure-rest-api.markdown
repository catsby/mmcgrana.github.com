---
layout: post
title: Building REST APIs for Clojure Web Applications
---

# {{page.title}}

<span class="meta">August 5 2010</span>

One common requirement for Clojure web applications is to provide REST APIs. REST APIs allow non-Clojure components to access the app's data and also facilitate loose coupling with other Clojure components. JSON is a convenient data format to use in these REST services because it maps well to Clojure's native data structures. In this post I'll show how to write a simple JSON-over-HTTP REST service in Clojure.

If you haven't seen the previous post on [developing and deploying a simple Clojure web app](http://mmcgrana.github.com/2010/07/develop-deploy-clojure-web-applications.html), you should read that first.

We'll start on this app by creating a directory `cabinet` and adding a `project.clj` file to specify the app's dependencies:

{% highlight clj %}
(defproject cabinent "0.0.1"
  :description "REST datastore interface."
  :dependencies
    [[org.clojure/clojure "1.2.0-RC1"]
     [org.clojure/clojure-contrib "1.2.0-RC1"]
     [ring/ring-jetty-adapter "0.2.5"]
     [ring-json-params "0.1.0"]
     [compojure "0.4.0"]
     [clj-json "0.2.0"]]
  :dev-dependencies
    [[lein-run "1.0.0-SNAPSHOT"]])
{% endhighlight %}

Now we'll make a quick "hello world" JSON service. Put the following in `src/cabinet/web.clj`:

{% highlight clj %}
(ns cabinet.web
  (:use compojure.core)
  (:use ring.middleware.json-params)
  (:require [clj-json.core :as json]))

(defn json-response [data & [status]]
  {:status (or status 200)
   :headers {"Content-Type" "application/json"}
   :body (json/generate-string data)})

(defroutes handler
  (GET "/" []
    (json-response {"hello" "world"}))

  (PUT "/" [name]
    (json-response {"hello" name})))

(def app
  (-> handler
    wrap-json-params))
{% endhighlight %}

Add create a script called `run.clj` with:

{% highlight clj %}
(use 'ring.adapter.jetty)
(require '[cabinet.web :as web])

(run-jetty #'web/app {:port 8080})
{% endhighlight %}

Now start the app by typing:

{% highlight sh %}
$ lein deps
$ lein run run.clj
{% endhighlight %}

You can test the app with `curl`:

{% highlight sh %}
$ curl -X GET http://localhost:8080/
{"hello":"world"}

$ curl -X PUT -H "Content-Type: application/json" \
    -d '{"name": "hacker"}' \
    http://localhost:8080/
{"hello":"hacker"}
{% endhighlight %}

Now that we have the JSON-over-HTTP basics proofed out, lets try creating a proper REST service. Our app will expose a set of "elements" with operations for listing, getting, putting, and deleting. Put the following code in `src/cabinet/elem.clj`:

{% highlight clj %}
(ns cabinet.elem
  (:refer-clojure :exclude (list get delete)))

(def elems (atom {}))

(defn list []
  @elems)

(defn get [id]
  (@elems id))

(defn put [id attrs]
  (let [new-attrs (merge (get id) attrs)]
    (swap! elems assoc id new-attrs)
    new-attrs))

(defn delete [id]
  (let [old-attrs (get id)]
    (swap! elems dissoc id)
    old-attrs))
{% endhighlight %}

Using the `elem` model directly from Clojure would look like:

{% highlight clj %}
(require '[cabinet.elem :as elem])

(elem/list)
=> {}

(elem/put "1" {"tag" "foo"})
=> {"tag" "foo"}

(elem/get "1")
=> {"tag" "foo"}

(elem/put "1" {"group" "bar"})
=> {"tag" "foo", "group" "bar"}

(elem/put "2" {"tag" "bat", "group" "biz"})
=> {"tag" "bat", "group" "biz"}

(elem/delete "1")
=> {"tag" "foo", "group" "bar"}

(elem/list)
=> {"2" {"tag" "bat", "group" "biz"}}
{% endhighlight %}

To expose this model with a REST service, first require the new `cabinet.elem` namespace with:

{% highlight clj %}
(:require [cabinet.elem :as elem])
{% endhighlight %}

Then change the definitions of `handler` to:

{% highlight clj %}
(defroutes handler
  (GET "/elems" []
    (json-response (elem/list)))

  (GET "/elems/:id" [id]
    (json-response (elem/get id)))

  (PUT "/elems/:id" [id attrs]
    (json-response (elem/put id attrs)))

  (DELETE "/elems/:id" [id]
    (json-response (elem/delete id))))
{% endhighlight %}

Now restart the web server and test the new API with `curl`:

{% highlight sh %}
$ curl -X GET http://localhost:8080/elems
{}

$ curl -X PUT -H "Content-Type: application/json" \
    -d '{"attrs": {"tag": "foo"}}' \
    http://localhost:8080/elems/1
{"tag":"foo"}

$ curl -X GET http://localhost:8080/elems/1
{"tag":"foo"}

$ curl -X PUT -H "Content-Type: application/json" \
    -d '{"attrs": {"group": "bar"}}' \
    http://localhost:8080/elems/1
{"tag":"foo","group":"bar"}

$ curl -X PUT -H "Content-Type: application/json" \
    -d '{"attrs": {"tag": "bat", "group": "biz"}}' \
    http://localhost:8080/elems/2
{"tag":"bat","group":"biz"}

$ curl -X DELETE http://localhost:8080/elems/1
{"tag":"foo","group":"bar"}

$ curl -X GET http://localhost:8080/elems
{"2": {"tag":"bat","group":"biz"}}
{% endhighlight %}

This is a great start to our REST API. Before we expose it to external clients, we should ensure that it handles errors gracefully. For example, if the client requests a non-existent element they currently get:

{% highlight sh %}
$ curl -X GET http://localhost:8080/elems/3
null
{% endhighlight %}

Or if the client sends malformed JSON they would see:

{% highlight sh %}
$ curl -i -X PUT -H "Content-Type: application/json" \
    -d 'foobar' \
    http://localhost:8080/elems/1
HTTP/1.1 500 Internal Server Error
Connection: close
Server: Jetty(6.1.14)
{% endhighlight %}

Let's add some error handling code to our service to make it more robust. First, update the `cabinet.elem` namespace:

{% highlight clj %}
(ns cabinet.elem
  (:use clojure.contrib.condition)
  (:refer-clojure :exclude (list get delete)))

(def elems (atom {}))

(defn list []
  @elems)

(defn get [id]
  (or (@elems id)
      (raise :type :not-found
             :message (format "elem '%s' not found" id))))

(defn put [id attrs]
  (if (empty? attrs)
    (raise :type :invalid
           :message "attrs are empty")
    (let [new-attrs (merge (@elems id) attrs)]
      (swap! elems assoc id new-attrs)
      new-attrs)))

(defn delete [id]
  (let [old-attrs (get id)]
    (swap! elems dissoc id)
    old-attrs))
{% endhighlight %}

Now the functions in `cabinet.elem` will raise `:type`d exceptions when an error is encountered, like an expected element being missing.

We also need to update the `web` component of the app. Start by adding these imports to the namespace declaration:

{% highlight clj %}
(:import org.codehaus.jackson.JsonParseException)
(:import clojure.contrib.condition.Condition)
{% endhighlight %}

Then by adding these two definitions:

{% highlight clj %}
(def error-codes
  {:invalid 400
   :not-found 404})

(defn wrap-error-handling [handler]
  (fn [req]
    (try
      (or (handler req)
          (json-response {"error" "resource not found"} 404))
      (catch JsonParseException e
        (json-response {"error" "malformed json"} 400))
      (catch Condition e
        (let [{:keys [type message]} (meta e)]
          (json-response {"error" message} (error-codes type)))))))
{% endhighlight %}

`error-codes` defines HTTP error codes for the two `:type`s of exceptions that can be thrown by `cabinet.elem`. The `wrap-error-handling` middleware function tries to run a given handler function, returning an appropriate response if an error is encountered. In particular it handles cases when the request does not match any of the handler's routes, when JSON parsing fails, or when `cabinet.elem` raises a `:type`d exception.

Enable this error handling by updating the `app` Var to:

{% highlight clj %}
(def app
  (-> handler
    wrap-json-params
    wrap-error-handling))
{% endhighlight %}

and then restarting the server. You can check the improved error handling with `curl`:

{% highlight sh %}
$ curl -X GET -w "%{http_code}\n" http://localhost:8080/files
{"error": "resource not found"}
404

$ curl -X PUT -H "Content-Type: application/json" \
    -d 'foobar' -w "%{http_code}\n" \
    http://localhost:8080/elems/1
{"error":"malformed json"}
400

$ curl -X GET -w "%{http_code}\n" http://localhost:8080/elems/3
{"error": "elem '3' not found"}
404

$ curl -X PUT -H "Content-Type: application/json" \
    -d '{"attrs": {}}' -w "%{http_code}\n" \
    http://localhost:8080/elems/1
{"error": "attrs are empty"}
400
{% endhighlight %}

With this error handling in place, we're ready to expose our service to clients.

I hope you'll try building your own REST APIs in Clojure. Feel free to use [the source of this sample app](http://github.com/mmcgrana/cabinet) as a starting point for your own experiments.
