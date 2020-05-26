# Structured Logging

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Log message structure](#log-message-structure)
    - [New klog methods](#new-klog-methods)
  - [References to Kubernetes objects](#references-to-kubernetes-objects)
    - [Object reference format in Logs](#object-reference-format-in-logs)
    - [Helper functions for log format](#helper-functions-for-log-format)
  - [Selecting most important logs](#selecting-most-important-logs)
  - [JSON output format](#json-output-format)
    - [Reserved JSON keys](#reserved-json-keys)
    - [Serialization strategy](#serialization-strategy)
  - [Logging configuration](#logging-configuration)
  - [Migration / Graduation Criteria](#migration--graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Performance](#performance)
    - [Logger implementation performance](#logger-implementation-performance)
    - [Log volume increase analysis](#log-volume-increase-analysis)
  - [Future work](#future-work)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Migration being abandoned halfway](#migration-being-abandoned-halfway)
    - [Huge increase of log volume](#huge-increase-of-log-volume)
- [Design Details](#design-details)
  - [Proposed list of log messages to change](#proposed-list-of-log-messages-to-change)
  - [Example of migrating klog call](#example-of-migrating-klog-call)
- [Alternatives](#alternatives)
    - [Just write guideline and update log messages](#just-write-guideline-and-update-log-messages)
    - [Replace klog with some other structured logging library](#replace-klog-with-some-other-structured-logging-library)
    - [Use glogr instead of proposed message structure](#use-glogr-instead-of-proposed-message-structure)
- [Code organisation](#code-organisation)
- [Test Plan](#test-plan)
- [Release Signoff Checklist](#release-signoff-checklist)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Summary

This KEP proposes to define standard structure for Kubernetes log messages, add methods to klog to enforce this structure, add ability to configure Kubernetes components to produce logs in JSON format and initiate migration to structured logging.

## Motivation

Current logging in the Kubernetes control plane doesn’t guarantee any uniform structure for log messages and references to Kubernetes objects in those logs. This makes parsing, processing, storing, querying and analyzing logs hard and forces administrators and developers to rely on ad-hoc solutions in most cases based on some regular expressions. Due to those problems any analytical solution based on those logs is hard to implement and maintain.

### Goals

* Make most common logs more queryable by standardizing log message and references to Kubernetes objects (Pods, Nodes etc.)
* Enforce log structure by introduction of new klog methods that could be used to generate structured logs.
* Propose reasonable scope of migration:
  * Executable within SIG-instrumentation resources
  * Minimize review burden on other SIGs
  * Leave logging in acceptable state if effort fails
* Simplify ingestion of logs into third party logging solutions by adding an option to output logs in the JSON format


### Non-Goals

* We are **not** replacing currently used logging library (klog) or the way in which it is used
* We are **not** preserving full compatibility of logging output
* We are **not** proposing structuring all logs in Kubernetes

## Proposal

This KEP proposes introducing a standardized log message structure. To enforce a new log message structure we are proposing adding new methods to the klog library: `InfoS`, `ErrorS` which will provide a structured interface for building log messages and creating new helper methods which will provide consistent identification of Kubernetes objects in logs.

With the new methods we would like also to migrate to klog v2 and introduce a new JSON logging output format to Kubernetes components that would be an alternative to the current text format and which will make querying and processing Kubernetes logs even simpler.

Taking into account the size of Kubernetes repository we will not tackle the problem of full migration to new klog methods but instead focus on improving the querability of most common logs. We will be targeting reaching a high percentage of logs in a new format generated by real cluster.

### Log message structure

We would like for the Kubernetes community to settle on one preferred log message structure, that will be enforced by new klog methods. Proposed structure should:
* Separate log message from its arguments
* Treat log arguments as key-value pairs
* Be easily parsable and queryable
* Have a specific guidance on log message and its arguments

For this purpose we suggest to use following log message structure:
```
<message> <key1>=<value1> <key2>=<value2> ...
```

where
* message is formatted using `%q` fmt logic
* keys are formatted using `%s` fmt logic
* values are formatted using `%q` fmt logic, except for number and boolean
* keys are separated from values with a "=” sign
* message and key-values are separated with a single space

#### New klog methods

To enforce a new log message structure we want to introduce new methods to klog library which will provide a more structured interface for formatting log messages compared to current methods based on fmt format strings.
For each format method (`Infof`, `Errorf`) we will add matching structured method (`InfoS`, `ErrorS`). Interface for this methods will model  https://github.com/go-logr/logr (suggested by @thockin)
Each of those methods will accept log messages as a first argument and a list of key-values pairs as a variadic second argument.
This approach allows incremental adoption of structured logging without converting ALL of kubernetes to a new API at one time.

Declaration:

```go
package klog

func InfoS(msg string, keysAndValues ...interface{})
func ErrorS(err error, msg string, keysAndValues ...interface{})
```

Example:

```go
klog.InfoS("Pod status updated", "pod", "kubedns", "status", "ready")
```

That would result in log
```
I1025 00:15:15.525108       1 controller_utils.go:116] "Pod status updated" pod="kubedns" status="ready"
```

And

```go
klog.ErrorS(err, "Failed to update pod status")
```

That would result in log
```
E1025 00:15:15.525108       1 controller_utils.go:114] "Failed to update pod status" err="timeout"
```

### References to Kubernetes objects

Kubernetes API first approach has given great importance to Kubernetes objects.
In this proposal we would like to make sure that transfer their importance to logs, making sure that references to objects are easy to query and correlate.

When browsing through kubernetes logs you can find a variety of ways how objects are represented, sometimes by only name, sometimes as pair name and namespace and sometimes as uid. We would like for community to settle on one canonical representation of object.

Object reference could be stored under different keys e.g. `pod=kubedns`. For now we don't want to tackle they keys and preserve freedom for caller to choose what's best.
Enforcing a consistent key space would require defining a schema that would need to handle variety of situations and contexts e.g. `podOld` and `podNew`.
We expect this to be tackled in future.

To improve how kubernetes objects are referenced we would like to propose:
* One preferred format for representing references in logs
* Helper functions for building this format

#### Object reference format in Logs

As first step we propose to go with simple and used format. This will make migration easy and natural.
Unifying format will need long time to produce measurable benefits, so we want start the migration early.

Proposed references that will be used in text format:
* Namespaced objects will be represented by joining their name and namespace with `/` character. Pattern: `<namespace>/<name>`, example for pod: `kube-system/kubedns`
* Non-namespaced objects will be represented by their name. Pattern: `<name>`, example for node: `cluster1-vm-72x33b8p-34jz`

Object UID in logs is much less popular, but still useful in some situation. In situation caller should consider adding a separate field with object UID.

To initiate migration into this format we would like to update logging user guide and introduce helper functions.

#### Helper functions for log format

Logging is needed in variety of situations in object lifecycle: before it was created, during update, after deletion.
Depending on lifecycle and code location access to whole objects is not always possible/easy.
Migration to preferred object reference format should not require major code rewrites, that's why we would like to propose two functions that are selected based on if Kubernetes object is easily accessible.
We expect that in future when passing object meta becomes the standard logging api could be more integrated.

```go
func KObj(obj ObjectMeta) ObjectRef
func KRef(namespace, name string) ObjectRef

type ObjectRef struct {
  Name      string `json:"name"`
  Namespace string `json:"namespace,omitempty"`
}
```

Example:
```go
pod := corev1.Pod{Name: "kubedns", Namespace: "kube-system", ...}
klog.InfoS("Pod status updated", "pod", klog.KObj(pod), "status", "ready")
```

And

```go
klog.InfoS("Pod status updated", "pod", klog.KRef("kube-system", "kubedns"), "status", "ready")
```

Will result in log
```
I1025 00:15:15.525108       1 controller_utils.go:116] "Pod status updated" pod="kube-system/kubedns" status="ready"
```

For not namespaced objects we propose to use:

```go
klog.InfoS("Node unavailable", "node", klog.KRef("", "nodepool-1"))
```

### Selecting most important logs

As migration to new message structure will be done manually we will be focusing on logs that have the greatest impact for log querying and processing. We will be focusing on log messages Proposed plan of measurements:

* Using log data from single run of kubernetes [gce-master-scale-performance](https://k8s-testgrid.appspot.com/sig-scalability-gce#gce-master-scale-performance) tests
* Based on logs generated on master node
* Aggregate from components:
  * kube-controller-manager
  * kube-scheduler
  * kube-apiserver
  * kubelet
* Only for log lines generated by klog (e.g. prefix `I0129 03:26:19.857798       1 garbagecollector.go:517]`)
* Identified by code line that generated log (e.g. `garbagecollector.go:517`)

We assume that exact details of taking measurements can be improved, but overall methodology is solid. With those criteria we concluded that covering 99.9% of logs on master node are generated by 22 log messages. Exact list of those logs is provided in the detail design section.

### JSON output format

Introduction of new methods to klog library will make identification of different components of the log message much easier.
With klog v2 we can take further advantage of this fact and add an option to produce structured logs in JSON format.

Some pros of using JSON:
* Broadly adopted by logging libraries with very efficient implementations (zap, zerolog).
* Out of the box support by many logging backends (Elasticsearch, Stackdriver, BigQuery, Splunk)
* Easily parsable and transformable
* Existing tools for ad-hoc analysis (jq)

We would like to discuss some details of format definition that will guide implementation. Definition will define:
* Reserved JSON keys that will have special meaning
* Serialization strategy for different types

#### Reserved JSON keys

We would like to reserve some JSON keys to preserve same meaning for all logs within kubernetes.
Those keys will have special meaning and should not be used as arguments.
We would like to keep this set of keys as minimal to reduce possible changes, before we introduce a way to version the log schema.

For simplicity we will use flat key structure without namespacing.
We will update Kubernetes logging guide to discourage usage of those keys as log arguments.
We will decide if additional static analysis or runtime validation is needed if such problems arise.

Special JSON keys:
* `ts` - timestamp as Unix time (required, `float`)
* `v` - verbosity (required, `int`, default 0)
* `err` - error string (optional, `string`)
* `msg` - message (required, `string`)

#### Serialization strategy

We would like to describe the serialization as it will directly impact results of most logs as it will work as default. What do we want from serialization, it should:
* Have predictable results and performance
* Not diverge too much from `text` format (e.g. change formatting, add more information)

Proposed Serialization of types (we will expand list of special cases if needed):
* Go primitive types (e.g. `int`, `float`, `time.Duration`) - default format `%v`
* `time.Time` - Unix time format, as `%v` would result in format meant for [debugging](https://golang.org/pkg/time/#Time.String)
* arbitrary structure (`http.Request`) - shallow reflection into JSON objects. Object will consist of non-pointer public fields.

For arbitrary structures we propose to use a shallow strategy as it is:
* Analogous to `%v` currently used in format strings and in proposed text format (pointers are serialized as numbers)
* Has much more predictable performance (skips deeply nested structures and pointer cycles)

We will consider adding a way to override serialization strategy to deep, depending of feedback. For now caller can decide on logging particular fields as separate keys.

Example:

```go
klog.Infof("Updated pod %s status to ready", pod.name)
```
That would result in log (pretty printed)

```json
{
   "ts": 1580306777.04728,
   "v": 4,
   "msg": "Updated pod kubedns status to ready"
}
```

And

```go
err := fmt.Errorf("timeout")
klog.ErrorS(err, "Failed to update pod status")
```

That would result in log (pretty printed)

```json
{
   "ts": 1580306777.04728,
   "v": 4,
   "err": "timeout",
   "msg": "Failed to update pod status"
}
```

And

```go
pod := corev1.Pod{Name: "kubedns", Namespace: "kube-system", ...}
klog.InfoS("Pod status updated", "pod", klog.KObj(pod), "status", "ready")
```

That would result in log (pretty printed)
```json
{
   "ts": 1580306777.04728,
   "v": 4,
   "msg": "Pod status updated",
   "pod":{
      "name": "nginx-1",
      "namespace": "default"
   },
   "status": "ready"
}
```

And

```go
type Request struct {
  Method  string
  Timeout int
  secret  string
  Con     *Connection
}
req := Request{Method: "GET", Timeout: 30, secret: "pony"}
klog.InfoS("Request finished", "request", Request)
```

That would result in log (pretty printed)
```json
{
   "ts": 1580306777.04728,
   "v": 4,
   "msg": "Request finished",
   "request":{
      "Method": "GET",
      "Timeout": 30
   }
}
```

### Logging configuration

To allow selection of the logging output format we would like to introduce a new logging configuration shared by all kubernetes components.

`LoggingConfig` structure should be implemented as part of `k8s.io/component-base` options and include a common set of flags for logger initialization.

`--logging-format` flag should allow you to pick between logging output formats. Setting this flag will select a particular logger implementation.

Proposed flag `--logging-format` values:
* `text` for text-based logging format (default)
* `json` for new JSON format

Additionally we propose to expose to allow for customization of logging by providing custom implementation.
Custom logger format can be introduced by implementing [https://github.com/go-logr/logr](https://github.com/go-logr/logr) interface.
Registering custom implementation within `LoggingConfig` will allow to pick it by via `--logging-format` flag.
Klog interface was selected as it is already supported by klog.v2 with `SetLogger` method.

### Migration / Graduation Criteria

#### Alpha

Introduce structured logging and JSON format:
* Most important logs are migrated to structured methods.
* Flag for selecting logging format is implemented
* JSON format is implemented

#### Beta

Adding guarding against regression and update user guideline:
* Static analysis protects important log lines from reverting to string formating
* [Kubernetes Logging convention](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/logging.md) document is updated

#### GA

Logging formats become an API:
* Feedback about logging methods and log formats is collected and addressed
* Format string methods are deprecated
* Static analysis protects against creating new string format calls
* Logging output formats fall under [Kubernetes deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)

### Performance

Proposed changes could impact k8s performance in two ways:
* By introducing degrading in klog implementation
* By largely increasing log volume

#### Logger implementation performance

We have benchmarked proof of concept implementation of new logging format. Results measured on 30s benchmark for passing 2 arguments to format function.

|logger                 |time [ns/op]|bytes[B/op]|allocations[alloc/op]|
|-----------------------|------------|-----------|---------------------|
|Text Infof             |2252        |248        |3                    |
|Text InfoS             |2455        |280        |3                    |
|JSON Infof             |1406        |19         |1                    |
|JSON InfoS             |319         |67         |1                    |

InfoS implementation for text is 9% slower than Infof. This increase should not have a big impact on overall Kubernetes performance as logging takes less than 2% of overall CPU usage.
For json format we used [zap](https://github.com/uber-go/zap).

#### Log volume increase analysis

Changes in format presented in this KEP will result in adding additional metadata to logs, potentially largely increasing log volume.
Unexpected log volume increase can lead to performance problems due exhausting disk IO or halting whole process when logs cannot be written on full disk.
To avoid breaking existing poduction clusters we would like to target of around 10% of log volume increase between kubernetes versions (proposed by @wojtekt).

During log volume analysis we would like to mainly focus on Kubernetes apiserver HTTP access logs. Those logs will have biggest impact
* They are responsible for 85% of log volume generated in cluster
* They are expected to get at a lot of additional metadata as they take 8 arguments

HTTP access log in Kubernetes 1.17
```go
klog.Infof("%s %s: (%v) %v%v%v [%s %s]", rl.req.Method, rl.req.RequestURI, latency, rl.status, rl.statusStack, rl.addedInfo, rl.req.UserAgent(), rl.req.RemoteAddr)
```

Resulting log line is 206 characters long
```
I1025 00:15:15.525108       1 httplog.go:79] GET /api/v1/namespaces/kube-system/pods/metrics-server-v0.3.1-57c75779f-9p8wg: (1.512ms) 200 [pod_nanny/v0.0.0 (linux/amd64) kubernetes/$Format 10.56.1.19:51756]
```

HTTP access log in Kubernetes 1.18 (after [changes by @lavalamp](https://github.com/kubernetes/kubernetes/pull/87203))
```go
klog.Infof("verb=%q URI=%q latency=%v resp=%v UserAgent=%q srcIP=%q: %v%v", rl.req.Method, rl.req.RequestURI, latency, rl.status, rl.req.UserAgent(), rl.req.RemoteAddr, rl.statusStack, rl.addedInfo)
```

Resulting log line is 248 characters long. Additional 42 characters has resulted in increased log volume by 17% (average size of http log line in 1.17 is 243 characters).
```
I0129 03:30:57.673664       1 httplog.go:90] verb="GET" URI="/api/v1/namespaces/kube-system/pods/metrics-server-v0.3.1-57c75779f-9p8wg" latency=1.512ms resp=200 UserAgent="kubelet/v1.18.0 (linux/amd64) kubernetes/15c3f1b" srcIP="10.56.1.19:51756":
```

For migration to structured api we would like to propose to remove `statusStack` and `addedInfo` arguments. Our reasons:
* Log format already neglects to put metadata for them, breaking the convention.
* We didn't find any case of those fields being non empty in [gce-master-scale-performance](https://k8s-testgrid.appspot.com/sig-scalability-gce#gce-master-scale-performance)

Proposed log format for http access:
```go
klog.InfoS("HTTP", "verb", rl.req.Method, "URI", rl.req.RequestURI, "latency", latency, "resp", rl.status, "UserAgent", rl.req.UserAgent(), "srcIP", rl.req.RemoteAddr)
```

Resulting log line is 258 characters long. Additional 10 characters would result in increased log volume by 4% (average size of http log line in 1.18 is 285 characters).
```
I0227 11:52:25.212310  138043 experiment.go:38] "HTTP" verb="GET" URI="/api/v1/namespaces/kube-system/pods/metrics-server-v0.3.1-57c75779f-9p8wg" latency="1.512ms" resp=200 UserAgent="pod_nanny/v0.0.0 (linux/amd64) kubernetes/$Format srcIP="10.56.1.19:51756"
```

Concluding, migration to new logging format will increase log volume by around 4%, meaning it is acceptable from performance standpoint.

### Future work

List of ideas that were put on the original proposal that we would like to tackle in future, when structured logging fundamentals are in place:
* Replacement of Klog for a better logging solutions (faster, lighter on dependencies)
* Log schema to ensure more uniform metadata
* Context-ful logging to support shared metadata with tracing

### Risks and Mitigations

#### Migration being abandoned halfway

Kubernetes is a huge project spread across multiple repositories with thousands of logging calls. As not all of those calls can be migrated to new structured methods at once.

To mitigate the impact of this risk:

* We will start migration from the most common log lines to maximize the benefits from this manual work.
* We will submit changes in separate commits to increase the chance of each of the changes being merged quickly.


#### Huge increase of log volume

This effort should result in reducing costs of log analysis in production setups. Introducing structured logs will allow for creating better indices and reducing the size of data needed for analysis. From a log throughput perspective there will be an increase of log size causing a pressure of log ingestion (disk, logging agents, logging API).

To reduce the potential impact we will try to minimize the number of key-values logged when performing manual migration and we will try to use short keys.

If log volume increase cannot be avoided we will include information about large increase in Kubernetes release change log so users can prepare additional capacity or change log verbosity.

## Design Details

### Proposed list of log messages to change

Log messages ordered by their share in log line count.

|nr |line                        |Log line share|Coverage|
|---|----------------------------|----------|--------|
|1  |[httplog.go:90](https://github.com/kubernetes/kubernetes/blob/15c3f1b11/staging/src/k8s.io/apiserver/pkg/server/httplog/httplog.go#L90)  |84.44%    |84.44%  |
|2  |[get.go:251](https://github.com/kubernetes/kubernetes/blob/15c3f1b11/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/get.go#L251)    |12.48%    |96.92%  |
|3  |[watcher.go:363](https://github.com/kubernetes/kubernetes/blob/15c3f1b11/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/get.go#L251)|0.83%     |97.76%  |
|4  |[watcher.go:256](https://github.com/kubernetes/kubernetes/blob/15c3f1b11/staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go#L363) |0.52%     |98.28%  |
|5  |[event.go:278](https://github.com/kubernetes/kubernetes/blob/15c3f1b11/staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go#L256)   |0.42%     |98.70%  |
|6  |[scheduler.go:715](https://github.com/kubernetes/kubernetes/blob/15c3f1b11/staging/src/k8s.io/client-go/tools/record/event.go#L278)      |0.33%     |99.03%  |
|7  |garbagecollector.go:404     |0.32%     |99.35%  |
|8  |garbagecollector.go:517     |0.32%     |99.67%  |
|9  |node_authorizer.go:197      |0.03%     |99.70%  |
|10 |controller_utils.go:603     |0.03%     |99.73%  |
|11 |replica_set.go:561          |0.03%     |99.76%  |
|12 |trace.go:116                |0.02%     |99.78%  |
|13 |deployment_controller.go:484|0.02%     |99.80%  |
|14 |deployment_controller.go:575|0.02%     |99.82%  |
|15 |endpoints_controller.go:340 |0.02%     |99.84%  |
|16 |cloud_cidr_allocator.go:233 |0.02%     |99.86%  |
|17 |ttl_controller.go:271       |0.01%     |99.87%  |
|18 |replica_set.go:597          |0.01%     |99.88%  |
|19 |replica_set.go:225          |0.01%     |99.88%  |
|20 |scheduler.go:604            |0.01%     |99.89%  |
|21 |factory.go:431              |0.01%     |99.90%  |
|22 |kubelet_getters.go:173      |0.01%     |99.90%  |


### Example of migrating klog call

Before migration:
```go
klog.Infof("%s %s: (%v) %v%v%v [%s %s]", rl.req.Method, rl.req.RequestURI, latency, rl.status, rl.statusStack, rl.addedInfo, rl.req.UserAgent(), rl.req.RemoteAddr)
```

After migration:
```go
klog.InfoS("Access", "method", rl.req.Method, "uri", rl.req.RequestURI, "latency", latency, "status", rl.status, "agent", rl.req.UserAgent(), "addr", rl.req.RemoteAddr)
```

## Alternatives

#### Just write guideline and update log messages

* Doesn’t guarantee uniformity of log messages, is error prone and vulnerable to drifting
* Doesn’t improve log ingestion into external systems
* Doesn’t provide any value when switching to JSON output format

#### Replace klog with some other structured logging library

* Requires replacing all calls to klog with calls to a new logging library which makes this change hard to implement and merge into source repository.
* Large percentage of the logs cannot be migrated to structured logging completely automatically, therefore full migration would require a significant amount of manual work and increase the risk of this change being abandoned.
* Makes rollback of the change very hard to implement.

#### Use glogr instead of proposed message structure

* Glogr uses JSON serialization for both keys and values
* Significantly increases log volume
* Lack of support of warning and fatal severity levels

## Code organisation

As a result of this effort we are expecting changes in:
* `k8s.io/kubernetes` - Migration of log lines
* `k8s.io/component-base/logging` - Adding logging configuration
* `k8s.io/klog` - Implementing new structured logging methods

## Test Plan

Aside ot standard unit tests we propose to add detailed test suite ensuring no unexpected changes in out of both logging formats implementation to ensure their future stability.

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [x] (R) Graduation criteria is in place
- [x] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

## Production Readiness Review Questionnaire

For this questionnaire we will consider feature of switching between text (original) and json (new) log formats.
In this KEP we propose small changes in formatting of variables in text format, but as preamble will stay exacly the same we will not treat it as a change.

### Feature enablement and rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [ ] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name:
    - Components depending on the feature gate:
  - [x] Other
    - Describe the mechanism: New flag `--logging-format` will be used to change logging format of component
    - Will enabling / disabling the feature require downtime of the control
      plane? Switching between log formats will require restarting control plane
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled). Switching between log formats will require kubelet restart

* **Does enabling the feature change any default behavior?**
  Default behaviour will not change logging format.

* **Can the feature be disabled once it has been enabled (i.e. can we rollback
  the enablement)?**
  Yes

* **What happens if we reenable the feature if it was previously rolled back?**
  Changing logging format will result in inconsistent schema of logs.

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling and disabling feature
  gates. However, unit tests in each component dealing with managing data created
  with and without the feature are necessary. At the very least, think about
  conversion tests if API types are being modified.
  **Changing logging format is stateless, no need to test this**

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g. what if some components will restart
  in the middle of rollout? **Failed rollout will cause inconsistent log format from control plane. Users should be
  already prepared to handle two different log formats when planning to switch logger so this is not a problem. It will
  not impact running workloads.

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?**
  Setting logging format is stateless, no need to test upgrade/rollback.

* **Is the rollout accompanied by any deprecations and/or removals of features,
  APIs, fields of API types, flags, etc.?**
  When using non default logging format, klog flags will no longer work. Klog will validate if users are using conflicting flags.

### Monitoring requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  In this exact case operator should use logs as this feature only impacts log output.
  Adding a metric doesn't give any additional benefit.
  Operators should monitor results of parsing logs to determine if all logs are correctly formatted.

* **What are the SLIs (Service Level Indicators) an operator can use to
  determine the health of the service?**
  Monitorin of logging pipeline is outside of Kubernetes scope.

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  N/A

* **Are there any missing metrics that would be useful to have to improve
  observability if this feature?**
  N/A


### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  N/A


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirms the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**
  N/A

* **Will enabling / using this feature result in introducing new API types?**
  N/A

* **Will enabling / using this feature result in any new calls to cloud
  provider?**
  N/A

* **Will enabling / using this feature result in increasing size or count
  of the existing API objects?**
  N/A

* **Will enabling / using this feature result in increasing time taken by any
  operations covered by [existing SLIs/SLOs][]?**
  N/A

* **Will enabling / using this feature result in non-negligible increase of
  resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Log volume increase analysis discussed in [Log volume increase analysis] section.
  Expected increase on level of 4% which is considered acceptable by sig scalability.

  Peformance was discussed in [Logger implementation performance] section.
  InfoS implementation for text is 9% slower than Infof.
  This increase should not have a big impact on overall Kubernetes performance as logging takes less than 2% of overall CPU usage

### Troubleshooting

Troubleshooting section serves the `Playbook` role as of now. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now we leave it here though.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**
  No impact

* **What are other known failure modes?**
  No failure modes for changing log format.

* **What steps should be taken if SLOs are not being met to determine the problem?**
  N/A

## Implementation History

* 2019-11-15 - Original proposal
* 2020-03-13 - Merged as provisional
* 2020-03-16 - Moved to implementable
* 2020-03-18 - Implementation started
* 2020-05-18 - Updated PRR questionnaire added