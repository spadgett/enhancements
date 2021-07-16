---
title: hybrid-console
authors:
  - "@spadgett"
reviewers:
  - "@TheRealJon"
  - "@christianvogt"
  - "@jamestalton"
  - "@jhadvig"
  - "@showeimer"
  - "@simonpasquier"
approvers:
  - "@mdelder"
  - "@jwforres"
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: provisional
see-also:
  - "/enhancements/console/dynamic-plugins.md"
  - "/enhancements/console/user-settings.md"
---

# Hybrid Console

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

OpenShift console is currently a single cluster console. Today, users who work
in environments with multiple clusters must switch between consoles for each
cluster. We want to make OpenShift console multi-cluster aware. Users should be
able to visit one console running on the hub cluster and easily switch between
spoke clusters in the fleet without leaving that console or logging in again.

Additionally, we want to integrate the OpenShift and ACM consoles so that they
are a single console under a single URL. This hybrid console will have an "All
clusters" perspective that contains the ACM multi-cluster views. Selecting a
cluster from the cluster dropdown will drill down into the traditional
OpenShift console views for that cluster.

## Motivation

To provide a single web console that lets users interact will all clusters in
the fleet.

### Goals

* OpenShift and ACM consoles will function as a single console under one URL.
* The console will have a cluster selector that allows you to view resources for
  any cluster in the fleet.
* Users will be able to create and edit resources on spoke clusters from the
  console running on the hub.
* The console will support viewing metrics for workloads running on spoke
  clusters.
* Console will support single sign-on (SSO) to avoid logins every time a user
  switches clusters.

### Non-Goals

* For now, we do not plan to remove the OpenShift console from the release
  payload. Admins can disable the local console on spoke clusters using the
  existing `managementState` property in the console operator config.
* This enhancement does not cover the implementation details of the SSO operator.
* This enhancement does not cover authenticating with non-OpenShift clusters in
  the fleet.

## Proposal

### Feature Gate

