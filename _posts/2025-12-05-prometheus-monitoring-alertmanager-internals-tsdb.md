---
title: "Exploring Prometheus Internals: TSDB and XOR Encoding"
author: cef
date: 2025-12-04
categories: [Technical Writing, Open Source]
tags: [Monitoring, Prometheus, Alertmanager, Go]
render_with_liquid: false
description: "How Prometheus Monitoring work? This deep dive explores monitoring, alerting, TSDB and Gorilla XOR encoding!"
---

Monitoring and alerting are like working out: you have to do them to keep your system healthy. You can enjoy it, neglect it, and just ignore it. At your peril.


## Why Do We Need Monitoring?

In case you haven't had the pleasure of being in an on-call rotation to make sure some system is always up, the key idea is that for critical apps on which a lot of people depend, it's crucial to maintain uptime: the app must always be running and healthy.


We monitor an app or a system by exposing its metrics. A metric is a value or measurement of a certain attribute: available disk space, available memory, number of errors or warnings, number of requests, request latency, etc.

We store these values in a system and configure alerting rules. For example, if metric_value > X for more than 30 seconds, you have permission to freak out and wake me up even if it's 2 AM. We can have severity levels: a Warning if we have less than 30% disk space available, and an actual Critical alert (or "page") if it's below 10%.

Prometheus is widely regarded as the leading open-source monitoring solution. Its alerting component, Alertmanager, lets you define alerts and route them based on severity and labels. With powerful routing rules, you can send notifications to the right place (email, Slack, PagerDuty, Discord, and more) and even to specific teams or channels. A database issue goes to the DB team, an auth failure goes to the auth team, and everyone gets alerted only when it matters to them.

```yaml
# Warning-level alert
- alert: LowDiskSpaceWarning
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.30
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Low Disk Space (warning)"
    description: "Disk free < 20% for 2 minutes."

# Critical-level alert
- alert: LowDiskSpaceCritical
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.10
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Low Disk Space (critical)"
    description: "Disk free < 10% for 1 minute."

######

receivers:
  - name: 'email-alerts'
    email_configs:
      - to: 'sleep-deprived@example.com'
        send_resolved: true

  - name: 'pagerduty-alerts'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_INTEGRATION_KEY'
        severity: 'critical'
        send_resolved: true
```



As you can see, if we define an alert, `expr` is the expression using the collected metrics, and it will be triggered if it's true for longer than the specified `for` duration (2 minutes for Warning or 1 minute for Critical). A Warning sends an email, and a Critical alert notifies PagerDuty, which will contact the on-call engineer by any means possible (text, notification, call, electrical shock through a bracelet, etc.).

Defining what to alert on is a delicate balance. You don't want too many alerts, which would result in alert fatigue, but you still want to be notified if something goes wrong. 

