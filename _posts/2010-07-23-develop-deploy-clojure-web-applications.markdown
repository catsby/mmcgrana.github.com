---
layout: post
title: Developing and Deploying a Simple Clojure Web Application
---

# {{page.title}}

<span class="meta">July 23 2010</span>

The post walks through the process of developing and deploying a simple web application in Clojure. After reading this you should be able to build your own app and deploy it to a production server.

Our sample app performs addition for the user. The user enters a value in each of two text fields, the values are submitted to the app, and the app returns the corresponding sum. Eventually it will look like this:

<img src="/images/2010-07-23-eventual1.png">

<img src="/images/2010-07-23-eventual2.png">

Before getting started on the app, make sure that you have [Leiningen](http://github.com/technomancy/leiningen) installed and that you have a [clj](/2010/03/clojure-setup.html) script available.

We'll start with a minimal first version of the app. In a new directory `adder`, create a file `project.clj` with the following contents:

{% highlight clj %}
(defproject adder "0.0.1"
  :description "Add two numbers."
  :dependencies
    [[org.clojure/clojure "1.2.0-beta1"]
     [org.clojure/clojure-contrib "1.2.0-beta1"]
     [ring/ring-core "0.2.5"]
     [ring/ring-devel "0.2.5"]
     [ring/ring-jetty-adapter "0.2.5"]
     [compojure "0.4.0"]
     [hiccup "0.2.6"]])
{% endhighlight %}

We'll put the main app logic in the namespace `adder.core`. Create a file at `src/adder/core.clj` with this code:

{% highlight clj %}
(ns adder.core
  (:use compojure.core)
  (:use hiccup.core)
  (:use hiccup.page-helpers))

(defn view-layout [& content]
  (html
    (doctype :xhtml-strict)
    (xhtml-tag "en"
      [:head
        [:meta {:http-equiv "Content-type"
                :content "text/html; charset=utf-8"}]
        [:title "adder"]]
      [:body content])))

(defn view-input []
  (view-layout
    [:h2 "add two numbers"]
    [:form {:method "post" :action "/"}
      [:input.math {:type "text" :name "a" :value a}] [:span.math " + "]
      [:input.math {:type "text" :name "b" :value b}] [:br]
      [:input.action {:type "submit" :value "add"}]]))

(defn view-output [a b sum]
  (view-layout
    [:h2 "two numbers added"]
    [:p.math a " + " b " = " sum]
    [:a.action {:href "/"} "add more numbers"]))

(defn parse-input [a b]
  [(Integer/parseInt a) (Integer/parseInt b)])

(defroutes app
  (GET "/" []
    (view-input))

  (POST "/" [a b]
    (let [[a b] (parse-input a b)
          sum   (+ a b)]
      (view-output a b sum))))
{% endhighlight %}

Also, put the following in `script/run.clj`:

{% highlight clj %}
(use 'ring.adapter.jetty)
(require 'adder.core)

(run-jetty #'adder.core/app {:port 8080})
{% endhighlight %}

Now you're ready to test the first version of the app:

{% highlight clj %}
lein deps
clj script/run.clj
open http://localhost:8080/
{% endhighlight %}

Check out your app in the browser. You should be to perform the simple addition described above.

As you use the app you'll probably notice changes that you'd like to make. You might also notice that errors like giving `foo` as an input are not handled well. To fix this let's apply some reloading and stacktrace middleware.

Start by including the appropriate Ring middlewares into the `adder.core` namespace definition:

{% highlight clj %}
(:use ring.middleware.reload)
(:use ring.middleware.stacktrace)
{% endhighlight %}

We'll want to separate out the main app logic that we wrote earlier from the full, middleware wrapped application, so change `(defroutes app` to `(defroutes handler` and add the following at the bottom of the file:

{% highlight clj %}
(def app
  (-> #'handler
    (wrap-reload '[adder.core])
    (wrap-stacktrace)))
{% endhighlight %}


After stopping your running server and restarting it with `clj script/run.clj`, you should be able to see changes to your code in `adder.core` reflected immediately in the web interface. Also, if your application encounters any errors you will see a stacktrace indicating what went wrong:

<img src="/images/2010-07-23-stacktrace.png">

Speaking of errors, we may want to address some of those in our application. If a user enters something other than a number into one of the fields, we should respond with a useful error message. Update the `view-input` function to:

{% highlight clj %}
(defn view-input [& [a b]]
  (view-layout
    [:h2 "add two numbers"]
    [:form {:method "post" :action "/"}
      (if (and a b)
        [:p "those are not both numbers!"])
      [:input.math {:type "text" :name "a" :value a}] [:span.math " + "]
      [:input.math {:type "text" :name "b" :value b}] [:br]
      [:input.action {:type "submit" :value "add"}]]))
{% endhighlight %}

and update the `POST` route to:

{% highlight clj %}
(POST "/" [a b]
  (try
    (let [[a b] (parse-input a b)
          sum   (+ a b)]
      (view-output a b sum))
    (catch NumberFormatException e
      (view-input a b))))
{% endhighlight %}

You can immediately verify that your changes worked by trying some invalid input:

<img src="/images/2010-07-23-invalid.png">

We should also handle the case where the user enters an unrecognized URL. To do that, require the Ring response middleware with:

{% highlight clj %}
(:use ring.util.response)
{% endhighlight %}

and then add a catchall route to the bottom of the routes list:

{% highlight clj %}
(ANY "/*" [path]
  (redirect "/"))
{% endhighlight %}

Now when you visit e.g. `/foo`, you should be redirected back to the app's main page at `/`.

Our app is starting to shape up, but we're missing some necessary application infrastructure. For one, the application is not doing any logging, which makes it hard to understand what it is doing. Lets fix that with some request logging middleware. Create a new file `src/adder/middleware.clj` with these contents:

{% highlight clj %}
(ns adder.middleware)

(defn- log [msg & vals]
  (let [line (apply format msg vals)]
    (locking System/out (println line))))

(defn wrap-request-logging [handler]
  (fn [{:keys [request-method uri] :as req}]
    (let [start  (System/currentTimeMillis)
          resp   (handler req)
          finish (System/currentTimeMillis)
          total  (- finish start)]
      (log "request %s %s (%dms)" request-method uri total)
      resp)))
{% endhighlight %}

Then pull this middleware into `core` with:

{% highlight clj %}
(:use adder.middleware)
{% endhighlight %}

and add it to the app by updating the middleware stack to look like:

{% highlight clj %}
(def app
  (-> #'handler
    (wrap-request-logging)
    (wrap-reload '[adder.middleware adder.core])
    (wrap-stacktrace)))
{% endhighlight %}

Now each request will be noted in the app's logs, along with the time it takes. 

As soon as you try out the logging you'll probably notice requests to `/favicon.ico`. Since our simple app doesn't have a favicon, let's let the browser know with a `404` response. Add a `wrap-bounce-favicon` function to the `adder.middleware` namespace:

{% highlight clj %}
(defn wrap-bounce-favicon [handler]
  (fn [req]
    (if (= [:get "/favicon.ico"] [(:request-method req) (:uri req)])
      {:status 404
       :headers {}
       :body ""}
      (handler req))))
{% endhighlight %}

and then include it in the middleware stack by adding `(wrap-bounce-favicon)` immediately above `(wrap-stacktrace)`.

Now let's add a bit of styling to our utilitarian app. To do this we'll create and apply a CSS file that is served statically by the application. Put the following in public/adder.css:

{% highlight css %}
.math {
  font-family: Monaco, monospace; }

.action {
  margin-top: 2em; }
{% endhighlight %}

and update the `:head` markup in `view-layout` to look like:

{% highlight clj %}
[:head
  [:meta {:http-equiv "Content-type"
          :content "text/html; charset=utf-8"}]
  [:title "adder"]
  [:link {:href "/adder.css" :rel "stylesheet" :type "text/css"}]]
{% endhighlight %}

Next, include the necessary Ring middleware:

{% highlight clj %}
(:use ring.middleware.file)
(:use ring.middleware.file-info)
{% endhighlight %}

and update the middleware stack to look like:

{% highlight clj %}
(def app
  (-> #'handler
    (wrap-file "public")
    (wrap-file-info)
    (wrap-request-logging)
    (wrap-reload '[adder.middleware adder.core])
    (wrap-bounce-favicon)
    (wrap-stacktrace)))
{% endhighlight %}

We should also write a few tests for our newly developed application. Create a file at `test/adder/core_test.clj` with the following contents:

{% highlight clj %}
(ns adder.core-test
  (:use clojure.test)
  (:use adder.core))

(deftest parse-input-valid
  (is (= [1 2] (parse-input "1" "2"))))

(deftest parse-input-invalid
  (is (thrown? NumberFormatException
    (parse-input "foo" "bar"))))

(deftest view-output-valid
  (let [html (view-output 1 2 3)]
    (is (re-find #"two numbers added" html))))

(deftest handle-input-valid
  (let [resp (handler {:uri "/" :request-method :get})]
    (is (= 200 (:status resp)))
    (is (re-find #"add two numbers" (:body resp)))))

(deftest handle-add-valid
  (let [resp (handler {:uri "/" :request-method :post
                       :params {"a" "1" "b" "2"}})]
    (is (= 200 (:status resp)))
    (is (re-find #"1 \+ 2 = 3" (:body resp)))))

(deftest handle-add-invalid
  (let [resp (handler {:uri "/" :request-method :post
                       :params {"a" "foo" "b" "bar"}})]
    (is (= 200 (:status resp)))
    (is (re-find #"those are not both numbers" (:body resp)))))

(deftest handle-catchall
  (let [resp (handler {:uri "/foo" :request-method :get})]
    (is (= 302 (:status resp)))
    (is (= "/" (get-in resp [:headers "Location"])))))
{% endhighlight %}

You can verify that they all pass by running `lein test`.

Now that we have some tests we're ready to start thinking about deploying this app to production. We'll want the app to behave slightly differently in production and development, so we'll need a way to differentiate between the two environments. I'll use the environment variable `APP_ENV` to define a `production?` var in the `adder.core` namespace:

{% highlight clj %}
(def production?
  (= "production" (get (System/getenv) "APP_ENV")))
{% endhighlight %}

Use this var to update the middleware stack to look like:

{% highlight clj %}
(def app
  (-> #'handler
    (wrap-file "public")
    (wrap-file-info)
    (wrap-request-logging)
    (wrap-reload '[adder.middleware adder.core])
    (wrap-bounce-favicon)
    (wrap-exception-logging)
    (wrap-if production?       wrap-failsafe)
    (wrap-if (not production?) wrap-stacktrace)))
{% endhighlight %}

This code will enable a public-facing failsafe middleware in production while keeping the stacktrace middleware in development. We'll also add exception logging in both cases for additional visibility. This updated stack relies several new functions in `adder.middleware`. Add the following to the `adder.middleware` namespace declaration:

{% highlight clj %}
(:require [clj-stacktrace.repl :as strp])
{% endhighlight %}

and to the `adder.middleware` body:

{% highlight clj %}
(defn wrap-if [handler pred wrapper & args]
  (if pred
    (apply wrapper handler args)
    handler))

(defn wrap-exception-logging [handler]
  (fn [req]
    (try
      (handler req)
      (catch Exception e
        (log "Exception:\n%s" (strp/pst-str e))
        (throw e)))))

(defn wrap-failsafe [handler]
  (fn [req]
    (try
      (handler req)
      (catch Exception e
        {:status 500
         :headers {"Content-Type" "text/plain"}
         :body "We're sorry, something went wrong."}))))
{% endhighlight %}

The site will not run on port `8080` in production, so we'll need a way to specify the port to the run script. We'll use the `PORT` environment variable. Update the body of `script/run.clj` to the following:

{% highlight clj %}
(let [port (Integer/parseInt (get (System/getenv) "PORT" "8080"))]
  (run-jetty #'adder.core/app {:port port}))
{% endhighlight %}

Now we're ready to put this app into production. I'll walk through the steps needed for to deploying to EC2 using the standard [EC2 command line tools](http://developer.amazonwebservices.com/connect/entry.jspa?externalID=351), but the process would be similar for other hosting providers.

Start be allocating by setting up a security group and SSH keypair for the application:

{% highlight sh %}
ec2-add-group adder -d "adder deployment"
ec2-authorize adder -P tcp -p 22
ec2-authorize adder -P tcp -p 80

mkdir -p dev
ec2-add-keypair adder | tail -n +2 > dev/adder.pem
chmod 600 dev/adder.pem
{% endhighlight %}

Then allocate a server based on a public Ubunut AMI and wait for it to come up:

{% highlight sh %}
ec2-run-instances ami-2d4aa444 -g adder -k adder \
  -n 1 -t m1.small -z us-east-1a
watch ec2-describe-instances
{% endhighlight %}

Set some local environment variables to make subsequent commands easier:

{% highlight sh %}
export ADDER_PEM=dev/adder.pem
export ADDER_HOST=<ec2-public-ip>
{% endhighlight %}

To set up the server, SSH in

{% highlight sh %}
ssh -i $ADDER_PEM ubuntu@$ADDER_HOST
{% endhighlight %}

and run a few commands to install Java and set up the directory structure:
 
{% highlight sh %}
sudo su root
curl -L -o install-java.sh http://bit.ly/b5lesP
bash install-java.sh
mkdir -p /var/log/adder /var/adder
chown -R ubuntu:ubuntu /var/adder
{% endhighlight %}

We'll control the server process using Ubuntu's [upstart](http://upstart.ubuntu.com/). Put the following upstart configuration file in `deploy/adder.conf`:

{% highlight sh %}
script
  export PORT=80
  export APP_ENV=production
  cd /var/adder
  java -cp "lib/*:src/" clojure.main script/run.clj \
    >> /var/log/adder/adder.log 2>&1
end script
{% endhighlight %}

and then place it in the appropriate spot on the server with:

{% highlight sh %}
scp -i $ADDER_PEM deploy/adder.conf \
  ubuntu@$ADDER_HOST:/tmp/adder.conf
ssh -i $ADDER_PEM ubuntu@$ADDER_HOST \
  "sudo mv /tmp/adder.conf /etc/init/adder.conf"
{% endhighlight %}

Create a list in `deploy/exclude.txt` of files that should not be deployed to the production server:

{% highlight sh %}
.git
.gitignore
deploy
test
classes
{% endhighlight %}

Now install the app's files on the server with:

{% highlight sh %}
rsync --rsh='ssh -i '$ADDER_PEM \
      -vr --delete --exclude-from deploy/exclude.txt \
      ./ ubuntu@$ADDER_HOST:/var/adder/
{% endhighlight %}

After the first `rsync` completes, start the server with:

{% highlight sh %}
ssh -i $ADDER_PEM ubuntu@$ADDER_HOST "sudo start adder"
{% endhighlight %}

and check that it works by opening the production site from your local machine:

{% highlight sh %}
open http://$ADDER_HOST/
{% endhighlight %}

<img src="/images/2010-07-23-public.png">

If you want to deploy a change, `rsync` up your code and then run:

{% highlight sh %}
ssh -i $ADDER_PEM ubuntu@$ADDER_HOST "sudo restart adder"
{% endhighlight %}

I hope this post helps you develop and deploy your own Clojure web applications. If you have any questions about this post or about Clojure web development in general, feel feel to leave them in the comments. I'm also interested in hearing how others have approached the end-to-end Clojure web development and deployment process; please let me know what you think in in the comments as well.

The source code for this app is available on [GitHub](http://github.com/mmcgrana/adder).
