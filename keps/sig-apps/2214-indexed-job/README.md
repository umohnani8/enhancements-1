# KEP-2214: Indexed Job

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [JobSpec API](#jobspec-api)
  - [Pod detail](#pod-detail)
  - [Job completion and restart policy](#job-completion-and-restart-policy)
  - [Job parallelism](#job-parallelism)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
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
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [x] (R) Graduation criteria is in place
- [x] (R) Production readiness review completed
- [x] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP extends kubernetes with user-friendly support for running
embarrassingly parallel jobs.

Here, parallel means multiple pods. By embarrassingly parallel, it means that
the pods have no dependencies between each other.
In particular, neither ordering between pods nor gang scheduling are supported.

We propose the addition of completion indexes into the Pods of a *Job
[with fixed completion count]* to support running embarrassingly parallel
programs, with a focus on ease of use.
We call this new Job pattern an *Indexed Job*, because each Pod of the Job
specializes to work on a particular index, as if the Pods where elements of an
array.

[with fixed completion count]: https://kubernetes.io/docs/concepts/workloads/controllers/job/#parallel-jobs

## Motivation

Users can use some [Job patterns] to run embarrassingly parallel Jobs, but those
approaches have downsides:

- The queue patterns require setting up an external queue service and modifying
  the Job binary to be able to connect to the queue.
  Depending on the implementation, it is prone to race conditions when
  coordinating which Pod works on which item.
- The template pattern doesn't scale when the parallelism level is too high,
  in terms of job creation and querying status.

Due to these reasons, workloads where each Pod just needs a unique and
ordered completion index, are hard to adapt to the existing Job patterns.

The lack of support for this pattern in k8s forces users to implement their
own APIs and controllers or adopt third party implementations. Each
implementation splits the ecosystem, making it harder for higher level systems
for Job queueing or workflows to support all of them.

[Job patterns]: https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-patterns

### Goals

- Support the *array Job* pattern by adding completion indexes to each Pod
  of a Job in *fixed completion count* mode.

### Non-Goals

- Support for work lists, where each Pod receives a different element of a
  static list. This can be implemented by users from completion indexes.
- Support for completion index in non-parallel Jobs or Jobs with a work queue.
- All-or-nothing scheduling.

## Proposal

### User Stories (Optional)

#### Story 1

As a Job author, I can create an array Job where each Pod receives an ordered
completion index. I can use the index in my binary through an environment
variable or a file to statically select the load the Pod should work on.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-work
spec:
  completions: 100
  parallelism: 100
  template:
    spec:
      containers:
      - name: task
        image: registry.example.com/processing-image
        command: ["./process",  "--index", "$INDEX"]
        env:
        - name: INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.alpha.kubernetes.io/job-completion-index'] 
```

### Notes/Constraints/Caveats (Optional)

* An earlier proposal for [indexed Job]
suggested the support for work lists, i.e. passing different parameters to each
Pod. We decided to leave this out of the proposal to keep it simple and
because work lists can be implemented in a startup script using the completion
index as building block.
* The semantics of an indexed Job are similar to a StatefulSet, in the sense
that Pods have an associated index.
However, the APIs have a major difference: a StatefulSet doesn't have completion
semantics, as opposed to Jobs.

[indexed Job]: https://github.com/kubernetes/community/blob/b21d1b27c8c748bf81283c2d89cde2becb5f2709/contributors/design-proposals/apps/indexed-job.md

### Risks and Mitigations

Jobs have a known issue in which more than one Pod can be started even if
parallelism and completion are set to 1 ([reference]). In the case of indexed
Jobs, this translates to more than one Pod having the same index.

Just like for existing Job patterns, workloads have to handle duplicates at the
application level.

[reference]: https://kubernetes.io/docs/concepts/workloads/controllers/job/#handling-pod-and-container-failures

## Design Details

### JobSpec API

The JobSpec gets the field `completionMode` to control whether the Job should	
be treated as an Indexed Job.	

```golang
// CompletionMode specifies how Pod completions of a Job are tracked.
type CompletionMode string

const (
	// NonIndexedCompletion means that Pod completions of a Job are
	// indistinguishable from each other.
	NonIndexedCompletion CompletionMode = "NonIndexed"

	// IndexedCompletion means that each Pod completion of a Job is tracked
	// individually, being associated to a completion index.
	IndexedCompletion CompletionMode = "Indexed"
)

type JobSpec struct {	
  ...	
  // CompletionMode specifies how Pod completions are tracked. It can be	
  // `NonIndexed` (default) or `Indexed`.	
  //
  // `NonIndexed` means that each Pod completion is homologous to each other.	
  // The Job is considered complete when there have been .spec.completions	
  // successful completions.	
  //
  // `Indexed` means that each Pod completion needs to be tracked individually;	
  // each Pods gets an associated completion index, which is available in the
  // annotation	`batch.alpha.kubernetes.io/job-completion-index`.	
  // The Job is considered complete when there is one successful Pod for each	
  // index in the range 0 to (.spec.completions - 1).	
  // When value is `Indexed`, .spec.completions must have a non-zero positive
  // value and `.spec.parallelism` must be less than or equal to 10^5.
  //
  // More completion modes can be added in the future. If a Job controller
  // observes a mode that it doesn't recognize, it manages the Job as in
  // `NonIndexed`.
  CompletionMode CompletionMode
}	
```

As the comment describes, when `.spec.completionMode = "Indexed"`, the
`.spec.completions` must be:

- a non-zero positive value. This is to trigger Job management strategy for
  *fixed completion count*. That is, `Indexed` mode cannot	be used for work
  queue patterns.	
- less than or equal to `10^6`. This is to guarantee that we can keep track of
  completions per-index in the Job status in the future.

### Pod detail

The Pod and PodSpec APIs don't get any new fields. However, Pods created for
Indexed Jobs get the annotation `batch.alpha.kubernetes.io/job-completion-index`
with a value equal to its completion index. The annotation is immutable.

The annotation can be accessed through the downward API as a file or environment
variable.

For user convenience, the Job controller adds the completion index as an
environment variable through the downward API. That is, the Job controller
creates Pods like so:

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: test-container
      env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.alpha.kubernetes.io/job-completion-index'] 
```

The Job controller doesn't add the environment variable if there is a name
conflict with an existing environment variable. Users can specify other
environment variables for the same annotation.

### Job completion and restart policy

When dealing with Indexed Jobs, the Job controller keeps track of Pod
completions for each index from 0 to `.spec.completions - 1`.
Once the controller notices that a Pod finishes successfully, it will not create
another Pod with the same index.

The Job controller considers a Job completed when there is at least one
successful Pod for each completion index.

If an entire Pod fails, such as when the Pod gets kicked off the node or if a
container of the Pod fails and `.spec.template.spec.restartPolicy = "Never"`,
the Job controller gets the completion index for the failed Pod and creates a
new Pod with the same index. The application needs to handle restarts in a
different Pod.

The Pod might not be immediately replaced, as the number of active Pods could
have hit the parallelism limit. Once another Pod finishes, the Job controller
syncs the Job, scanning for unsatisfied indexes and creates the missing Pods for
them.

The kubelet handles container restarts as usual, according to the
`spec.template.spec.restartPolicy`.

<<[UNRESOLVED TBD Beta: Track completed indexes in Job status]>>
Once [kubernetes/kubernetes#28486](https://github.com/kubernetes/kubernetes/issues/28486)
is resolved:

The Job controller keeps track of completed indexes in
`.status.completedIndexes`, a string that represents a list of numbers in a
compressed format. For example, if a Job has completed indexes 2, 3, 4, 6 and 7,
the list looks like:

```golang
CompletedIndexes: "2-4,6-7"
```

The `kubectl describe` command crops the list of indexes if it's too long:

```
Completed Indexes: [1-25,28,30-32,...]
``` 
<<[/UNRESOLVED]>>

### Job parallelism

A user can change the number of active Pods for a Job changing `.spec.parallelism`
(note that `.spec.completions` is an immutable field).

When starting a Job or increasing the parallelism, the Job controller creates
Pods with lower completion index first, as long as there is no other completed
or running Pod with the same index. This is to make the controller behavior more
predictable. We do not offer guarantees on creation order based on completion
index.

Reducing parallelism is unaffected by completion index.

### Test Plan

Unit, integration and E2E tests cover the following Indexed Job mechanics:

  - Creation with indexed Pod names and index annotations.
  - Scale up and down.
  - Pod failures.
  
Additionally, we add unit tests for API defaulting and validation, with feature
gate enabled and disabled.
  
### Graduation Criteria

#### Alpha

- Complete features:
  - Completion index in annotation
  - Restart policy

#### Alpha -> Beta Graduation

- Complete features:
  - Tracking completions by index in Job status
- Gather feedback from end users and operators' developers. Open questions:
  - Are stable Pod names necessary?
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- E2E test graduates to conformance
- 2 examples of operators using Indexed Jobs.

### Upgrade / Downgrade Strategy

In the event of a kube-controller-manager upgrade, there should not be any
existing Indexed Jobs with running Pods.

In the event of a downgrade, existing Indexed Jobs will run as NonIndexed Jobs.
The controller will track the existing Pods ignoring the completion index. New
Pods will be created without a completion index. Existing workloads that
expected the completion index will fail. But this is expected in a downgrade.

In the event of an upgrade after a downgrade, the controller will remove
existing Pods without completion index for existing Indexed Jobs, without they
counting towards failures or completions. The controller will create new Pods
with indexes. This might cause a load spike in the cluster.

If, instead of a downgrade, the cluster administrator disables the feature gate:

  - kube-apiserver clears `.spec.completionMode` for new Jobs at creation time.
    That is, all new Jobs are interpreted as `NonIndexed`.
  - kube-controller-manager skips syncing existing Indexed Jobs and emits a
    warning event. More specifically, the controller does not create new Pods,
    track completion nor update status of the Job.
  
The above guarantees that the controller never creates Pods for Indexed Jobs
without a completion index.

### Version Skew Strategy

This features has no node runtime implications.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**

  - [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: IndexedJob
    - Components depending on the feature gate:
      - kube-apiserver
      - kube-controller-manager

* **Does enabling the feature change any default behavior?**

  No. Jobs need to opt-in with `.spec.completionMode=Indexed`.

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**
  
  Yes. Using the feature gate is the recommended way.

* **What happens if we reenable the feature if it was previously rolled back?**

  The Job controller starts managing Indexed Jobs again.
  More details covered in [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

* **Are there any tests for feature enablement/disablement?**

  Yes, unit and integration test for the feature enabled, disabled and
  transitions.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g., what if some components will restart
   mid-rollout?

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**
  Describe manual testing that was done and the outcomes.
  Longer term, we may want to require automated upgrade/rollback tests, but we
  are missing a bunch of machinery and tooling and can't do that now.

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs, 
fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
  checking if there are objects with field X set) may be a last resort. Avoid
  logs or events for this purpose.

* **What are the SLIs (Service Level Indicators) an operator can use to determine 
the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At a high level, this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide comprehensive guidance, but at the very
  high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code

* **Are there any missing metrics that would be useful to have to improve observability 
of this feature?**
  Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
  implementation difficulties, etc.).

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  Think about both cluster-level services (e.g. metrics-server) as well
  as node-level agents (e.g. specific version of CRI). Focus on external or
  optional services that are needed. For example, if this feature depends on
  a cloud provider API, or upon an external software-defined storage or network
  control plane.

  For each of these, fill in the following—thinking about running existing user workloads
  and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**

  No.

* **Will enabling / using this feature result in introducing new API types?**

  No.

* **Will enabling / using this feature result in any new calls to the cloud 
provider?**

  No.

* **Will enabling / using this feature result in increasing size or count of 
the existing API objects?**

  Yes.
  
  - API type(s): Job
  - Estimated increase in size: new field of about 30 bytes.
  
  - API type(s): Pod, only when created with the new completion mode.
  - Estimated increase in size: new annotation of about 50 bytes.

* **Will enabling / using this feature result in increasing time taken by any 
operations covered by [existing SLIs/SLOs]?**

  No.

* **Will enabling / using this feature result in non-negligible increase of 
resource usage (CPU, RAM, disk, IO, ...) in any components?**

  Additional CPU and memory increase in the controller-manager is negligible
  and restricted to Jobs using the new completion mode.

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
  For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

* 2021-01-08: First version of the KEP in provisional status. Design Details
  completed.

## Drawbacks

* Adds more complexity to the Job controller in terms of Pod and Pod status
  management, as it introduces a new mode.
* Failed Pods are removed before being replaced by new Pods, reducing end-user
  debugging capabilities. However, failed Pod persists when the whole Job fails.

## Alternatives

- **Leave Indexed Job to third-party implementations**

  The major painpoint is that this leaves Pod management to the third-party
  implementation. With different implementations, the ecosystem is split, making
  it harder for higher level Job orchestration frameworks to support all of
  them.
  
  On the other hand, with the Indexed Job native support in core k8s,
  third-party implementations can focus on application level APIs, using the Job
  API as their underlying Pod management mechanism.
  
- **Completion Index in the Pod Name**

  Completion indexes could also be part of the Pod name, leading to stable Pod
  names. This allows 2 things:
  - Uniqueness for each completion index, freeing applications from having to
    handle duplicated indexes.
  - Predictable hostnames, which benefits applications that need to communicate
    to Pods of a Job (or among Pods of the same Job) without having to do
    discovery.
  
  Stable pod names require the Job controller to remove failed Pods before
  creating a new one with the same index. This has some downsides:
  - Removing Job Pods is a breaking change. But this can be done if it's a new
    Job execution mode accessible through a JobSpec field.
  - Currently, the Job controller uses the tombstones of failed Pods to track
    the status of the Job, affecting retry backoffs and backoff limit. This
    needs to change before stable Pod names can be implemented
    [#28486](https://github.com/kubernetes/kubernetes/issues/28486).
  - Reduced availability of Job Pods per completion index. This happens when
    a Node becomes unavailable. The Job controller cannot remove such Pods.
    Either the kubelet in the Node recovers and marks the Pod as failed; or the
    kube-apiserver removes the Node and the garbage collector removes the orphan
    Pods.
    
  However, stable Pod names can be offered later as a new value for
  `.spec.completionMode` for Jobs.
