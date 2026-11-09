# 0001. REST at the edge, events between services, no gRPC

- Status: accepted
- Date: 2026-11-09

## Context

Horizon Social Network is split into three backend services (user-service, post-service,
feed-service) behind an Istio ingress, with a web app and a mobile app as clients. Because the
system is built as microservices, the team asked whether services should talk gRPC instead of
REST.

Facts that shape the answer:

- The clients are a browser and a mobile app. Browsers cannot call native gRPC. They need
  gRPC-Web and a proxy that translates it, and gRPC-Web has no client or bidirectional
  streaming.
- The public API is already specified as REST in `hsn-openapi.yml`, with real-time feed
  updates over Server-Sent Events (`GET /feed/stream`).
- The only cross-service needs are:
  1. feed-service must learn when a post is created or deleted, and when its like, comment or
     share counts change, so it can update feeds and push `post.created`, `post.deleted` and
     `post.stats` events.
  2. feed-service must know a user's connections to decide whose feeds a post belongs to.
- Redis is already part of the stack.
- Every team member must be able to explain any part of the system at milestone reviews.

## Decision

1. **Public API: REST + JSON**, as specified in `hsn-openapi.yml`. Real-time updates use SSE.
2. **Post changes travel as events on Redis Streams.** post-service appends an event to a
   stream after each successful write. feed-service reads the stream through a consumer group.
   Event names match the SSE events: `post.created`, `post.deleted`, `post.stats`.
3. **Synchronous service-to-service calls use plain HTTP/JSON** on internal endpoints that are
   not routed through the ingress and not part of `hsn-openapi.yml`.
4. **No gRPC** for now.

## Consequences

- One protocol and one contract format (OpenAPI) for the whole team. Endpoints can be tried
  with curl, and the same tooling works for public and internal calls.
- Istio routes, secures (mTLS) and measures HTTP traffic, so Prometheus metrics and SLOs are
  the same as they would be with gRPC.
- Redis Streams keep events after delivery, and consumer groups track what was read. A
  feed-service restart resumes from the last acknowledged event instead of losing posts, and
  stream ids can back the SSE `Last-Event-ID` resume.
- post-service does not depend on feed-service being up. A post is saved even if feeds update
  a few seconds later.
- Events are delivered at least once, so feed-service must handle the same event twice
  without duplicating a feed entry.
- Internal HTTP endpoints have no generated typed client. Each internal endpoint gets its
  request and response shape documented next to the service that owns it.

## When to revisit

Reconsider gRPC for service-to-service calls if synchronous internal calls grow to several
services calling each other at high volume, or if typed cross-language contracts become a real
source of bugs. If adopted, use it only between services, keep `.proto` files in this hub
repository, and keep REST at the edge.

## Options considered

- **gRPC everywhere.** Rejected: browsers need gRPC-Web and a translating proxy, the REST spec
  would be thrown away, and every repository would need a protobuf code generation step.
- **gRPC internally, REST at the edge.** Rejected for now: two protocols and a shared proto
  toolchain for about two internal interactions.
- **Synchronous HTTP call from post-service to feed-service on each post.** Rejected: couples
  post creation to feed-service availability and loses updates when feed-service is down.
- **Redis Pub/Sub instead of Streams.** Rejected: messages are lost when no subscriber is
  connected, so a feed-service restart would miss posts.
