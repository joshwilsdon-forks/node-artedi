# Design

* [Unique Design Elements](#unique-design-elements)
    * [Parent/Child Relationship](#parentchild-relationship)
    * [Log/Linear Buckets](#loglinear-buckets)
    * [Dynamic Labelling](#dynamic-labelling)
    * [Triggered Metrics](#triggered-metrics)
    * [Metric Expiry](#metric-expiry)
    * [Children are leaf collectors](#children-are-leaf-collectors)
* [Problems](#problems)
    * [Metric Cardinality](#metric-cardinality)
    * [Javascript Numbers](#javascript-numbers)
* [Other Thoughts and Ideas](#other-thoughts-and-ideas)
    * [Aggregated Metrics](#aggregated-metrics)
    * [Dropwizard and Prometheus Histograms](#dropwizard-and-prometheus-histograms)
    * [Instrumenting Libraries](#instrumenting-libraries)
    * [Scrape Endpoint and Push Gateway Support](#scrape-endpoint-and-push-gateway-support)
    * [Retention Period](#retention-period)
    * [Naming](#naming)
    * [CLI Usage](#cli-usage)

This document describes some of the unique design elements that factored into
the creation of this library. It also mentions some speculative ideas, and a
couple problems that may come about.

## Unique Design Elements
### Parent/Child Relationship

The parent/child relationship is very important for our use case. This
concept is taken from [bunyan](https://github.com/trentm/bunyan).
This is a description of how it is implemented in this library.
A parent collector is created with a set of properties including a name,
and a set of labels. Labels are key/value pairs that can be used to
'drill down' metrics. For example, to create a new parent collector:

```javascript
var collector = artedi.createCollector({
    labels: {
        zone: ZONENAME
    }
});
```
And to create a child collector:
```javascript
var gauge = collector.gauge({
    name: 'marlin_agent_jobs_running',
    labels: {
        key: 'value'
    }
});
```

All of the child collectors created from the parent collector will have
the label `zone=ZONENAME`. The child that we created is a gauge in this
case. A user could also call `.counter()`, `.histogram()`, or
`.gauge()`. Note that when we refer to histograms, we're referring to the
Prometheus-style histograms. There are currently no plans to implement
the Prometheus 'Summary' collector.

In the Prometheus nomenclature, gauges, counters, histograms, and
summaries are all called 'collectors.' For this library, we're also
calling the parent instance a 'collector.' The child collector (gauge)
that we just created has the following labels: zone=ZONENAME,key=value.
It inherited the 'zone' label from its parent. Any metric measurement
that we take will include these two labels in addition to any labels
provided at the time of measurement. For example:

```javascript
gauge.set(1, {
    owner: 'kkantor'
});
```
The reported metric from that operation will look something like this:
```
marlin_agent_jobs_running{zone="e5d03bc",key="value",owner="kkantor"} 1
```


### Dynamic Labelling
We can see in the last example that the metric inherited two labels, and
we additionally crated a third label (`owner="kkantor"`) on the fly.
Existing metric clients (prom-client, the Golang Prometheus client)
don't allow for dynamic creation of labels. We allow that in this
library. It is also unique for this library to allow for 'static'
key/value pairings of labels. Existing clients only allow users to
specify label keys at the time a collector is created, but we are
allowing a user to specify both a label key and value (`labels{'zone'}`
vs `labels{zone: ZONENAME}`).

By allowing on the fly creation of labels, we gain a lot of flexibility,
but lose the ability to strictly control labels. This makes it easier for
a user to mess up labeling. For example, a user could do something like
this:
```javascript
var counter = collector.counter({
    name: 'http_requests_completed',
    help: 'count of requests completed'
});

if (res.statusCode >= 500 && res.err) {
    counter.increment({
        method: req.method,
        err: res.err.name
    });
} else {
    counter.increment({
        method: req.method
    });
}

// Elsewhere in the code...
collector.collect(artedi.FMT_PROM, function (err, str) {
    console.log(str);
});

/*
 * If both of the counter.increment() statements are hit once each,
 * the output might look something like this:
 * HELP: http_requests_completed count of requests completed
 * TYPE: http_requests_completed counter
 * http_requests_completed{method="putobject"} 1
 * http_requests_completed{method="putobject",err="Service Unavailable"} 1
 */
```
The above example shows that two different metrics were created. This is
possibly the intended behavior, but depending on how queries are being
run, this may result in information being lost. Other implementations
would require a user to define labels up-front, and throw an error if
the user tried to create a label ad-hoc, like above.

Merits of up-front label declaration:
* Programming mistakes become runtime errors, rather than producing
    valid, but confusing data
    * This defines a type of 'metric schema'

Merits of dynamic label declaration:
* Flexibility, ease of use

In the end, we decided to implement dynamic declaration due to the
increase in flexibility.

### Triggered Metrics
Traditionally, metrics are collected continuously and synchronously. For
example, whenever a user makes a GET request, a counter to track HTTP requests
is incremented. This makes sense for lightweight things, like counting HTTP
requests, but doesn't work well for remote instrumentation. Remote
instrumentation might mean that we are trying to collect metrics from a Postgres
instance running on another machine, for instance. Instrumenting something like
Postgres would require us to make a network request, and may involve some heavy
lifting on the Postgres side as well depending on what type of information we're
trying to retrieve.

To provide for this use case, we'll introduce the notion of 'triggered metrics.'
A triggered metric can take two forms:
1. A metric that is observed only once metrics are scraped
2. A metric that is observed once in a while

An example of 1) could be a Gauge tracking the amount of free RAM on a system.
The amount of RAM that was free 10 seconds ago doesn't matter. The amount of RAM
that is free when we make our collection is what we want.

An example of 2) could be a Gauge for the total number of records stored in a
Postgres table. Something like that may involve a SQL query being run, and we
may only want coarse time granularity for such an operation (i.e. every couple
minutes). Note that this could also be implemented as an example of 1).

To accomplish this, the `collect()` function will be asynchronous. An API will
be provided that allows a user to register a metric with a scheduled collection
period. At collection time, the each of the metrics in the 'trigger registry'
could be invoked. This is similar to existing solutions, like
[boolean health checks](http://metrics.dropwizard.io/3.2.2/getting-started.html#health-checks).

Basic triggered metrics (type 1 from above) are implemented in node-artedi
version 1.1.0. Type 2 triggered metrics are not yet implemented.

### Metric Expiry
Metrics have an optional expiry feature that allows them to be reset to a
default value after a given time period has passed in which the value of the
metric is not otherwise updated. The expiry feature is currently only exposed
for gauge types, but it is a feature of the more general metric and could be
exposed for other types in the future. This feature was introduced in version
1.4.0.

An example use case for metric expiry is when a gauge is used to track the
progress of a finite event or sequence of steps, but the event has no discrete
final state that can be observed to indicate completion. Without intervention a
gauge will continue providing a reading of the last known value indefinitely
once the metric data is no longer extant. The expiry option provides a way to
infer the completion of the event or sequence from the fact that the metric
value and its associated timestamp are not updated within the observation period
and then take action to reset the value to a default.

### Children are leaf collectors
Children cannot be created from children. That is, a user can't call
.gauge() on a Counter to create a child Gauge.

The name of the metric (`marlin_agent_jobs_running` from our example) is
created by appending the `namespace` field from the parent collector to
the `subsystem` and `name` fields of the child, separated by
underscores.

When a child collector is created, the parent registers the collector.
This is done for two reasons. The first is that the user only needs to
call .collect() on the parent collector in order to generate metric
output. The second is so that child collectors are not garbage collected
into non-existence after they are dereferenced by whatever function
created them to measure a certain task. When a user calls .gauge() on a
parent collector, it may or may not create a new gauge depending on if
one with the same metric name (`marlin_agent_jobs_running` from our
example) already has been registered.

## Problems
### Metric Cardinality
To illustrate this problem, let's say that we have the label keys
`method`, `code`, and `username`. `method` can take the values `get`,
and `post`. `code` can take the values `200`, `500`, and `404`.
`username` can take any value from a list of 100 users. The number of
possible counters created in this situation is `(2*3)*100=600`. For this
very small data set, you can see that we have the possibility of having
a shitload of data. Keeping unbounded fields like `username` to a
minimum (or not including them at all) is very important to maintaining a
manageable amount of data (as well as conserving memory in the metric
client and server). This is called the problem of **metric
cardinality**.

### Javascript Numbers
Numbers in Javascript are 64-bit floats. The maximum value for a number in
Javascript is `(2^53)-1`. There is a possible danger here because of the way
Counters function. A Counter counts up, and is unbounded. We could theoretically
overflow the number value. If we instrument a process that performs one thousand
requests per second, and we were incrementing a counter for each request, it
would take us many years to reach the overflow point.

## Other Thoughts and Ideas

### Aggregated Metrics
Rather than making a programmer choose which fields to collect at the time of
observation, the programmer could observe all of the possible fields, and then
choose which fields to keep.

Take Muskie as an example. Currently we collect only a subset of fields
(latency, request method, response code, etc.). We could change that to collect
all of the information about a request (user, local/remote IP, metadata shard,
sharks, etc.). When we initially create the collector, we specify which of the
metrics we want to keep. The metrics that we don't want to keep are aggregated
away at observation time.

This is especially useful when we go to instrument things like Cueball, where
different fields will be relevant (and efficient to collect) for each
application component. For example, it may be efficient (in terms of
cardinality) for Muskie to collect a 'remoteIP' field, but not efficient for
CloudAPI to do so.

This makes it easier for application developers to know which fields will be
collected, and stops them from accidentally adding fields that may make querying
difficult.

### Dropwizard and Prometheus Histograms
Dropwizard is a Java framework that (among other things) provides a popular Java
library used to instrument applications. The instrumentation library is similar
in scope to `artedi`. There are a couple fundamental differences in the way that
Dropwizard's library and `artedi` function, especially with respect to
histograms.

Dropwizard histograms have a pretty complex implementation on the client-side.
As metrics are observed, they are added to a `reservoir`. Quantiles are
calculated each time a metric is observed. If a user wants to know, for example,
what the 90th percentile of a given metric is, it would be a simple lookup due
to the functionality of `reservoirs` and histograms. A `reservoir` is a type of
in-memory database that can have a number of different rules for eviction of
records. For example, a `reservoir` could be configured to use a `sliding
window`, which retains a user-defined `N` metrics in memory. When `N+1` metrics
have been observed, the first observed metric is evicted from memory. Dropwizard
places the burden of calculating quantiles on the client library.

Prometheus histograms have a relatively simple implementation. Prometheus places
the burden of calculating histogram quantiles on the server. When a metric is
observed, the Prometheus client simply increments a set of `buckets`. Metrics
are retained indefinitely on the client-side because they carry next to no cost
to retain.

Server-side histogram quantile calculation has benefits and drawbacks. One major
benefit is that a quantile based on any percentile can be calculated at any
time. With Dropwizard histograms, only a set number of quantiles can be
calculated, and they have to be defined when the client begins collecting
metrics. The major drawback to server-side quantile calculation is that the
server may have to iterate through an enormous amount of historical data to
produce a result. This can cause lead to slow server performance.

Prometheus clients may implement a metric type called a 'summary'. Summaries
more closely resemble Dropwizard-style histograms. Quantile calculation is done
on the client side, and there are sets of rules for how metrics are retained and
evicted. At this point Prometheus-style summaries are not implemented in
`artedi`, though we could add them at a later point if we find it necessary.

For more information on Prometheus histograms and summaries, see this page:
https://prometheus.io/docs/practices/histograms/ .

### Instrumenting Libraries
One cool thing that we might think about doing is instrumenting common
libraries. We are already planning on doing something like this for Cueball. We
could conceivably instrument something like Restify so we wouldn't need to
specifically track counts and latency of http requests in our applications. The
downside of doing something like this (specifically with Restify) is that the
library may not know all of the information that we may like to convey in our
metrics. Examples of those things would be usernames, request IDs, etc.

### Scrape Endpoint and Push Gateway Support
Some other metric client libraries provide a built-in server that can make it
more convenient to expose metrics. In our applications, we may be required to
stand up another server to expose metrics as a way to ensure metrics don't fall
into the wrong hands. We could design this in such a way that it is pluggable
so a user could choose to expose metrics via Restify, node-fast, a flat file,
or something else. Disadvantages to this approach is that it is potentially
quite a bit of extra code to maintain, and we may not be able to efficiently
write this in a way that gives us the flexibility we require. We will have to
revisit if this is needed or helpful.

Separately, it may make sense for us to add support of push gateways to this
library. For applications that are only behind private networks the pull model
will sometimes not work. Prometheus suggests placing a Prometheus server within
the boundaries of the private network, but that won't work well for our case
when application owners may create an arbitrary number of private networks. Push
gateways are one way to solve this, and would require client library support.

We'll have to revisit this topic when we have thought more about how we will
discover applications (and processes within applications) providing metrics.

### Retention Period
For now, the amount of time we retain metrics will be up to the operators of the
metric server. Retention periods are in the scope of a not-yet-created RFD, so
when we start working on an end-to-end solution for instrumentation we will have
to revisit the question of how long to retain scraped metrics.

### Naming
We should try to maintain standard naming conventions for our metrics. For
Prometheus, they are outlined
[here](https://prometheus.io/docs/practices/naming/). As a summary, metrics
should have generic names when it makes sense (`http_requests_completed`), and
specific names when needed (`marlin_agent_jobs_running`). Labels should be used to
break generic metrics into specific measurements. These two things allow for
more powerful queries to be made at the monitoring dashboard.

Names chosen for metrics are deceptively important. Changing names of metrics
after they have been running in the wild should be considered a breaking change.
Changing names of metrics results in a couple possibly unforeseen consequences.
The first is that dashboards and queries have to be recreated to use the new
metric name. The second, and more deceptive reason, is that changing names make
historical metric queries much more difficult, if not impossible.

This will be covered more in-depth in a future RFD, as this applies specifically
to the instrumentation of applications.

### CLI Usage
It may be useful to provide a CLI wrapper around this library to dynamically
instrument arbitrary applications. Dave Pacheco's
[statblast](https://github.com/davepacheco/node-statblast) is an example of
what we're would be going for. Imagine being able to instrument your favorite
system monitoring tools directly from the CLI! Further, we could use DTrace to
inspect a running system and instrument pieces of the application that are not
normally instrumented. An example of this could be counting Restify entry/exits
by pairing the already existing DTrace probes with this library to tie DTrace
output into the power of monitoring systems.

It may also separately be useful for applications to provide CLI tooling for
scraping their metrics. For instance, if my application exposes metrics via
node-fast, my application could provide a simple tool to allow CLI users
to call the proper fast RPC endpoint to access the metrics and print them to the
terminal. This portion is out of the scope of this document, but it is good to
keep in mind, and is related to the previous idea.
