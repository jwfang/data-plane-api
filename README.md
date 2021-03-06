# Envoy v2 JSON REST and gRPC APIs

## Goals

This repository contains the draft v2 JSON REST and gRPC
[Envoy](https://github.com/envoyproxy/envoy/) APIs. Envoy today has a number of JSON
REST APIs through which it may discover and have updated its runtime
configuration from some management server. These are:

* [Cluster Discovery Service (CDS)](https://envoyproxy.github.io/envoy/configuration/cluster_manager/cds.html)
* [Rate Limit Service (RLS)](https://envoyproxy.github.io/envoy/configuration/overview/rate_limit.html)
* [Route Discovery Service (RDS)](https://envoyproxy.github.io/envoy/configuration/http_conn_man/rds.html)
* [Service Discovery Service (SDS)](https://envoyproxy.github.io/envoy/configuration/cluster_manager/sds_api.html)

Version 2 of the Envoy API will evolve existing APIs and introduce new APIs to:

* Allow for more advanced load balancing through load and resource utilization reporting to management servers.
* Improve N^2 health check scalability issues by optionally offloading health checking to other Envoy instances.
* Support Envoy deployment in edge, sidecar and middle proxy deployment models via changes to the listener model and CDS/SDS APIs.
* Allow streaming updates from the management server on change, instead of polling APIs from Envoy. gRPC APIs will be supported
  alongside JSON REST APIs to provide for this.
* Ensure all Envoy runtime configuration is dynamically discoverable via API
  calls, including listener configuration, certificates and runtime settings, which are today sourced from the filesystem. There
  will still remain a static bootstrap configuration file that will specify items
  unlikely to change during runtime, including the Envoy node identity, xDS
  management server addresses, administration interface and tracing
  configuration.
* Revisit and where appropriate cleanup any v1 technical debt.

## Status

The LDS/CDS/EDS/RDS APIs are now frozen and will maintain backwards
compatibility according to standard proto rules (e.g. new fields will not reuse
tags, field types will not change, fields will not be renumbered, etc.).

The remainder of the API (ADS, HDS, RLS, SDS, filter fragments other than HTTP
connection manager, the bootstrap proto) are draft work-in-progress. Input is
welcome via issue filing. Small, localized PRs are also welcome, but any major
changes or suggestions should be coordinated in a tracking issue with the
authors.

Implementation work has begun and work items are tracked at
[here](https://github.com/envoyproxy/envoy/issues?q=is%3Aopen+is%3Aissue+label%3A%22v2+API%22).

New features that correspond to the v2 API are initially tracked in this
repository. When they are agreed upon and the related PRs are merged, they
should be closed out and a corresponding issue created in
https://github.com/envoyproxy/envoy/ and tagged with `v2 API`. A reference to the
closed issue should also be included.

## Principles

* [Proto3](https://developers.google.com/protocol-buffers/docs/proto3) will be
  used to specify the canonical API. This will provide directly the gRPC API and
  via gRPC-JSON transcoding the JSON REST API. A textual YAML input will be
  supported for filesystem configuration files (e.g. the bootstrap file), in
  addition to JSON, as a syntactic convenience. YAML file contents will be
  internally converted to JSON and then follow the standard JSON-proto3
  conversion during Envoy config ingestion.

* xDS APIs should support eventual consistency. For example, if RDS references a
  cluster that has not yet been supplied by CDS, it should be silently ignored
  and traffic not forwarded until the CDS update occurs. Stronger consistency
  guarantees are possible if the management server is able to sequence the xDS
  APIs carefully (for example by using the ADS API below). By following the
  `[CDS, EDS, LDS, RDS]` sequence for all pertinent resources, it will be
  possible to avoid traffic outages during configuration update.

* The API is primarily intended for machine generation and consumption. It is
  expected that the management server is responsible for mapping higher level
  configuration concepts to API responses. Similarly, static configuration
  fragments may be generated by templating tools, etc. The APIs and tools
  used to generate xDS configuration are beyond the scope of the definitions in
  this repository.

* REST-JSON API equivalents will be provided for the basic singleton xDS
  subscription services CDS/EDS/LDS/RDS/SDS. Advanced APIs such as HDS, ADS and
  EDS multi-dimensional LB will be gRPC only. This avoids having to map
  complicated bidirectional stream semantics onto REST.

* Listeners will be immutable. Any updates to a listener via LDS will require
  the draining of existing connections for the specific bound IP/port. As a
  result, new requests will only be guaranteed to observe the new configuration
  after existing connections have drained or the drain timeout.

* Versioning will be expressed via [proto3 package
  namespaces](https://developers.google.com/protocol-buffers/docs/proto3#packages),
  i.e. `package envoy.api.v2;`.

* Custom components (e.g. filters, resolvers, loggers) will use a reverse DNS naming scheme,
  e.g. `com.google.widget`, `com.lyft.widget`.

* [Wrapped](https://github.com/google/protobuf/blob/master/src/google/protobuf/wrappers.proto)
  protobuf fields should be used for all non-string [scalar
  types](https://developers.google.com/protocol-buffers/docs/proto3#scalar), to
  support non-zero default values. While only some fields require wrapping, for
  consistency we prefer to have all non-string scalar fields wrapped.

## APIs

Unless otherwise stated, the APIs with the same names as v1 APIs have a similar role.

* [Cluster Discovery Service (CDS)](api/cds.proto).
* [Endpoint Discovery Service (EDS)](api/eds.proto). This has the same role as SDS in the [v1 API](https://envoyproxy.github.io/envoy/configuration/cluster_manager/sds_api.html),
  the new name better describes what the API does in practice. Advanced global load balancing capable of utilizing N-dimensional upstream metrics is now supported.
* [Health Discovery Service (HDS)](api/hds.proto). This new API supports efficient endpoint health discovery by the management server via the Envoy instances it manages. Individual Envoy instances
  will typically receive HDS instructions to health check a subset of all
  endpoints. The health check subset may not be a subset of the Envoy instance's
  EDS endpoints.
* [Listener Discovery Service (LDS)](api/lds.proto). This new API supports dynamic discovery of the listener configuration (which ports to bind to, TLS details, filter chains, etc.).
* [Rate Limit Service (RLS)](api/rls.proto)
* [Route Discovery Service (RDS)](api/rds.proto).
* [Secret Discovery Service (SDS)](api/sds.proto).

In addition to the above APIs, an aggregation API will be provided to allow for
fine grained control over the sequencing of API updates across discovery
services:

* [Aggregated Discovery Service (ADS)](api/discovery.proto). While fundamentally Envoy
  employs an eventual consistency model, ADS provides an opportunity to sequence
  API update pushes and ensure affinity of a single management server for an
  Envoy node for API updates. ADS allows one or more APIs to be delivered on a
  single gRPC bidi stream by the management server, and within an API to have all
  resources aggregated onto a single stream. Without this, some APIs such as RDS
  and EDS may require the management of multiple streams and connections to
  distinct management servers.

  ADS will allow for hitless updates of configuration by appropriate sequencing.
  For example, suppose *foo.com* was mappped to cluster *X*. We wish to change
  the mapping in the route table to point *foo.com* at cluster *Y*. In order to
  do this, a CDS/EDS update must first be delivered containing both clusters *X*
  and *Y*.

  Without ADS, the CDS/EDS/RDS streams may point at distinct management servers,
  or when on the same management server at distinct gRPC streams/connections
  that require coordination. The EDS resource requests may be split across two
  distinct streams, one for *X* and one for *Y*. ADS allows these to be
  coalesced to a single stream to a single management server, avoiding the need
  for distributed synchronization to correctly sequence the update. With ADS,
  the management server would deliver the CDS, EDS and then RDS updates on a
  single stream.

A protocol description for the xDS APIs is provided [here](XDS_PROTOCOL.md).

## Terminology

Some relevant [existing terminology](https://envoyproxy.github.io/envoy/intro/arch_overview/terminology.html) is
repeated below and some new v2 terms introduced.

* Cluster: A cluster is a group of logically similar endpoints that Envoy
  connects to. In v2, RDS routes points to clusters, CDS provides cluster configuration and
  Envoy discovers the cluster members via EDS.

* Downstream: A downstream host connects to Envoy, sends requests, and receives responses.

* Endpoint: An endpoint is an upstream host that is a member of one or more clusters. Endpoints are discovered via EDS.

* Listener: A listener is a named network location (e.g., port, unix domain socket, etc.) that can be connected to by downstream clients. Envoy exposes one or more listeners that downstream hosts connect to.

* Locality: A location where an Envoy instance or an endpoint runs. This includes
  region, zone and sub-zone identification.

* Management server: A logical server implementing the v2 Envoy APIs. This is not necessarily a single physical machine since it may be replicated/sharded and API serving for different xDS APIs may be implemented on different physical machines.

* Region: Geographic region where a zone is located.

* Sub-zone: Location within a zone where an Envoy instance or an endpoint runs.
  This allows for multiple load balancing targets within a zone.

* Upstream: An upstream host receives connections and requests from Envoy and returns responses.

* xDS: CDS/EDS/HDS/LDS/RLS/RDS/SDS APIs.

* Zone: Availability Zone (AZ) in AWS, Zone in GCP.
