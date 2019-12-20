---
title: Metrics
shortdesc: Gather metrics on your applications
tags:
- apps, metrics, accesslogs, warp10
---

<div class="panel panel-warning">
  <div class="panel-heading">
    <h4>Warning:</h4>
  </div>
  <div class="panel-body">
    Clever Cloud Metrics is still in beta.
  </div>
</div>

In addition to logs, you can have access to metrics to know how your application
behaves. By default, system metrics like CPU and RAM use are available, as well
as application-level metrics when available (apache or nginx status for instance).

# Publish your own metrics

We currently support two ways to push / collect your metrics: the `statsd` protocol and `Prometheus`.

The statsd server listens on port `8125`. You can send metrics using regular statsd protocol or using an advanced
one [as described here](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/statsd#influx-statsd).

We also support Prometheus metrics collection. By default our agent collects exposed metrics on `localhost:9100/metrics`.

If needed, you can override those settings with the two following environment variables:
- `CC_METRICS_PROMETHEUS_PORT`: Define the port on which the Prometheus endpoint is available
- `CC_METRICS_PROMETHEUS_PATH`: Define the path on which the Prometheus endpoint is available

# Display metrics

For each application, there is a `Metrics` tab in the console.

## Overview pane

To get a quick overview of the current state of your scalers, the overview pane
displays the current CPU, RAM, Disk and Network activity. On supported platforms,
you can also have access to requests / second, and GC statistics.

## Advanced pane

Advanced metrics allow you to access all gathered metrics,
on a specified time range.

## Custom queries

All metrics are stored in [Warp10](/doc/tools/warp10/), so you explore data directly with
the [quantum](/doc/tools/warp10/) interface, with [WarpScript](http://www.warp10.io/reference/). For instance,
you can derive metrics over time, do custom aggregations or combine metrics.

# Access Logs metrics

All your applications access logs are pushed to [Warp10](/doc/tools/warp10/). You are now able to process them directly in the console in the Metrics tab of your applications.

## Access Log data model

Access logs are defined in the `'accessLogs'` Warp10 class and there are three Warp10 labels available:

* owner_id;
* app_id;
* adc (reverse proxy used).

To reduce space used to store access logs, we defined a light key-value model as following:

```bash
t -> timestamp
a -> appId
o -> ownerId
i -> instanceId
ipS -> ipSource
pS -> portSource # 0 if undefined
s -> source
  lt -> latitude
  lg -> longitude
  ct -> city
  co -> country
ipD -> ipDestination
pD -> portDestination # 0 if undefined
d -> destination
  lt -> latitude
  lg -> longitude
  ct -> city
  co -> country
vb -> verb
path -> path
bIn -> bytesInt
bOut -> bytesOut
h -> hostname
rTime -> responseTime
sTime -> serviceTime
scheme -> scheme
sC -> statusCode
sT -> statusText
tS -> Haproxy termination_state
adc -> Reverse proxy hostname
w -> workerId (Sozu)
r -> requestId (Sozu)
```


## queries examples:

The main ways to use `accessLogs` data are to `FETCH` over its and to get interesting values by a JSON processing.

```warpscript
// Fetch on the 'accessLogs' class for your application id as labels
[ '<READTOKEN>' 'accessLogs' { 'app_id' '<APP_ID>'  } NOW 30 s ] FETCH

// get the path field
<% 
    DROP
    VALUES
    <% DROP
        JSON->
        'path' GET
    %> LMAP
%> LMAP
FLATTEN

// distinct|unique results
UNIQUE
```

A convenient way to integrate the intercepted data in a workflow, is to use the power of [warp10](/doc/tools/warp10/). Hence, it is often a good idea to use the GTS format to be able to apply all GTS treatment on the output.

In the following example, we get the `accessLogs` status codes and create a GTS as output to be able to use some FILTER or other treatment on it in a second time.

```warpscript
// Get all application status code  for the last hour
[ '<READTOKEN>' 'accessLogs' { 'app_id' '<APPLICATION ID>' } NOW 10 m ] FETCH
<%
  DROP
  'gts' STORE
  // output new GTS
  NEWGTS
  $gts 
  <%
    DUP
    // store the timestamp
    0 GET 'ts' STORE
    // store the status code
    -1 GET JSON-> 'sC' GET 'sC' STORE
    // Keep the same labels in the output GTS than in the input ones
    $gts LABELS RELABEL
    // Add timestamp and status code value to the output GTS
    [ $ts NaN NaN NaN $sC ] ADDVALUE
  %>
  FOREACH
%> LMAP
```

# Custom metrics

You can expose custom metrics via [`statsd`](https://github.com/etsy/statsd#usage).
These metrics will be gathered and displayed in advanced view as well.
On some platforms, standard metrics published over `statsd` are even integrated on the overview pane.

Metrics published over `statsd` are prefixed with `statsd`.

## statsd socket

To publish custom metrics, configure to use your client to push to `localhost:8125`
(it's the default host and port, so it should work with default settings as well).

## NodeJS example

You can use `node-statsd` to publish metrics

```javascript
// npm install node-statsd

const StatsD = require('node-statsd'),
      client = new StatsD();

// Increment: Increments a stat by a value (default is 1)
client.increment('my_counter');

// Gauge: Gauge a stat by a specified amount
client.gauge('my_gauge', 123.45);
```

## Haskell example

<!-- Maybe this should be put in haskell documentation? -->

In Haskell, metrics are usually gathered with [EKG](https://hackage.haskell.org/package/ekg).
The package `ekg-statsd` allows to push EKG metrics over `statsd`.

If you're using [warp](https://hackage.haskell.org/package/warp), you can use
`wai-middleware-metrics` to report request distributions (request count,
responses count aggregated by status code,
responses latency distribution).

EKG allows you to have access to GC metrics, make sure you compile your application
with `"-with-rtsopts=-T -N"` to enable profiling.

```haskell
{-# LANGUAGE OverloadedStrings #-}

-- you need the following packages
-- ekg-core
-- ekg-statsd
-- scotty
-- wai-middleware-metrics

import           Control.Monad                   (when)
import           Network.Wai.Metrics             (WaiMetrics, metrics,
                                                  registerWaiMetrics)
import           System.Metrics                  (newStore, registerGcMetrics)
import           System.Remote.Monitoring.Statsd (defaultStatsdOptions,
                                                  forkStatsd)
import           Web.Scotty

handleMetrics :: IO WaiMetrics
handleMetrics = do
  store <- newStore
  registerGcMetrics store
  waiMetrics <- registerWaiMetrics store
  sendMetrics <- maybe False (== "true") <$> lookupEnv "ENABLE_METRICS"
  when sendMetrics $ do
    putStrLn "statsd reporting enabled"
    forkStatsd defaultStatsdOptions store
    return ()
  return waiMetrics

main = do
  waiMetrics <- handleMetrics
  scotty 8080 $ do
     middleware $ metrics waiMetrics
     get "/" $
       html $ "Hello world"
```
