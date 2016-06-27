%%%

    Title = "A Standard for the exposing metrics data from software"
    abbrev = "Labeled metrics exposition format"
    category = "info"
    docName = "draft-rfc-01"
    area = "Internet"
    workgroup = "Ops Working Group"
    keyword = [""]

    date = 2016-06-27T00:00:00Z

    [[author]]
    initials="T."
    surname="Proud"
    fullname="Matt"
    organization = "Prometheus"
      [author.address]
      email = "matt.proud@gmail.com"
      phone = "X"
      [author.address.postal]
      street = "X"
      city = "X"
      code = "X"

%%%

{mainmatter}

# Overview and Rational

# Metrics Format

# Discussion

# Security Considerations

{backmatter}

# History


The original Prometheus prototype exposition format used JSON text as the default.  JSON is slow and cannot have it data model structure defined in one single place like a IDL.  Numerous encoders/decoders use runtime reflection to attempt to map field values back to fields on dumb data objects (DDO).  This meant routine schema changes would break legacy Prometheus servers or clients.  Further all Prometheus metrics are floats, and JSON lacked standardization around NAN and INF.

Protocol Buffers were a natural choice since they were mature, production ready, and defined the data model in a single source IDL file (.proto) that the protoc compiler emits into user code.  If one designs the Protocol Buffer schema correctly, it is nearly 100 percent compatible with future changes on both server/client end.  It is also fast.  The internal encoding scheme that Prometheus uses is a variation of RecordIO (you'll find that if you lookup Sawzall), which is a minimized version of an internal encoding scheme used at Google.  It makes Protocol Buffer message streams appendable and support near random traversal.  It also means decoding can be done incrementally , which saves resources.  This is not a possibility in most JSON implementations.  Basically this is the textbook reason why folks use Protocol Buffers.  Why not Avro or Message Pack?  They were immature and lacked the single site IDL file.  Why not Thrift?  It is an inferior reverse engineering of Protocol Buffers, and many of its language-specific generated code emitters exhibited bugs and inconsistencies (e.g., in 2012, the encoding for the first enum value in Thrift in its binary wire format was "0" or "1" depending on which language you used even from the same Thrift release).  Sorry, that is Mickey Mouse stuff.

The text format emerged strictly to support two cases: no Protocol Buffer support in an environment (e.g., shell script) and support one-off human debugging.

The binary format was also designed to not be stateful, meaning each individual decoded message from the stream could be treated in isolation.  The text format lacks this: state spans multiple lines.  This increases complexity.

Because gRPC did not exist at the time and Thrift had defects, HTTP was the common denominator for transport that supports both binary and text formats.  The final protocol version uses HTTP MIME type autonegotiation to resolve whether binary or text is used.  Previous versions used HTTP request headers, but that is a poor practice.  I had considered using different named endpoints (/metrics.pb or /metrics.txt) for different formats (á la REST), but that would have increased complexity with middleware registration in each client library.  HTTP MIME auto-negotiation is standardized too and predictable.

The metrics in the binary format are done as concrete types (as opposed to loose key-value pairs as some had proposed).  That takes guess work out of clients' interpretation of them.  Protocol Buffers did not have union "oneof" or "any" support at that time (became public last year).  This necessitated the creation of the separate per-metric enum field to instruct which concrete metric type was populated: counter, gauge, histogram, summary, untyped.  This meant the server could perform loose invariant enforcement and reject bad messages (e.g., a metric that populated both gauge and histogram fields).  Were I to do this format over again, I would deprecated the individual gauge, counter, histogram nested fields in the metric entry and replace it with a "oneof".  I am unsure this would be wise: I do not think Protocol Buffer 3 ("any", "oneof", and map<K,V>) support is sufficiently universal in the public ecosystem to warrant a conversion.  This might be worth fixing in two years.  Google is working on and publishing more official Protocol Buffer language integrations.

Why pull not push for transport?  Push requires a HA receiver for the metrics and possibly for the metrics producers/clients to maintain more state on the receiver, what has been sent, how old, publication interval, extra configuration, etc.  Pull was simply easier.  It also meant all that a client required was to have support for HTTP, which is ubiquitous.  Supporting push directly is more practical these days, but …

In terms of metric design (not wire format), these metric types match most software, industrial, and sciences use cases.  Other metric types could be added later if the needs warrant it.
