# KEP-NNNN: Unix Domain Socket Support for Kubernetes Apiserver and Client-go

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Enhanced Security with Authenticating Proxies](#enhanced-security-with-authenticating-proxies)
    - [Simplified and More Isolated Control Plane Deployments](#simplified-and-more-isolated-control-plane-deployments)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
    - [High-Level Design](#high-level-design)
    - [The https+unix:// scheme.](#the-httpsunix-scheme)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
    - [Prerequisite testing updates](#prerequisite-testing-updates)
    - [Unit tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in
  [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and
  SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806)
    must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
    within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for
  publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to
  mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP proposes adding support for Unix Domain Sockets (UDS) to the Kubernetes
API server and `client-go`. This enables secure, local communication between
control plane components without relying on TCP/IP networking, simplifying
deployments and enhancing security by reducing the network attack surface.

## Motivation

Currently, communication between Kubernetes components (like the API server and
`kubelet`) and clients (like `kubectl`) relies on TCP/IP sockets, even when
components are running on the same host. TCP connections might expose sensitive
internal ports if not properly firewalled, and managing these connections can be
complex in certain network environments.

Support for UNIX domain sockets in kubernetes api clients and servers can help
simplify control plane deployments, removing the need to manage IP addresses and
firewalls between control plane components.

### Goals

*   Enable the Kubernetes API server to listen on Unix Domain Sockets (UDS) in
    addition to, or as an alternative to, TCP/IP sockets.
*   Allow `client-go` to connect to the api servers using UDS via the
    `https+unix://` scheme for secure transport.

### Non-Goals

*   Support for non-secure (HTTP over UDS) communication for the main API
    server.
*   Replacement of TCP/IP for all Kubernetes communication; UDS is intended for
    local host-level optimizations.

## Proposal

### User Stories

#### Enhanced Security with Authenticating Proxies

API servers can be fronted by authenticating proxies (e.g., using requestheader
authentication). When these proxies communicate with the API server via Unix
Domain Sockets, the API server itself does not need to be directly exposed or
secured against bypass over the external IP network, as the proxy handles
inbound traffic and authentication.

#### Simplified and More Isolated Control Plane Deployments

By removing the necessity for IP networking between control plane components,
UDS can simplify deployment configurations and enable better isolation and
sandboxing.

### Notes/Constraints/Caveats

#### High-Level Design

Support for unix domain sockets will be added to the rest client
(`k8s.io/client-go`), and to the generic apiserver (`k8s.io/apiserver`).

- Implement support for specifying unix socket urls in rest.Config.
  - References to UDS backed servers are distinguished by a separate URI scheme,
    see below.
  - As a security measure, we propse the following Kubernetes specific
    restrictions on UDS support:
    - Only file path socket addresses are allowed, as abstract sockets are not
      subject to file permission checks.
    - Only 'clean', absolute paths are supported. For example,
      "./apiserver.sock", "apiserver.sock", or "../config/apiserver.sock" are
      now allowed.
  - Since socket file names are local to the client environment, the exact file
    path can vary independently of the server identity. To ensure proper TLS
    validation without requiring clusters to be signed for the exact socket
    path, `rest.TLSClientConfig.ServerName` is required to be set when UDS
    server is specified.

- Implement support for apiserver listening on unix domain socket URLs.
  - The generic apiserver can be configured with a set of additional socket
    addresses the apiserver listens on. To help with future flexibility, we
    format these as "listener URLs" (`--listen-urls="https://...,
    https+unix://"`) instead of separate address and port flags.
  - The following changes would be made to `options.SecureServingOptions`.
    - `BindAddress` and `BindPort` can be left unset to disable listening on
      TCP addresses.
    - The `ExternalAddress` field is required if no TCP listener is configured,
      as there would be no way to compute the "external address" of a unix
      socket.

#### The https+unix:// scheme.

For clients and servers, we propose the adoption of the following URL scheme
`https+unix://SOCK_ADDR/PATH][?QUERY][#FRAGMENT]` where `SOCK_ADDR` is a unix
socket address. Since socket addresses can contain path separator, literal `/`
must be percent encoded.

For example, `/var/run/docker.sock` and path `/v1.40/containers/json` is encoded
as `https+unix://%2Fvar%2Frun%2Fdocker.sock/v1.40/containers/json`. Likewise, in
case we want to extend support to abstract namespace sockets as well, The `@`
symbol must be percent encoded as well.

Note on the golang URL parser: The golang stdlib parser fails to parse these
URLs, as it does not allow control characters to appear in the host segment to
protect against request smuggling attacks. We will need to slightly relax this
protection to allow `/` and optionally `@` as percent-encoded characters in the
host segment, only in the context of https+unix scheme urls.

### Risks and Mitigations

*   **File Permissions:** UDS relies on file system permissions. If the socket
    file is accessible to unauthorized users, it could lead to privilege
    escalation.
    *   *Mitigation:* The API server will ensure restricted permissions on the
        created socket file (e.g., `0600` or `0660` with a specific group).
*   **Cleanup:** Orphaned socket files might prevent the API server from
    restarting.
    *   *Mitigation:* The API server will implement proper cleanup logic on
        shutdown and handle pre-existing socket files on startup.

## Design Details

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.
-->

[ ] I/we understand the owners of the involved components may require updates
to existing tests to make this code solid enough prior to committing the
changes necessary to implement this enhancement.

#### Prerequisite testing updates

None identified at this stage.

#### Unit tests

- `k8s.io/apiserver/pkg/server/options`: Test `SecureServingOptions` with UDS
  listener URLs.
- `k8s.io/client-go/rest`: Test `Config` with `https+unix://` URLs.
- `k8s.io/client-go/transport`: Test UDS dialer integration.
- `k8s.io/apimachinery/pkg/util/net`: Test UDS path validation and cleaning.

#### Integration tests

- Start a `generic-apiserver` instance listening on a UDS.
- Verify `client-go` can perform basic CRUD operations over the UDS.
- Verify TLS certificate validation works correctly with `ServerName` override.

#### e2e tests

- Verify `kube-apiserver` can be started with `--bind-unix-address`.
- Verify `kube-controller-manager` and `kube-scheduler` can communicate with
  `kube-apiserver` over UDS when co-located.

### Graduation Criteria

#### Alpha

- Feature implemented behind the `UnixSocketAPIClient` and
  `UnixSocketAPIServer` feature gates.
- Support for `https+unix://` scheme in `client-go`.
- `kube-apiserver` support for listening on UDS.
- Initial unit and integration tests completed.

#### Beta

- `UnixSocketAPIClient` and `UnixSocketAPIServer` feature gates enabled by
  default.
- Comprehensive E2E tests for co-located control plane components.
- Documentation on recommended UDS deployment patterns and security best
  practices.

#### GA

- `UnixSocketAPIClient` and `UnixSocketAPIServer` feature gates locked to true.
- No major regressions or issues reported during the Beta phase.

### Upgrade / Downgrade Strategy

UDS support is opt-in and controlled by the `UnixSocketAPIClient` (client-side)
and `UnixSocketAPIServer` (server-side) feature gates.

- **Upgrade:** Enabling the feature gates allows the use of UDS-related flags
  and URL schemes. Existing TCP configurations are unaffected.
- **Downgrade:** Disabling the feature gates will cause components configured
  to use UDS to fail to start or fall back to TCP if alternative configurations
  are available.

### Version Skew Strategy

UDS communication is inherently local to a host.
- A client attempting to use UDS against a server that does not support it
  (e.g., an older version) will fail with a connection error.
- Standard Kubernetes version skew policies apply to the control plane
  components themselves.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [ ] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `UnixSocketAPIClient` (client-side),
    `UnixSocketAPIServer` (server-side)
  - Components depending on the feature gate: `kube-apiserver`,
    `kube-controller-manager`, `kube-scheduler`, `client-go` based tools.
- [X] Other
  - Describe the mechanism: In addition to the feature gates, explicit
    configuration (flags or config files) is required to use UDS.
  - Will enabling / disabling the feature require downtime of the control
    plane? Yes, `kube-apiserver` restart is required for the server side, and
    client components like `kube-controller-manager` require restart to pick up
    the gate/config.
  - Will enabling / disabling the feature require downtime or reprovisioning of
    a node? No.

###### Does enabling the feature change any default behavior?

No. It is opt-in.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. By disabling the `UnixSocketAPIClient` feature gate and reverting flags to
use TCP addresses.

###### What happens if we reenable the feature if it was previously rolled back?

The components will attempt to bind to the specified UDS path again. If the
previous socket file exists, the server will attempt to clean it up before
binding.

###### Are there any tests for feature enablement/disablement?

Unit tests will verify that UDS configuration is ignored when the feature gate
is disabled.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

A rollout can fail if the specified UDS path is not writable by the
`kube-apiserver` process or if the path is invalid. It does not impact already
running workloads on other nodes, but it can impact control plane availability
on the local node.

###### What specific metrics should inform a rollback?

- `apiserver_request_total` (check for drop in local requests)
- `apiserver_request_duration_seconds` (check for latency spikes)
- Process restart loops for `kube-apiserver`.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

To be tested during the Alpha phase.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

By inspecting the `kube-apiserver` command-line arguments and checking for the
presence of the socket file on the host.

###### How can someone using this feature know that it is working for their instance?

- [ ] Events
- [ ] API .status
- [X] Other
  - Details: `kubectl cluster-info` or checking connectivity via the UDS path.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Same as existing API server SLOs for latency and availability.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [X] Metrics
  - Metric name: `apiserver_request_total` (filtered by transport/listener)
  - Components exposing the metric: `kube-apiserver`
- [ ] Other

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

Metrics specifically tracking UDS connection counts and errors might be useful.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. It only depends on the local file system.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No, UDS is expected to be slightly faster or equivalent to local TCP.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

UDS uses inodes. Exhaustion is unlikely given the small number of sockets
involved in control plane communication.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

The feature is part of the API server's listening mechanism. If etcd is
unavailable, the API server will fail to serve requests as usual, regardless of
whether it's using UDS or TCP.

###### What are other known failure modes?

- **Permission Denied:** The process doesn't have permissions to create or
  listen on the socket.
  - *Detection:* Error logs in `kube-apiserver`.
  - *Mitigations:* Correct file system permissions.
- **Path Too Long:** Unix socket paths have length limits (usually ~108 chars).
  - *Detection:* Error logs on startup.
  - *Mitigations:* Use shorter absolute paths.

###### What steps should be taken if SLOs are not being met to determine the problem?

Check `kube-apiserver` logs for UDS-related errors. Verify socket file
existence and permissions.

## Implementation History

- 2026-05-09: Initial KEP draft (Provisional)

## Drawbacks

- Increased complexity in API server and `client-go` transport logic.
- Potential for platform-specific bugs (UDS behavior can vary slightly between
  OSs, though Go abstracts much of this).

## Alternatives

- **Transparent Proxies:** Using `iptables` or `nftables` to redirect local
  traffic. While effective, it adds networking complexity.
- **Shared Memory:** Much more complex to implement for standard HTTP/REST
  communication.

## Infrastructure Needed

None.