One cool feature Alertmanager has is [Grouping](https://prometheus.io/docs/alerting/latest/alertmanager/#grouping). If your app fails, multiple alerts are likely to be triggered (status code 500, uptime, unreachable, other systems depending on the app failing). One thing failing could trigger an outage and a storm of alerts. Grouping sends a compact notification instead of a gazillion ones.

## Prometheus Overview and Architecture

Prometheus's [official documentation](https://prometheus.io/docs/introduction/overview/#architecture) is a sight for sore eyes: concise, straightforward, informative, unpretentious. I'm not sure why, but that's the vibe it gives (as opposed to some other docs that shall remain nameless). I wonder how big a role this accessibility played in Prometheus's success.

From the official doc:

![prometheus architecture diagram](/assets/architecture-prometheus.svg)

> Prometheus scrapes metrics from instrumented jobs, either directly or via an intermediary push gateway for short-lived jobs. It stores all scraped samples locally and runs rules over this data to either aggregate and record new time series from existing data or generate alerts. Grafana or other API consumers can be used to visualize the collected data.

What a beautifully written overview. Hard to find a redundant word there. Let's unpack it.

### Scraping and Instrumentation

> Prometheus scrapes metrics from instrumented jobs

Prometheus is **pull-based**, unlike other **push-based** systems. The Prometheus process has a list of "targets" (essentially IP:port), and it sends an HTTP request to an endpoint exposed by the target. For example, you query `http://app.cefboud.com:8000/metrics` and get:

```
myapp_http_requests_total{method="GET",endpoint="/"} 1523
myapp_http_requests_total{method="POST",endpoint="/upload"} 87

myapp_memory_usage_bytes 834568192
myapp_disk_usage_bytes 4563456000
myapp_disk_available_bytes 20971520000
```

This endpoint is queried every **scrape interval** (15 seconds by default), and each value (together with its current timestamp) is added to a time series identified by the metric name (`myapp_http_requests_total`, `myapp_disk_usage_bytes`, etc) and labels (`method`, `endpoint`), if any.

```
────────────────────────────────────────────────────────────────────────►
     10:00         10:00:15      10:00:30      10:00:45

myapp_memory_usage_bytes:
    ● 834M ------ ● 835M ------ ● 836M ------ ● 837M

myapp_http_requests_total{method="GET", endpoint="/"}:
    ▲ 1523 ------ ▲ 1524 ------ ▲ 1525 ------ ▲ 1526

myapp_http_requests_total{method="POST", endpoint="/upload"}:
    ◆ 87 -------- ◆ 88 -------- ◆ 89 -------- ◆ 90
```

Every 15 seconds, all the time series (metric + labels) are scraped and added to Prometheus's TSDB (time series database).

Now, how the heck does the app expose this `/metrics` endpoint? Well, that's what we mean by "instrument".

We instrument an app by defining metrics via the Prometheus client library and actually keeping track of things. Each time a GET request arrives on `/`, we increase the `myapp_http_requests_total` metric with the appropriate labels. If it's a POST request on `/upload`, we increment the metric with different labels, that's a different time series.


```go
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var httpRequests = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "myapp_http_requests_total",
		Help: "Count of HTTP requests by method and endpoint.",
	},
	[]string{"method", "endpoint"},
)

var registry = prometheus.NewRegistry()

func main() {
	// Register the metric with Prometheus
	registry.MustRegister(httpRequests)
  
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		httpRequests.WithLabelValues("GET", "/").Inc()
		w.Write([]byte("Hello from /"))
	})

	http.HandleFunc("/upload", func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodPost {
			httpRequests.WithLabelValues("POST", "/upload").Inc()
		}
		w.Write([]byte("Upload OK"))
	})

	// Expose /metrics endpoint
	http.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{}))

	http.ListenAndServe(":8000", nil)
}
```

```
go run main.go

# let's run the following a few times
curl http://localhost:8000/
curl -X POST http://localhost:8000/upload -d "file=data"
```

After fooling around a few times, when I access `/metrics`, I can find

```
# HELP myapp_http_requests_total Count of HTTP requests by method and endpoint.
# TYPE myapp_http_requests_total counter
myapp_http_requests_total{endpoint="/",method="GET"} 4
myapp_http_requests_total{endpoint="/upload",method="POST"} 1
```

And that's really it. It's up to the app to define its metrics. Here we defined a counter metric using NewCounterVec, which is a metric that can only increase.

Another is a gauge, defined using NewGauge, which represents a value that can change up or down (e.g., memory or disk usage). 

A third one is a histogram, which provides quantiles/percentiles and is useful for durations (e.g., latency). It essentially defines intervals (buckets) and counts the occurrences within each interval. From that, using some simple arithmetic, we can calculate things such as the p50 percentile, the threshold below which 50% of the observed values fall, or the p99, meaning 99% of values are less than that number.

```go

// Gauge: arbitrary changing value
var currentLoad = prometheus.NewGauge(
	prometheus.GaugeOpts{
		Name: "myapp_current_load",
		Help: "Current load value (example gauge).",
	},
)

// Histogram: request duration
var requestDuration = prometheus.NewHistogramVec(
	prometheus.HistogramOpts{
		Name:    "myapp_request_duration_seconds",
		Help:    "Histogram of request durations.",
		Buckets: prometheus.DefBuckets, // default: 0.005 -> 10 seconds
	},
	[]string{"endpoint"},
)
```

That's for custom apps. How about existing apps or systems for which there is no code? Well, in that case, we can use an exporter which a tool that collects the metrics and exposes them. A popular one is `node_exporter` which calculates a node or machine OS and hardware metrics and exposes them. Another one is `mysql_expoter`.



We just unpacked the first sentence! Let's keep going. 

### Time Series Database
> It stores all scraped samples locally

The amazing thing about Prometheus is how easy it is to run. There is just one Go binary that you can download and run as follows:

```sh
curl -LO  https://github.com/prometheus/prometheus/releases/download/v3.8.0/prometheus-3.8.0.darwin-arm64.tar.gz
tar -xvf prometheus-3.8.0.darwin-arm64.tar.gz 
cd prometheus-3.8.0.darwin-arm64/
./prometheus --config.file=prometheus.yml
``

And you're off to the races. By default `prometheus.yml` has

```yml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"
```

We can define targets, here we're scraping `localhost:9090`  which is exposed by prometheus itself. But you can point it to any app or exporter that's accessible via HTTP and that has been instrumented. 

Prometheus stores this data locally, there is no distributed system shenanigans. If your data gets too large, you need to make use of [remote storage](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage), which provides compatible long-term storage. Prometheus itself is intended for short to mid-term retention. After letting my Greek titan run for some time:


```sh
prometheus-3.8.0.darwin-arm64 $ tree data 
data
├── 01KCYNWZ5ZJWX7QRR8XR69XYG8
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01KCYS09WRNHBZXQJMAP3GCMVY
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── chunks_head
│   └── 000001
├── queries.active
└── wal
    ├── 00000003
    ├── 00000004
    ├── 00000005
    └── checkpoint.00000002
        └── 00000000
```

A lot is happening! Reading about Prometheus's TSDB is beyond fascinating. I recommend this [blog series](https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/), which is delightful.
Prometheus stores data in blocks (each block is ~2 hours' worth of data). `01KCYS09WRNHB...` directories are the blocks. When querying data, depending on the time range, the Prometheus engine reads the appropriate directory and uses the `index` file to locate the time series data in chunk files (e.g., `chunks/000001`).

Fundamentally, that's the meat and potatoes of it. The metrics data is available in the chunk files (each file can span up to 512 MB), and we have gigabytes and gigabytes of data. And the `index` of each block knows exactly the offsets within the files for each time series the block contains and is used to fetch the data.

You might have noticed those `chunks_head` and `wal` directories. They had to exist, they couldn't just blend with the regular blocks and keep things simple!

Well, my friend, they're part of the reason Prometheus can scrape thousands and thousands of targets without blinking. When samples come in, they are kept in memory in the chunks head or the head block instead of the orders-of-magnitude slower disk, and only after the head block is full (2 or 3 hours' worth of data) do we write this data into a compact immutable block. This would, however, mean that the most recent data would be lost if the process crashes or is stopped. Well, thanks to the WAL (Write-Ahead Log), it isn't. As samples come in, they are kept in memory and sequentially added to the WAL files on disk.

Wait, Moncef! Hold the phone! You're contradicting yourself. You said the disk is slow. The disk is bad. The disk is… EVIL!

Not quite! Nuance, my dear chap!

Random-access disk writes are. Sequential append-only writes are dramatically faster than random I/O. Plus, the WAL writes are batched, and we write [32 KB](https://github.com/prometheus/prometheus/blob/22b3860e2ad21780728ec90727d944de90d43fa0/tsdb/wlog/wlog.go#L42) at a time. So instead of doing a write per sample or few samples in a random-access fashion, we batch a big WAL page write. The WAL is not something only used by Prometheus or TSDB, all DBs have a WAL of some sort.
After some time (2-3 hours), the head block is considered full and is written into a proper compact block. The WAL files are **checkpointed**, meaning they're stripped of any redundant data that has already been written to the block.

One further clarification: only the most recent chunk ([1024 bytes](https://github.com/prometheus/prometheus/blob/22b3860e2ad21780728ec90727d944de90d43fa0/tsdb/chunkenc/chunk.go#L56)) is really kept in memory. Once a chunk is filled, it's written to disk and memory-mapped using [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) (the syscall behind all the magic of high-performance systems). By being mmaped, the data remains in memory if the OS has enough space; otherwise, it's moved to disk and brought back when needed (during a query). The cool thing is that the move back and forth is transparent to the process when using mmap. You access your data as if it's in memory, but it might not be!


### Gorilla XOR Encoding
One bit (pun intended) that really delighted me while digging through Prometheus internals is how insanely efficient the storage format is. Because if you scrape 1000 targets every 15 seconds, each with 500 time series, and each sample is 16 bytes (8 bytes ts + 8 bytes value), the raw numbers are... scary:

```
(24 * 3600 / 15) * 1000 * 500 * 16 = 46,080,000,000 bytes
```

Yep, 46 GB per day. That's clearly not going to scale.

Enter **Gorilla-style compression**. The magic observation: time series don't change much between samples (timestamp and value pairs). 

Values? 10GB, 10GB, 10.1GB, 10.1GB… tiny shifts.  Timestamp? Almost constant.

Let's remember the following binary operation: `N XOR N = 0`. 

XORing a number with itself yields 0, and the more similar two numbers are (bitwise), the more zeros their XOR will contain.

Prometheus exploits this heavily. Since metric values are usually very close, it stores only the XOR against the previous value. That XOR is likely to have many zeros, allowing it to be encoded using far fewer than the full 64 bits.

For the **first** value, all 64 bits are stored. After that:

* **If the value is identical:** the XOR is 0, so write a single `0` bit. Done.
* **If the value is different:** write `1`, then encode the XOR in a compact form:

```
<5 bits leading zeros> <6 bits significant width> <significant bits>
```

Trailing zeros are inferred as `64 - leading - width`. Because consecutive values are usually close, the XOR often has many leading and trailing zeros, making this encoding extremely compact. This allows us to store stable metric
with just one bit (instead of 64) and small changes with way less bits, as few as 12 bits instead of 64.

Timestamps aren't XORed, they use **delta-of-delta** encoding. Because scrape intervals barely move, the deltas barely move, and the delta-of-delta is often zero and encodes to just a couple bits.

Put it all together and you compress what would be tens of gigabytes per day into something tiny enough that Prometheus can handle millions of series without breaking a sweat.

The [implementation reads](https://github.com/prometheus/prometheus/blob/22b3860e2ad21780728ec90727d944de90d43fa0/tsdb/chunkenc/xor.go#L406) almost like a paragraph (is this Go's superpower?).

Here's the same explanation in video format. 
<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/666Z9Imq2dE"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>

### Out-of-order Samples
By definition, Prometheus handles ordered data. You scrape metrics from targets and append them to their respective series. New samples must have timestamps newer than existing ones, otherwise, they are rejected as "out of order."

This is the canonical behavior. Sometimes, however, you want to ingest older data into the TSDB. Why? Metric data may be aggregated in systems that introduce delays. IoT devices may take time to send data. A system might be down for a few hours and, once back up, wants to submit its metrics.

Before v2.39, these use cases were merely an elusive dream. With v2.39, the [`out_of_order_time_window`](https://promlabs.com/blog/2022/10/05/whats-new-in-prometheus-2-39/#experimental-out-of-order-ingestion) setting was introduced, enabling out-of-order ingestion. The [design document](https://docs.google.com/document/d/1Kppm7qL9C-BJB1j6yb6-9ObG3AbdZnFUBYPNNWwDBYM) is an excellent read: it explains how out-of-order (OOO) samples are ingested in memory and on disk (head and blocks), and how queries behave when regular and OOO samples coexist.

OOO samples are stored in their own HeadChunk in memory alongside normal chunks. These OOO chunks are ordered so that new samples can be inserted at any position, rather than sequentially like regular chunks. 

At query time, the engine reads from both chunk types and merges the results while handling conflicts.

OOO samples are written to a Write-Behind Log (WBL), not the regular WAL.

Blocks on disk may overlap when they contain OOO samples. Prometheus reconciles this later during compaction.


## Next

I think it'd be fun to explore the implementation of PromQL and the query engine. Next time maybe.


<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/FgCdVQsViRA"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>