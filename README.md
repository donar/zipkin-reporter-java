[![Gitter chat](http://img.shields.io/badge/gitter-join%20chat%20%E2%86%92-brightgreen.svg)](https://gitter.im/openzipkin/zipkin) [![Build Status](https://travis-ci.org/openzipkin/zipkin-reporter-java.svg?branch=master)](https://travis-ci.org/openzipkin/zipkin-reporter-java) [![Download](https://api.bintray.com/packages/openzipkin/maven/zipkin-reporter-java/images/download.svg) ](https://bintray.com/openzipkin/maven/zipkin-reporter-java/_latestVersion)

# zipkin-reporter-java
Shared library for reporting zipkin spans onto transports including http and kafka. Requires JRE 6 or later.

# Usage
These components can be called when spans have been recorded and ready to send to zipkin.

## Encoder
The span encoder is a specialized form of Zipkin's Codec, which only deals with encoding one span.
It is also extensible in case the type of span reported is not `zipkin2.Span`

## Reporter
After recording an operation into a span, it needs to be reported out of process. There are two
builtin reporter implementations in this library, although you are free to create your own.

The simplest mechanism is printing out spans as they are reported.

```java
Reporter.CONSOLE.report(span);
```

## AsyncReporter
AsyncReporter is how you actually get spans to zipkin. By default, it waits up to a second
before flushes any pending spans out of process via a Sender.

```java
reporter = AsyncReporter.create(URLConnectionSender.create("http://localhost:9411/api/v2/spans"));

// Schedules the span to be sent, and won't block the calling thread on I/O
reporter.report(span);
```

### Tuning

By default AsyncReporter starts a thread to flush the queue of reported
spans. Spans are encoded before enqueuing so it is easiest to relate the
backlog as a function of bytes.

Here are the most important properties to understand when tuning.

Property | Description
--- | ---
`queuedMaxBytes` |  Maximum backlog of span bytes reported vs sent. Corresponds to `ReporterMetrics.updateQueuedBytes`. Default 1% of heap
`messageMaxBytes` | Maximum bytes sendable per message including overhead. Default `Sender.messageMaxBytes`
`messageTimeout` |  Maximum time to wait for messageMaxBytes to accumulate before sending. Default 1 second
`closeTimeout` |  Maximum time to block for in-flight spans to send on close. Default 1 second

#### Dealing with span backlog
When `messageTimeout` is non-zero, a single thread is responsible for
bundling spans into a message for the sender. If you are using a blocking
sender, a surge of reporting activity could lead to a queue backup. This
will show in metrics as spans dropped. If you get into this position,
switch to an asynchronous sender (like kafka), or increase the concurrency
of your sender.

## Sender
The sender component handles the last step of sending a list of encoded spans onto a transport.
This involves I/O, so you can call `Sender.check()` to check its health on a given frequency. 

Sender is used by AsyncReporter, but you can also create your own if you need to.
```java
class CustomReporter implements Flushable {

  --snip--
  URLConnectionSender sender = URLConnectionSender.json("http://localhost:9411/api/v2/spans");

  Callback callback = new IncrementSpanMetricsCallback(metrics);

  // Is the connection healthy?
  public boolean ok() {
    return sender.check().ok();
  }

  public void report(Span span) {
    pending.add(SpanBytesEncoder.JSON_V2.encode(span));
  }

  @Override
  public void flush() throws IOException {
    if (pending.isEmpty()) return;
    List<byte[]> drained = new ArrayList<byte[]>(pending.size());
    pending.drainTo(drained);
    if (drained.isEmpty()) return;

    sender.sendSpans(drained, callback).execute();
  }
```

### Legacy Encoding
V2 builders use json v2 encoding, which is easy to understand and twice
as efficient as the v1 json encoding. However, it relies on recent (1.31+)
versions of zipkin server.

You can switch to v1 encoding like so:

```java
reporter = AsyncReporter.builder(URLConnectionSender.create("http://localhost:9411/api/v1/spans"))
                        .build(SpanBytesEncoder.JSON_V1);
```
