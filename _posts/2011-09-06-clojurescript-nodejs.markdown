---
layout: post
title: ClojureScript and Node.js
---

# {{page.title}}

<span class="meta">September 6 2011</span>

[ClojureScript](https://github.com/clojure/clojurescript) is a new [Clojure](http://clojure.org) compiler targeting JavaScript. The Clojure contributors [designed](https://github.com/clojure/clojurescript/wiki/Rationale) ClojureScript primarily as a replacement for application-level JavaScript in the client side of web applications. However, ClojureScript can also be deployed outside of the browser by executing the compiled JavaScript with the [V8](http://code.google.com/p/v8/)-based [Node.js](http://nodejs.org).

This approach may be useful for writing Clojure programs that interact closely with the host system, that have small startup times, and that leverage the Node.js runtime and libraries.

In this post we'll demonstrate the mechanics of using ClojureScript with Node.js and see some programs written with this stack. The information here should provide a good starting point for you to explore using ClojureScript and Node.js in your own applications.

The code for the programs described below is available on [Github](https://github.com/mmcgrana/cljs-demo).


### Hello World

We'll start by setting up a local ClojureScript installation according to the [ClojureScript quickstart guide](https://github.com/clojure/clojurescript/wiki/Quick-Start):

{% highlight sh %}
$ git clone git://github.com/clojure/clojurescript.git
$ cd clojurescript
$ script/bootstrap
$ echo "export CLOJURESCRIPT_HOME=$PWD" >> ~/.bash_profile
$ echo "export PATH=\$PATH:$PWD/bin" >> ~/.bash_profile
$ source ~/.bash_profile
{% endhighlight %}

Here we've cloned the ClojureScript source, downloaded its dependencies, and configured our environment to point to the ClojureScript compiler.

Now create a project skeleton for our first hello world ClojureScript program:

{% highlight sh %}
$ mkdir cljs-demo
$ cd cljs-demo
$ mkdir -p src/cljs-demo
{% endhighlight %}

Put this in the first source file `src/cljs-demo/hello.cljs`:

{% highlight clj %}
(ns cljs-demo.hello)

(defn start [& _]
  (println "Hello World!"))

(set! *main-cli-fn* start)
{% endhighlight %}

To compile the program:

{% highlight sh %}
$ mkdir -p out
$ cljsc src/cljs-demo/hello.cljs \
   '{:optimizations :simple :pretty-print true :target :nodejs}' \
   > out/hello.js
{% endhighlight %}

And to run it:

{% highlight sh %}
$ node out/hello.js
Hello World!
{% endhighlight %}

Here we've used the ClojureScript compiler `cljsc` to compile our `hello.cljs` source to the `hello.js` output target. The `hello.js` output file is self-contained; it will include all of the JavaScript code needed to run our program, including the compiled output corresponding to our program and the ClojureScript runtime library.

Our choice of compilation options to `cljsc` here is important. We choose the `:simple` level `:optimization` because that gives us both a single `.js` output file, readable JavaScript code within this file, and readable stack traces on runtime errors. The `:advanced` optimization level would also result in a single `.js` file but with unreadable source code and therefore unreadable stack traces, while no optimizations produce would produce many `.js` output files unsuitable as arguments to the `node` binary. The `:pretty-print` option ensures that the JavaScript is emitted with normalized whitespace. Finally, the `:nodejs` value for the `:target` option instructs the ClojureScript compiler to adjust the the JavaScript output to bind to the Node.js runtime.

Note also that we've explicitly defined our top-level entry point with `(set! *main-cli-fn* <fn>)`. This instructs the Node.js runtime binding mentioned above to invoke that function on startup.


### Node.js Libraries

A primary motivation for targeting Node.js is to give our programs access to Node.js libraries. The Node.js libraries for subprocesses, OS interop, IO, and networking in particular may be useful. Let's see how we can access Node.js libraries from our ClojureScript programs.

First we'll convert the HTTP server on the [Node.js home page](http://nodejs.org) from Node.js to ClojureScript. Put the following code in `src/cljs-demo/http.cljs`:

{% highlight clj %}
(ns cljs-demo.http
  (:require [cljs.nodejs :as node]))

(def http
  (node/require "http"))

(defn handler [_ res]
  (.writeHead res 200 (.strobj {"Content-Type" "text/plain"}))
  (.end res "Hello World!\n"))

(defn start [& _]
  (let [server (.createServer http handler)]
    (.listen server 1337 "127.0.0.1")
    (println "Server running at http://127.0.0.1:1337/")))

(set! *main-cli-fn* start)
{% endhighlight %}

Compile the server program with:

{% highlight sh %}
$ cljsc src/cljs-demo/http.cljs \
    '{:optimizations :simple :pretty-print true :target :nodejs}' \
    > out/http.js
{% endhighlight %}

Then run and test it:

{% highlight sh %}
$ node out/http.js
Server running at http://127.0.0.1:1337/

$ curl -i http://127.0.0.1:1337/
HTTP/1.1 200 OK
Content-Type: text/plain
Connection: keep-alive
Transfer-Encoding: chunked

Hello World!
{% endhighlight %}

There are several things to point out about this example. First, we've `require'd` the `cljs.nodejs` namepsace into our program's namespace. This will give us access to the top-level Node.js function `require`. Indeed, we see that we use `(node/require "http")` to bring in the Node.js HTTP module. Where in Node.js we would write:

{% highlight js %}
var http = require("http");
http.createServer(...);
{% endhighlight %}

In ClojureScript we write:

{% highlight clj %}
(def http (node/require "http"))
(.createServer http ...)
{% endhighlight %}

We also see that ClojureScript uses a Clojure-like dot syntax for host interop. This syntax is used to invoke functions on Node.js modules and on JavaScript objects:

{% highlight clj %}
(.parse (node/require "url") "http://google.com")

(.toFixed 0.9876 3)
{% endhighlight %}

Note that 0-arity functions cannot be called in exactly this way. For example, the following returns the function `end` on the object `response` but does not invoke the function:

{% highlight clj %}
(.end response)
{% endhighlight %}

To actually invoke the function we would use:

{% highlight clj %}
(. response (end))
{% endhighlight %}

In our HTTP server code, note we also coerced the last argument of `writeHead` from a ClojureScript map to a JavaScript object. The Node.js libraries and JavaScript libraries in general expect JavaScript objects for their map-like arguments, so we'll need to explicitly coerce them when making function calls like this. Here we use ClojureScript's `strobj` to access the internal representation of the ClojureScript map, which for all-string maps happens to be what we need. We'll see a more robust method for performing this coercion in the full example program below.


### Complete Example

Let's look at a substantial example program in ClojureScript and Node.js to get a better idea of how this stack works.

Our example program will act as streaming HTTP event processing service. We've chosen this example because it requires a combination of coordinated global state, HTTP clients and servers, JSON transport, concurrent connection handling, request and response streaming, event-orientation, and signal handling. These attributes make it a great fit for ClojureScript and Node.js.

The particular requirements that we'd like our HTTP event processing service to meet are:

* Bind an HTTP server to a specified `PORT`.
* Service clients sending events to the server in a streaming HTTP POST. The events in the stream will represent named values and be encoded as JSON lines of the form `{"name": <string>, "value": <number>}`.
* Service clients fetching summary statistics from the server as a streaming HTTP GET. The statistics in the stream will represent named sums over the corresponding input values, be encoded as JSON lines of the form `{"name": <string>, "sum": <number>}`, and be emitted once per second for every client and known stat name.
* Enable graceful shutdown by trapping `TERM` signals and closing any open clients before exiting.

We'll break the program into a few namespaces. Let's start with the `cljs-demo.util` namespace in the `src/cljs-demo/util.cljs` file. Here is the preamble, which sets us up for some helper functions discussed below:

{% highlight clj %}
(ns cljs-demo.util
  (:require [cljs.nodejs :as node]
            [clojure.string :as string]))

(def url (node/require "url"))
{% endhighlight %}

The following helper function will recursively convert ClojureScript data structures and types to JavaScript ones. This is a more robust alternative to the raw `.strobj` call for coercing arguments to JavaScript library functions:

{% highlight clj %}
(defn clj->js
  "Recursively transforms ClojureScript maps into Javascript objects,
   other ClojureScript colls into JavaScript arrays, and ClojureScript
   keywords into JavaScript strings."
  [x]
  (cond
    (string? x) x
    (keyword? x) (name x)
    (map? x) (.strobj (reduce (fn [m [k v]]
               (assoc m (clj->js k) (clj->js v))) {} x))
    (coll? x) (apply array (map clj->js x))
    :else x))
{% endhighlight %}

Note that this function's approximate inverse `js->clj` is provided by ClojureScript core.

Next a few JSON utilities:

{% highlight clj %}
(defn json-generate
  "Returns a newline-terminate JSON string from the given
   ClojureScript data."
  [data]
  (str (JSON/stringify (clj->js data)) "\n"))

(defn json-parse
  "Returns ClojureScript data for the given JSON string."
  [line]
  (js->clj (JSON/parse line)))
{% endhighlight %}

These functions handle both generation/parsing and coercing to/from JavaScript native types.

A simple URL-parsing helper wrapping the built-in Node.js library:

{% highlight clj %}
(defn url-parse
  "Returns a map with parsed data for the given URL."
  [u]
  (let [raw (js->clj (.parse url u))]
    {:protocol (.substr (get raw "protocol")
                        0 (dec (.length (get raw "protocol"))))
     :host (get raw "hostname")
     :port (js/parseInt (get raw "port"))
     :path (get raw "pathname")}))
{% endhighlight %}

Note that we've used `js/parseInt` to access the global JavaScript function `parseInt`. This `js/` syntax be used to access other globals like `encodeURI` and `Date`.

We'll also include in our utiliites a pair of timer helpers wrapping the standard `setInterval` and `clearInterval` functions:

{% highlight clj %}
(defn set-interval
  "Invoke the given function after and every delay milliseconds."
  [delay f]
  (js/setInterval f delay))

(defn clear-interval
  "Cancel the periodic invokation specified by the given interval id."
  [interval-id]
  (js/clearInterval interval-id))
{% endhighlight %}

A handful of helpers to wrap some Node.js OS interaction:

{% highlight clj %}
(defn env
  "Returns the value of the environment variable k,
   or raises if k is missing from the environment."
  [k]
  (let [e (js->clj (.env node/process))]
    (or (get e k) (throw (str "missing key " k)))))

(defn trap
  "Trap the Unix signal sig with the given function."
  [sig f]
  (.on node/process (str "SIG" sig) f))

(defn exit
  "Exit with the given status."
  [status]
  (.exit node/process status))
{% endhighlight %}

Finally, we'll add a function to help us manage top-level entry points in our project:

{% highlight clj %}
(defn main
  "Set the top-level entry point to the given function."
  [main-name f]
  (let [cl-name (or (get (js->clj (.argv node/process)) 2)
                    (throw "no main name given"))]
  (if (= cl-name main-name)
    (set! *main-cli-fn* #(f (rest %))))))
{% endhighlight %}

This function will allow us to compile a project to a JavaScript file that can be used to invoke a variety of different top-level entry points, in our case `"generator"` and `"processor"`. We need such a helper because multiple direct calls to `set! *main-cli-fn*` within the same project would result in one of these calls clobbering the others.

For this reason we'll also need comment out any manual `set! *main-cli-fn*` calls we already have in the `src` tree, namely those in `hello.cljs` and `http.cljs`:

{% highlight clj %}
; (set! *main-cli-fn* start)
{% endhighlight %}

Now that we have some foundations in place, we can look at the event generator and processor namespaces. We'll start with `cljs-demo.generator` in `src/cljs-demo/generator.cljs`, since this namespace is relatively simple:

{% highlight clj %}
(ns cljs-demo.generator
  (:require [cljs.nodejs :as node]
            [cljs-demo.util :as util]))

(def http (node/require "http"))

(defn log [data]
  (prn (merge {:ns "generator"} data)))

(defn start [& _]
  (log {:fn "start" :event "request"})
  (let [req-opts (-> (util/env "EVENTS_URL")
                   (util/url-parse)
                   (assoc :method "POST"))
        req (.request http (util/clj->js req-opts))]
    (util/set-interval 500 (fn []
      (doseq [name ["tick" "tock" "whiz" "bang"]]
        (log {:fn "start" :event "emit" :name name})
        (let [data {"name" name "value" (rand-int 5)}
              json (util/json-generate data)]
          (.write req json)))))))

(util/main "generator" start)
{% endhighlight %}

The code here is similar to what we saw earlier with `http.cljs`, but there a few additional points to note. First, we've used the `log` function to generate log messages tagged with the `generator` namespace. For the streaming HTTP request itself we extract and parse the given `EVENTS_URL`, open a connection to that endpoint, and then write 4 pieces of JSON data to that connection every 500 milliseconds. Note that we've also used our `util/main` function to indicate that if the project is invoked with the `"generator"` argument then we should start this namespace.

The receiving end of this generator client will be implemented in the `cljs-demo.processor` namespace, which is the heart of the project. At the top of `src/cljs-demo/processor.cljs` we'll have:

{% highlight clj %}
(ns cljs-demo.processor
  (:require [cljs.nodejs :as node]
            [cljs-demo.util :as util]))

(def url  (node/require "url"))
(def http (node/require "http"))

(defn log [data]
  (prn (merge {:ns "processor"} data)))

(defn rand-id []
  (let [chars (apply vector "abcdefghijklmnopqrstuvwxyz0123456789")
         num-chars (count chars)]
     (apply str
       (take 8 (repeatedly #(get chars (rand-int num-chars)))))))
{% endhighlight %}

This snippet includes a typical namespace preambe, a logging helper like the one we saw before, and a helper function `rand-id` for generating random ids for use within the processor.

Next come a few definitions related to the running summary statistics that the processor keeps:

{% highlight clj %}
(def stats-a
  (atom {}))

(defn update-stats [{:strs [name value]}]
  (log {:fn "update-stats" :name name})
  (swap! stats-a update-in [name] #(+ value (or % 0))))
{% endhighlight %}

We'll use an atom since this is a single piece of shared global data, and update it on incoming data using a simple `swap!`.

For global connection tracking we'll use an atom as well:

{% highlight clj %}
(def conns-a
  (atom {}))

(defn add-conn [id conn]
  (log {:fn "add-conn" :conn-id id})
  (swap! conns-a assoc id conn))

(defn remove-conn [id]
  (log {:fn "remove-conn" :conn-id id})
  (swap! conns-a dissoc id))
{% endhighlight %}

Finally, we'll define the core request/response handling functions. This code is all tightly related, so we'll show it here in one listing:

{% highlight clj %}
(defn close-conns []
  (log {:fn "close-conns" :event "start"})
  (doseq [{:keys [id res]} (vals (deref conns-a))]
    (log {:fn "close-conns" :conn-id id :event "close"})
    (.end res ""))
  (log {:fn "close-conns" :event "finish"}))

(defn stream-stats [{:keys [id req res] :as conn}]
  (log {:fn "stream-stats" :conn-id id :event "respond"})
  (.writeHead res 200
    (util/clj->js {"Content-Type" "application/json"}))
  (log {:fn "stream-stats" :conn-id id :event "register"})
  (add-conn id conn)
  (let [interval-id
    (util/set-interval 1000 (fn []
      (log {:fn "stream-stats" :conn-id id :event "tick"})
      (doseq [[name sum] (deref stats-a)]
        (log {:fn "stream-stats" :conn-id id :name name :event "emit"})
        (.write res (util/json-generate {"name" name "sum" sum})))))]
    (.on req "close" (fn []
      (log {:fn "stream-stats" :conn-id id :event "close"})
      (util/clear-interval interval-id)
      (remove-conn id)))))

(defn stream-events [{:keys [id req res] :as conn}]
  (log {:fn "stream-events" :conn-id id :event "register"})
  (add-conn id conn)
  (.on req "data" (fn [line]
    (when line
      (log {:fn "stream-events" :conn-id id :event "data"})
      (let [data (util/json-parse line)]
        (update-stats data)))))
  (.on req "close" (fn []
    (log {:fn "stream-events" :conn-id id :event "close"})
    (remove-conn id))))

(defn not-found [{:keys [id res]}]
  (log {:fn "not-found" :conn-id id :event "respond"})
  (.writeHead res 404
    (util/clj->js {"Content-Type" "application/json"}))
  (.write res (util/json-generate {"error" "not found"}))
  (. res (end)))

(defn parse-request [req]
  {:method (.method req)
   :path   (.pathname (.parse url (.url req)))})

(defn handle-conn [req res]
  (let [conn-id (rand-id)
        {:keys [method path]} (parse-request req)
        conn {:id conn-id :req req :res res}]
    (log {:fn "handle-conn" :conn-id conn-id :method method :path path})
    (condp = [method path]
      ["GET" "/stats"]   (stream-stats conn)
      ["POST" "/events"] (stream-events conn)
      (not-found conn))))

(defn listen [handler port callback]
  (let [server (.createServer http handler)]
    (.on server "clientError" (fn [e]
      (log {:fn "listen" :event "error" :message (. e (toString))})))
    (.listen server port "0.0.0.0" #(callback server))))

(defn stop [server]
  (log {:fn "stop" :event "close-server"})
  (.close server)
  (log {:fn "stop" :event "close-conns"})
  (close-conns)
  (log {:fn "stop" :event "exit" :status 0})
  (util/exit 0))

(defn start [& _]
  (let [port (js/parseInt (util/env "PORT"))]
    (log {:fn "start" :event "listen" :port port})
    (listen handle-conn port (fn [server]
      (log {:fn "start" :event "listening"})
      (doseq [signal ["TERM" "INT"]]
        (util/trap signal (fn []
          (log {:fn "start" :event "catch" :signal signal})
          (stop server)))
        (log {:fn "start" :event "trapping" :signal signal}))))))

(util/main "processor" start)
{% endhighlight %}

To explain this code we'll start from the bottom up and trace the call graph of the running server.

The `util/main` call indicates that `start` should be called at runtime. The `start` function extracts the `PORT`, binds an HTTP server to that port via the `listen` function, and registers shutdown handlers on the `TERM` and `INT` signals. If one of these signals is sent to the running processor, it will initiate graceful shutdown via the `stop` function.

The core HTTP request processing logic starts in `handle-conn`. Here we parse the incoming request and branch based on its request method and path. If the request is to `GET /stats` we serve the streaming summary stats to the client, if it's to `POST /events` we accept streaming events from the client, and otherwise we return a 404 response. In the `handle-conn` function we also generate a `conn-id` that will be used to track the connection state internally through the connection's lifetime.

The `stream-events` function handles clients that are streaming data into the processor. Here we register the connection so that it can be handled in the case of a shutdown, and accept data events from the client, passing the corresponding parsed information to `update-stats` for processing.

`stream-stats` works in a similar way, though writes to the clients are driven by a periodic timer that is setup and torn down at the beginning and end of the client request, respectively. The function triggered by this timer `deref`s the accumulated stats and then sends that data to the client.

Lastly, the `close-conns` connection is available for invocation at shutdown. Here we will iterate over any open connections and explicitly close them before the server exits.

With this we finally have all the code that we need to test the service end-to-end. First we'll need to compile it:

{% highlight sh %}
$ cljsc src \
    '{:optimizations :simple :pretty-print true :target :nodejs}' \
    > out/cljs-demo.js
{% endhighlight %}

We've given the compiler a `src` directory argument this time to indicate that it should compile all of the source files that now compose the project.

Once the program is compiled, start the event processor:

{% highlight clj %}
$ PORT=5000 node out/cljs-demo.js processor
{:ns "processor", :fn "start", :event "listen", :port 5000}
{:ns "processor", :fn "start", :event "listening"}
{:ns "processor", :fn "start", :event "trapping", :signal "TERM"}
{:ns "processor", :fn "start", :event "trapping", :signal "INT"}
{% endhighlight %}

Then start a few event generators to send data to the processor:

{% highlight clj %}
$ EVENTS_URL=http://127.0.0.1:5000/events node out/cljs-demo.js generator
{:ns "generator", :fn "start", :event "request"}
{:ns "generator", :fn "start", :event "emit", :name "tick"}
{:ns "generator", :fn "start", :event "emit", :name "tock"}
{:ns "generator", :fn "start", :event "emit", :name "whiz"}
{:ns "generator", :fn "start", :event "emit", :name "bang"}
{% endhighlight %}

Finally, observe the streaming sums with `curl`. On these requests you should see 4 lines of JSON emitted every second, with the indicated sums growing at a rate proportional to the number of connected generator clients:

{% highlight sh %}
$ curl -i http://127.0.0.1:5000/stats
HTTP/1.1 200 OK
Content-Type: application/json
Connection: keep-alive
Transfer-Encoding: chunked

{"name": "tick", "sum": 3}
{"name": "whiz", "sum": 5}
{"name": "bang", "sum": 4}
{"name": "tock", "sum": 2}
{"name": "tick", "sum": 5}
{"name": "whiz", "sum": 9}
{"name": "bang", "sum": 7}
{"name": "tock", "sum": 8}
{% endhighlight %}

If you disconnect a client by hitting Control-C in its terminal, you should see that the processor gracefully handles updating its internal state and continues to serve the other clients:

{% highlight clj %}
{:ns "processor", :fn "stream-events", :conn-id "rx4pymm8", :event "close"}
{:ns "processor", :fn "remove-conn", :conn-id "rx4pymm8"}
{% endhighlight %}

The server will also gracefully close all connections upon receiving a termination signal, which again you can send with Control-C:

{% highlight clj %}
{:ns "processor", :fn "start", :event "catch", :signal "INT"}
{:ns "processor", :fn "stop", :event "close-server"}
{:ns "processor", :fn "stop", :event "close-conns"}
{:ns "processor", :fn "close-conns", :event "start"}
{:ns "processor", :fn "close-conns", :conn-id "3jd4faxu", :event "close"}
{:ns "processor", :fn "close-conns", :conn-id "5wrhtc8i", :event "close"}
{:ns "processor", :fn "close-conns", :event "finish"}
{:ns "processor", :fn "stop", :event "exit", :status 0}
{% endhighlight %}

The example program that we've seen here is simple, but it does demonstrate some of the compelling features that Node.js can bring to ClojureScript.


### Faster Compilation

So far we've been using the `cljsc` binary to compile our ClojureScript programs for execution with Node.js. This is straightforward but slow. To speed up the development cycle you may want to use a different compilation approach.

One option is to open a Clojure REPL and call the ClojureScript compilation library function directly. This will substantially cut compilation time by eliminating the JVM boot on each compile. For example, to compile our original hello world program:

{% highlight clj %}
$ cd cljs-demo
$ ~/code/clojurescript/script/repl
Clojure 1.3.0-beta1
user=> (require '[cljs.closure :as cljsc])
user=> (cljsc/build "src/cljs-demo/hello.cljs"
  {:optimizations :simple
   :pretty-print true
   :target :nodejs
   :output-to "out/hello.js"})
{% endhighlight %}

Indeed, the `cljsc` binary is simply a small wrapper around this `cljsc/build` function, and the function arguments to the latter are the same as the command-line arguments to the former.

An even faster option for compilation is the [`cljs-watch`](https://github.com/ibdknox/cljs-watch) tool, which will watch your ClojureScript source directory and automatically recompile when anything changes. To use this tool, first install up `cljs-watch` by putting the binary on your `PATH`:

{% highlight sh %}
$ git clone https://github.com/ibdknox/cljs-watch.git
$ cp cljs-watch/cljs-watch /usr/local/bin
{% endhighlight %}

Again to compile our original hello world program:

{% highlight sh %}
$ cd cljs-demo
$ cljs-watch src/cljs-demo/hello.cljs \
    '{:optimizations :simple :pretty-print true :target :nodejs
      :output-to "out/hello.js"}'
09:30:02 :: watcher :: Building ClojureScript files in...
09:30:07 :: watcher :: Waiting for changes
09:30:31 :: watcher :: Compiling updated files...     [done]
{% endhighlight %}

These compilation approaches will significantly shorten your development cycle and make your ClojureScript programming more fun and productive.


### Looking Forward

ClojureScript and Node.js are both young, but they are already showing significant promise. Their combination may prove valuable as JavaScript becomes an increasingly important platform, Node.js a more powerful and widely-used runtime, and ClojureScript a more mature language.
