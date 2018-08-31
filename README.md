# bpmux

bpmux is a family of protocols for multiplexing messages, requests/responses and duplex streams over a single logical connection between to endpoints. It supports backpressure on all (non-meta) data, and it supports heartbeat pings (that can correctly [work over tcp](http://250bpm.com/blog:22)). Each protocol of the family implements the bpmux abstractions over a specific kind of connection (e.g. udp, dccp, sctp, reliable ordered connections like tcp, etc.).

## Abstractions

A *payload* is an arbitrary sequence of bytes, with any length between `0` and `2^64 -1` (inclusive). A *sink* is a logical channel a peer can send data to, a *stream* is a logical channel where it can receive data from. Most data is sent in the form of *messages*, which contain an arbitrary payload. All messages sent down the same sink are guaranteed to be received at the corresponding stream in the same order.

The *top-level* context of a bpmux connection is both a sink and a stream for both peers. In addition, it is possible for either peer to open up new streams (the other peer then gets a corresponding sink it can use to send data to the stream), to open up new sinks (the other peer then gets a corresponding stream it can use to receive data from the sink), or to open up new duplexes which serve as both a sink and a stream (the other peer gets a corresponding duplex as well). The opening of a new sink/stream/duplex carries an arbitrary payload.

The top-level also supports *request*/*response* pairs, other duplexes do not. A request consists of an arbitrary payload, the other peer can then send a response to it, again with an arbitrary payload. Unlike a simple exchange of messages, there is a one-to-one mapping between requests and the corresponding responses.

Requests and streams support *cancellation*, i.e. notifying the peer that no more data is desired. A cancellation carries an arbitrary payload. Responses and sinks support *closing*, i.e. notifying the peer that no more data will be send. Closing carries an arbitrary payload. While not mandatory, it is often appropriate to consider cancellation/closing with a zero-length payload as normal termination, and to consider other payloads as error conditions.

Any duplexes (including the top-level) support both cancellation and closing, these can be done independently. With a duplex, it is possible for one endpoint to cancel it and for the other to close it simultaneously. Applications built on bpmux should handle this gracefully. The same is true for requests/responses.

An implementation of the bpmux abstractions is free to place upper limits on the number of sinks, streams, duplexes and requests that can coexist at the same time. It is recommended to support at least `2^32 - 1` concurrent ones. If there is a limit, the programming interfaces should be able to signal when the limit is hit, and applications should correctly handle this case.

All sending of payloads is subject to *backpressure*, which is tracked on a per-stream basis (including the top-level). At any time, one peer can give some *credit* to a stream. Sending messages/requests/sink-cancellations down a sink consumes as much credit as the size of the payload. If the credit on a sink reaches zero, no more data can be sent to it, until the peer has given more credit to it. If a peer receives more data on a channel than it gave credit for, it must immediately close the connection.

Request-cancellations, Responses, creation of a sink/stream/duplex, closing of a response, and closing of a non-duplex stream consume top-level credit. Closing a duplex stream consumes credit on the corresponding sink.

Communication must work even if only a single byte of credit is given at once. Implementations of the bpmux abstract specification must thus include a mechanism to split up payloads into arbitrarily fine parts. Everything else would be prone to deadlocks.

Note that credit-based backpressure only throttles payload data, not meta data (such as giving credit or the metadata necessary for splitting up payloads). Otherwise, there would be deadlocks where neither endpoint has enough credit to grant more credit to the other endpoint. The backpressure mechanism thus does not prevent a malicious peer from spamming the connection, e.g. by granting a lot of credit one byte at a time, or simply by sending an arbitrary number of zero-length messages. Dealing with malicious peers is out of scope for bpmux.

For any stream (including the top-level) and for any unanswered request, a peer may send a *heartbeat ping* at any time. The other peer should then respond with a corresponding *heartbeat pong*. If the heartbeat pong does not arrive after a sensible time, the stream/request can be considered broken. For resilience, implementations should still send a cancellation to a stream/request that has timed out. Heartbeats do not consume any credit.

## Protocols Implementing the Abstractions

- [bpmux-rel](https://github.com/AljoschaMeyer/bpmux-rel) (reliable, ordered, bidirectional communication channels)
