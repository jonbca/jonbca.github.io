---
layout: post
title: "Testing functions without mocking in Clojure"
date: 2015-02-28
categories: clojure
tags:
 - clojure
 - testing
 - lambda-days
---
While attending [Lambda Days][lambda-days] in Kraków, I learned a lot
about functional programming, and a *lot* more about F# (a.k.a.
_F-hash_) than I ever wanted to know. But one of the things that
intrigued me the most was about a topic that I had been struggling with:
How do you mock in functional languages? Fortunately, [Lars
Hupel][lars-hupel-twitter] (@larsrh) gave an [excellent talk][lars-hupel-talk] on functional mocking
using Haskell. It opened my eyes to some of the possibilities of
functional programming that I'd never explored. In particular, he showed
how monads in Haskell can make testing easier. I wondered about how applicable some
of these principles might be in Clojure.

## Mocking with Midje
My current project uses [Midje][midje] which, among other
things, offers a way of "mocking" functions for testing purposes, via
a `provided` function. Here's an example of some ring middleware that does request
logging (namespace and requires declarations omitted):

{% highlight clojure %}
(defn log-request
  [handler]
    (fn [request]
        (log/info (str "request-uri " (:uri request)))
        (log/info (str "user-agent " (-> request :headers (get "user-agent"))))
        (handler request)))
{% endhighlight %}

And here's a test that sort-of checks the logging occurred:

{% highlight clojure %}
(fact "It logs something to the log"
   (let [response {}
         handler (log-request (constantly response))]
     (handler {:uri "/"}) => response
     (provided (log/info "request-uri /") => anything
               (log/info "user-agent ")   => anything)))
{% endhighlight %}

The `provided` functions (called "prerequisites" in Midje) act as traditional mocks,
which produce a given output in response to an input, but also ensure that they have been
called with the specified arguments.

Although this test passes, it smells funny. Let's set aside the fact
that we're mocking functions we don't control (`log/info` comes from
Clojure's logging library). Midje prerequisites just don't feel functional to
me, and what's more, *this test doesn't even test what I said it does*.

In the `fact`, I said that I was checking "it logs something to the
log." But my test here only checks that `log/info` was called twice with
those specific arguments. Not good.

## Mocking with Higher Order Functions
Based on Lars's talk, I wondered how I might fix my test so that it's:

1. Actually testing what I intended to test,
2. More readable, and
3. Less coupled to the implementation.

First, I'll change the test to be what I want it to be. We'll need to
introduce some helper things to replace `provided`:

{% highlight clojure linenos %}
(fact "request information gets logged"
   (let [log (atom [])
         log-fn (fn [& args] (swap! log conj args))
         response {}
         handler ((log-request-provider log-fn) (constantly response))]
     (handler {:uri "/"})
     (count @log) => 2
     (first @log) => ["request-uri /"]
     (second @log) => ["user-agent "])))
{% endhighlight %}

I also need to introduce a new higher-order function in my production code,
and slightly alter the existing `log-request` function:

{% highlight clojure %}
(defn- log-request
  [log-fn handler]
  (fn [request]
    (do
      (log-fn (str "request-uri " (:uri request)))
      (log-fn (str "user-agent " (get-in request [:headers "user-agent"])))
      (handler request))))

(defn log-request-provider
  [log-fn]
  (partial log-request log-fn))
{% endhighlight %}

In this case, `partial` is sufficient for my needs but there may be
other cases where a more specialised HOF fits the bill.

Now, I've eliminated calls to `provided`, and I get exactly the checking
that I want! The `log-accumulator` atom now collects all my log
messages, and I can verify them directly after the fact, rather than
hacking around checking that certain methods have been called at certain
times. Now, when I install this middleware, I just do:

{% highlight clojure %}
(log-request-provider log/info)
{% endhighlight %}

and get the logging I had before. If I decide I now want these things
logged at the warning level, or to syslog, or kibana, I only have to change my code in one place.

## A brief note on Dependency Inversion
If this looks familiar to OO developers, it should be. If you've been
following [SOLID][solid] principles, you might
recognise this as similar to the Dependency Inversion Principle. In this
case, we've lifted our `log-request` middleware via the
`log-request-provider` function, which now allows us to specify any
logger we like at runtime (and test time).

## Wrapping up
This may be a trivial example, but I hope these principles can be
applied to even slightly more complicated cases.

[lambda-days]: http://lambdadays.org
[lars-hupel-twitter]: http://www.twitter.com/larsr_h
[lars-hupel-talk]: https://speakerdeck.com/larsrh/functional-mocking
[solid]: http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)
[midje]: https://github.com/marick/Midje