To enable the multi-cluster console integration, administrators will need to
enable a `TechPreviewNoUpgrade` `MultiClusterConsole`
[feature gate](https://docs.openshift.com/container-platform/4.1/nodes/clusters/nodes-cluster-enabling-features.html).

### Sharing a Single URL

Initially, we plan to expose the OpenShift console under a single URL using a
shared ingress approach, which ACM uses today to serve different areas of its
UI under micro-services. When the `MultiClusterConsole` feature gate is enabled:

* The console operator will remove its normal route.
* The console operator will create an ingress in the `openshift-console`
  namespace using the custom ACM ingress controller.
* The ACM operator will use the console public hostname in its ingresses,
  discovered through the `console-public` config map in the
  `openshift-config-managed` namespace.
* OpenShift and ACM consoles will use the same session cookie,
  `openshift-session-token`, so that they can share the active session. 

See [CONSOLE-2888](https://issues.redhat.com/browse/CONSOLE-2888) for
additional background.

Long term, we are evaluating different approaches in addition to the shared
ingress, including [dynamic plugins](dynamic-plugins.md).

### Cluster Selector

A cluster selector will be added to the OpenShift console navigation bar. This
selector functions like the project bar in the console masthead except it
controls the cluster context rather than simply the project context.

When multi-cluster is enabled, the hub console is capable of connecting to the
API server on any of the spoke clusters in the fleet. API requests from the
frontend go to the console backend, which in turn proxies each request to the
public API server URL for the selected cluster. The console backend reads the
`Cluster` request header set by the frontend to determine the cluster to
route the request to. The various console hooks and utilities that make
requests to the API server will be updated to read the current cluster from the
Redux store and include the right `Cluster` header. This lets us implement
multi-cluster without needing to update every page and component that makes API
requests. The utilities the will automatically add the `Cluster` header include

* The `Firehose` component
* The `useK8sWatchResource` and `useK8sWatchResources` hooks
* The `k8s{Get,Create,Update,Patch,Delete}` functions

The console operator watches the `ManagedCluster` custom resources created by
ACM to determine what clusters are in the fleet. It reads the [API server
public URL and CA file](https://issues.redhat.com/browse/ACM-786) for each
`ManagedCluster`, and passes them to the console backend as configuration using
the existing `console-config` config map in the `openshift-console` namespace.
On startup, the console backend will set up a proxy to each of the clusters in
the fleet. When the `ManagedCluster` list changes, the console operator will
update the console configuration, triggering a new rollout of console.

### Authentication

The OpenShift console backend must support authentication challenges from each
of the clusters in the fleet. The backend discovers the OAuth endpoints for
each `ManagedCluster` through the `/.well-known/oauth-authorization-server`
endpoint and sets up separate authenticators for each cluster. The user's
access token for each spoke cluster is be stored in a separate cookie that
includes the cluster name, `openshift-session-token-<cluster-name>`.

A separate [SSO operator](https://issues.redhat.com/browse/ACM-779) is being
developed that will allow us to avoid a login prompt when switching clusters.
The user will still have a separate access token on each cluster that the
console backend tracks, however.

### Monitoring

In the initial dev preview implementation, OpenShift console will talk directly
to the monitoring endpoints exposed as routes on each of the spoke clusters.
The endpoints on each cluster are discovered and exposed by ACM
([ACM-876](https://issues.redhat.com/browse/ACM-876)). Additionally, ACM
must create a new route in the `openshift-monitoring` namespace to expose
the `tenancy` port on the `thanos-querier` service. This is required for normal
users to see workload metrics in OpenShift console.

*TODO*: Determine how ACM will expose this information.

Long term, we'd like to explore collecting workload metrics on the hub cluster.
Console will then be able to talk to the `thanos-querier` on the hub to get all
metrics without needing to make requests to the monitoring endpoints on each
spoke cluster.

### Console URLs

Console resource URLs will be updated to
[include the cluster name in the URL](https://issues.redhat.com/browse/CONSOLE-2831).
Currently, a resource path to the pods details page looks like

```
/k8s/ns/openshift-console/pods/console-fd5694f75-vdmn2
```

This will be updated to include the cluster in the path

```
/k8s/cluster/local-cluster/ns/openshift-console/pods/console-fd5694f75-vdmn2
```

When the console sees a URL without the cluster segment in its path, it will
assume the current cluster context and add a React Router redirect to update
the URL to include the cluster name. This allows any links in the console that
do not get updated to continue to work.

### User Settings

[User Settings](user-settings.md) are stored as JSON in config maps in the
`openshift-console-user-settings` namespace. In a multi-cluster environment, we
will continue to use these config maps and always store the user settings on
the hub. Most user settings are not specific a particular cluster.

For settings that are cluster-specific, they will need to be set by cluster
name under a `cluster-settings` stanza in the settings JSON. For example,

```json
{
  "cluster-settings": {
    "local-cluster": {
      "console.lastNamespace": "my-namespace"
    }
  }
}
```

### User Stories

See [Hybrid Console personas, use cases, and principles](https://docs.google.com/document/d/1-UodIqKYIsOaIYVNYNqpGWu7PnRvtml_nn1EpvnsSVo/edit#).

### Implementation Details/Notes/Constraints [optional]

* This proposal assumes the console backend running on the hub cluster will be
  able to make network requests to public endpoints like the API server on each
  spoke cluster.
* Dynamic plugins will only be loaded from the hub cluster. If a dynamic plugin
  is added by an operator on a spoke, it will not be loaded by the
  multi-cluster console on the hub. We'll need additional exploration to see if
  loading dynamic plugins from a spoke can be supported.
* The console service account will not have special permissions on the spoke
  clusters. This means a few features like cloud shell that require special
  access will need to be disabled for multi-cluster.

### Risks and Mitigations

This is a significant architectural change to OpenShift console, and the
enhancement proposal makes assumptions about whether the console backend on a
hub cluster can reach public endpoints on various clusters in the fleet. Some
capabilities like cloud shell require special privileges from the console
service account, which won't work in a multi-cluster environment.

We are mitigating these risks by initially offering this as a dev preview as we
collect feedback and work through technical limitations.

## Design Details

### Open Questions [optional]

1. Will the console running on the hub always be able to talk to public API
   endpoints on the spoke cluster?
1. What is our long-term approach to integrating OpenShift and ACM consoles:
   shared ingress or dynamic plugins?
1. What is our long-term approach to multi-cluster metrics? Will workload
   metrics from all clusters the fleet be available from Thanos on the hub
   cluster?
1. Do we need to be able to load dynamic plugins from spoke clusters?

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

We will initially deliver this as a dev preview.

#### Dev Preview -> Tech Preview

- Alignment on long-term ACM/OCP console integration approach (shared ingress vs. dynamic plugins)
- Alignment on long-term monitoring approach (fetching metrics from hub vs. spokes)
- Ability to utilize the enhancement end-to-end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

#### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default when ACM is installed
- Ability to connect to non-OpenShift clusters in the fleet
- End-to-end tests

#### Removing a deprecated feature

N/A

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to this should be
  identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

### Version Skew Strategy

The OpenShift console running on the hub cluster needs to be able to talk to
clusters at different OpenShift versions. It's not feasible or desirable to
require all clusters in the fleet to run at the same version. Version skew is
handled through API discovery and feature detection which console already does
today. Console re-runs discovery when the selected cluster changes and only
shows navigation items for resources that exist on the current cluster. Console
will either need to use the lowest common denominator when selecting API
versions for resource or will need to handle a range of versions if newer
features are required.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
