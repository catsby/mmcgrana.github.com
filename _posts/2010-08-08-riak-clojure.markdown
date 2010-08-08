---
layout: post
title: Exploring Riak with Clojure
---

# {{page.title}}

<span class="meta">August 8 2010</span>

[Riak](http://www.basho.com/Riak.html) is an open source distributed datastore optimized for internet-scale operations. Riak provides horizontally scalable key-value storage and a distributed JavaScript map-reduce query layer. These features make Riak an interesting candidate for backing large-scale applications. The first step on the road to such deployments is experimenting with Riak on your development machine. In this post I'll walk through the basics of accessing a local Riak cluster with Clojure.

This post loosely follows the [Riak Fast Track](https://wiki.basho.com/display/RIAK/The+Riak+Fast+Track) tutorial provided by [Basho](http://www.basho.com/), the company behind Riak. I'd suggest reading through the Fast Track as you work through the corresponding code examples in this post.

We'll start by installing Riak and starting a local cluster. First, ensure that you have Erlang (Riak is written in Erlang). On a Mac:

{% highlight sh %}
brew install erlang
{% endhighlight %}

For other systems, see this [wiki page](https://wiki.basho.com/display/RIAK/Installing+Erlang).

Now build the most recent version of Riak:

{% highlight sh %}
git clone git://github.com/basho/riak.git
cd riak
make all devrel
{% endhighlight %}

This will build a set of 3 standalone Riak releases that you can use to start your local cluster:

{% highlight sh %}
cd dev
dev1/bin/riak start
dev2/bin/riak start
dev3/bin/riak start
dev2/bin/riak-admin join dev1@127.0.0.1
dev3/bin/riak-admin join dev1@127.0.0.1
{% endhighlight %}

Here `riak start` boots an independent Riak node while `riak-admin join` brings a node into a cluster. After executing these commands, you should have a 3 node Riak cluster running. Sanity check your setup with:

{% highlight sh %}
curl -s -H "Accept: text/plain" http://127.0.0.1:8091/stats | grep own
{% endhighlight %}

You should see something like:

{% highlight sh %}
"[{'dev3@127.0.0.1',21},{'dev2@127.0.0.1',21},{'dev1@127.0.0.1',22}]"
{% endhighlight %}

This indicates the three nodes have successfully joined into a Riak cluster.

Let's try accessing this cluster from Clojure. We'll use the [`clj-riak`](http://github.com/mmcgrana/clj-riak) library, a wrapper for the [`riak-java-pb-client`](http://github.com/krestenkrab/riak-java-pb-client) Java library for the [Riak Protocol Buffers API](https://wiki.basho.com/display/RIAK/PBC+API). Start by `clone`ing into the repository, pulling the dependencies, and starting a REPL:

{% highlight sh %}
git clone git://github.com/mmcgrana/clj-riak.git
cd clj-riak
lein deps
lein repl
{% endhighlight %}

Now create a client connected to one of the Riak nodes in the cluster:

{% highlight clj %}
(require '[clj-riak.client :as client])

(def rc (client/init {:host "127.0.0.1" :port 8081}))
{% endhighlight %}

Here `rc` stands for "Riak client". Make sure the client is connected by issuing a `ping`:

{% highlight clj %}
(client/ping rc)
=> true
{% endhighlight %}

You can get some information about the server the client is connected to with:

{% highlight clj %}
(client/get-server-info rc)
=> {:node "dev1@127.0.0.1"
    :server-version "0.12.0"}
{% endhighlight %}

Let's look at the key-value functionality provided by Riak and the `clj-riak` client. Start by `put`ing a simple text object to the bucket `"docs"` with the key `"doc1"`:

{% highlight clj %}
(client/put rc "docs" "doc1"
  {:value (.getBytes "the content")
   :content-type "text/plain"})
{% endhighlight %}

Now `get` the data back:

{% highlight clj %}
(client/get rc "docs" "doc1")
=> {:value #<byte[] [B@4d33b92c>
    :content-type "text/plain"
    :vtag "7b5JVYfs7nr4YZsZjM5eTG"
    :vclock #<ByteString com.google.protobuf.ByteString@69a4447a>
    :charset ""
    :content-encoding ""
    :last-mod-usecs 669730
    :last-mod 1281290355
    :user-meta {}
    :links ()}
{% endhighlight %}

To deserialize the returned `byte` `:value`:

{% highlight clj %}
(String. (:value (client/get rc "docs" "doc1")))
=> "the content"
{% endhighlight %}

After we `delete` the object, subsequent `get` requests will return `nil`:

{% highlight clj %}
(client/delete rc "docs" "doc1")

(client/get rc "docs" "docs1")
=> nil
{% endhighlight %}

We mentioned that documents are stored within buckets. These buckets can be individually configured. Use `get-bucket-props` to see a bucket's configuration:

{% highlight clj %}
(client/get-bucket-props rc "docs")
=> {:n-value 3
    :allow-mult false}
{% endhighlight %}

Then use `set-bucket-props` to change that configuration:

{% highlight clj %}
(client/set-bucket-props rc "docs"
  {:n-value 2
   :allow-mult true})

(client/get-bucket-props rc "docs")
=> {:n-value 2
    :allow-mult true}
{% endhighlight %}
  
The last Riak feature we'll explore is map-reduce querying. Following the Riak Fast Track, we'll use some sample Google stock price data. Download this data in preparation for loading it into Riak:

{% highlight sh %}
curl -L -o goog.csv http://bit.ly/dtD3JC
{% endhighlight %}

Include some additional Clojure libraries that we need below:

{% highlight clj %}
(use '[clojure.contrib.io :only (read-lines)])
(use '[clojure.contrib.json :only (json-str)])
(use '[clojure.contrib.string :only (split)])
{% endhighlight %}

The Google stock price data is in simple CSV format:

{% highlight clj %}
(take 2 (read-lines "goog.csv"))
=> ("Date,Open,High,Low,Close,Volume,Adj Close"    
    "2010-05-05,500.98,515.72,500.47,509.76,4566900,509.76")
{% endhighlight %}

Use the following code to load all of this data into the Riak cluster as JSON documents within the `"goog"` bucket and keyed by the date of the record:

{% highlight clj %}
(doseq [line (rest (read-lines "goog.csv"))]
  (let [[d o h l c v a] (split #"," line)]
    (client/put rc "goog" d
      {:value (.getBytes (json-str
         {"date" d "open" o "high" h "low" l
          "close" c "volume" v "adjclose" a}))
       :content-type "application/json"})))
=> nil
{% endhighlight %}

As before, we can `get` records according to their bucket and key:

{% highlight clj %}
(String. (:value (client/get rc "goog" "2010-05-05")))
=> "{\"date\":\"2010-05-05\",
     \"open\":\"500.98\",
     \"high\":\"515.72\",
     \"low\":\"500.47\",
     \"close\":\"509.76\",
     \"volume\":\"4566900\",
     \"adjclose\":\"509.76\"}"
{% endhighlight %}

The first map-reduce query we'll try will simply collect all of the records in the `"goog"` bucket using the built in `"Riak.mapValuesJson"` function. I'll prefix the `map-reduce` call with a `(take 2` to prevent all of that data from being dumped to the REPL:

{% highlight clj %}
(take 2 (client/map-reduce rc
          {"inputs" "goog"
           "query" [{"map" {"language" "javascript"
                            "name" "Riak.mapValuesJson"
                            "keep" true}}]}))
=> ("[{\"date\":\"2010-03-29\",\"open\":\"563.00\",
       \"high\":\"564.72\",\"low\":\"560.57\",\"close\":\"562.45\",
       \"volume\":\"3104500\",\"adjclose\":\"562.45\"}]"
    "[{\"date\":\"2010-02-12\",\"open\":\"532.97\",
       \"high\":\"537.15\",\"low\":\"530.50\",\"close\":\"533.12\",
       \"volume\":\"2279700\",\"adjclose\":\"533.12\"}])
{% endhighlight %}

For the next example, we'll give specific bucket/key pairs as inputs to the map-reduce request:

{% highlight clj %}
(client/map-reduce rc
  {"inputs" [["goog" "2010-01-04"]
             ["goog" "2010-01-05"]
             ["goog" "2010-01-06"]
             ["goog" "2010-01-07"]
             ["goog" "2010-01-08"]]
   "query" [{"map" {"language" "javascript"
             "name" "Riak.mapValuesJson"
             "keep" true}}]})
=> ("[{\"date\":\"2010-01-05\",\"open\":\"627.18\"...}]" ...)
{% endhighlight %}

That's nice to see, but it's really an elaborate multi-get. We see more potential in the map-reduce functionality when using custom JavaScript in the query:

{% highlight clj %}
(client/map-reduce rc
  {"inputs" [["goog" "2010-01-04"]
             ["goog" "2010-01-05"]
             ["goog" "2010-01-06"]
             ["goog" "2010-01-07"]
             ["goog" "2010-01-08"]]
   "query" [
     {"map" {"language" "javascript"
             "source" "function(value, keyData, arg) {
                         var data = Riak.mapValuesJson(value)[0];
                         return [data.high]; }"}}
     {"reduce" {"language" "javascript"
                "name" "Riak.reduceMax"
                "keep" true}}]})
=> ("[\"629.51\"]")
{% endhighlight %}

This requests uses a custom `map` function and the built-in `Riak.reduceMax` reduce function to find the maximum high price across the five given days. 

More elaborate queries are possible. For example, the Fast Track provides an example query that can be used to find the maximum high prices for all months represented in the dataset. To avoid having to paste the entire query into the REPL, we'll download it to a local file first and then load it from there:

{% highlight sh %}
curl -L -o max-highs-by-month.json http://bit.ly/bpwmHK
{% endhighlight %}

Now use this JSON to fire off the actual query:

{% highlight clj %}
(client/map-reduce rc (slurp "max-highs-by-month.json"))
=> ("[{\"2010-02\":\"547.50\",
       \"2010-03\":\"588.28\",
       \"2009-07\":\"452.70\", ...}]")
{% endhighlight %}

This brief introduction leaves many aspects of Riak unaddressed. For example, we have not looked at throughput, scalability, fault tolerance, conflict resolution, or production operations -- all critical to a complete understanding of the datastore. I nonetheless hope that this gets you started with exploring Riak and that you continue by perusing the various resources available on the [Riak wiki](https://wiki.basho.com/display/RIAK/Riak).







