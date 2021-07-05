---
status: proposed
title: Support retries for custom task in a pipeline.
creation-date: '2021-05-31'
last-updated: '2021-05-31'
authors:
- '@Tomcli'
- '@ScrapCodes'
---

# TEP-0069: Support retries for custom task in a pipeline.

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases (optional)](#use-cases-optional)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats (optional)](#notescaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [User Experience (optional)](#user-experience-optional)
  - [Performance (optional)](#performance-optional)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [References (optional)](#references-optional)
<!-- /toc -->

## Summary

A pipeline task can be configured with a `retries` count, this is 
currently only supported for `TaskRun`s and not `Run`s (i.e. custom tasks).

This TEP is about, a pipeline task can be configured with a `retries` count
for Custom tasks.

Also, a `PipelineRun` already manages a retry for regular task
by updating it's status. However, for custom task, a tekton owned controller
can signal a custom task controller, to retry. A custom task controller may
optionally support it.

## Motivation

A pipeline currently already supports a `task.retries`
for regular tasks, the same information can also be propagated for custom tasks.
It is possible, for a custom task controller can develop its own retries
support, which is not configurable at the pipeline level. So, each custom
controller has to implement `retries` in its own way. This would imply, it is
not possible for the custom tasks to propagate `context.pipelineTask.retries`
and `context.pipelineTask.retry-count` values.

In addition to supporting the above, this TEP will give us an opportunity to
bring in standard/uniform way of supporting retry amongst custom controllers.

A custom task controller - developer SDK, might also benefit from this support,
in the future, for example it can include documentation and stub code to make it
easy how to support it.

### Goals
* Support propagating `pipelineSpec.task.retries` count information to custom-task
  controllers.
* Support signalling a `retry` to custom task controller, for a specific run. A
  custom controller may optionally support it.
* Allow for custom task controllers to propagate the `context.pipelineTask.retry-count`
  and `context.pipelineTask.retries`.
* Gracefully handle the case where, custom controller does not support retry
  and yet the `PipelineRun` happens to be configured with retry. This also
  implies, an existing controller should not mis-behave if it is _not_ upgraded
  to support retries.

### Non-Goals
* Directly, force update the `status.conditions` of a custom task.


### Use Cases (optional)

<!--
Describe the concrete improvement specific groups of users will see if the
Motivations in this doc result in a fix or feature.

Consider both the user's role (are they a Task author? Catalog Task user?
Cluster Admin? etc...) and experience (what workflows or actions are enhanced
if this problem is solved?).
-->

## Requirements

None.

## Proposal

Requesting API changes:

1. Add field `Retries` to `RunSpec`, an integer count acts as an FYI. 
2. Add a new `RunRetry`, in addition to `RunCancelled` status to `RunSpecStatus`
   i.e. `v1alpha1.RunSpecStatusRetry`
3. Add a field `RetriesStatus` to `RunStatusFields`, to maintain the retry
   history for a `Run`, similar to `v1beta1.TaskRunStatusFields.RetriesStatus`

Proposed algorithm for performing a retry for custom task.

- Step 1. A `pipelineTask` consisting of a custom task X, is configured with 
  `retries` count.
  
- Step 2. On failure of task X, `pipelinerun` controller sees a request for a
  retry. It then communicates the same to custom task `Run` by patching 
  `/spec/status` with a `v1alpha1.RunSpecStatusRetry` i.e. `RunRetry`. Similar
  to request a custom task to cancel.
  
- Step 3. In addition to patching the `pipelinerun` controller also enqueue a timer
  `EnqueueAfter(30*time.Second)` (configurable). On completion of timeout
  (i.e. 30s), it checks if `/spec/status` is `RunRetry`, then it assumes that
  custom task does not support retry. 
    - a) if custom task does not supports retry as above, It sets no. of `retry done`
    to the `retries` count configured - i.e. exhaust all retries.
    - b) if custom task does support retry, update retry history and
      `context.pipelineTask.retry-count`

- Step 4. The custom task that wants to support the retry, has to update
  - a) clear `/spec/status` if it is `RunRetry`.
  - b) `status.conditions` to indicate it is `Running`.

_A task may retry and immediately fail, so controller cannot
fully rely on `status.conditions`._

### Notes/Caveats (optional)

Q. A Custom task does not support retry, and is configured to run with retry.
  How to gracefully handle this case?

  _Approach proposed:_ The `pipelineRun` waits for a configurable shorter timeout
  (say 30s), and if the custom task controller does not signal that it has begun
  to retry, assume it does not support retry.

Other options:

* Option 1: The `pipelineRun` should wait till the timeout and fail. The
  downside of this approach is, it may wait for a very long period of time.
* Option 2: It should have a way of knowing custom task does not support a
  retry. e.g. Custom controllers declaring that they support retry, somehow
  (not sure how this can be done).
  
### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate. Think broadly.
For example, consider both security and how this will impact the larger
kubernetes ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside the WGs or subproject.
-->

### User Experience (optional)

<!--
Consideration about the user experience. Depending on the area of change,
users may be task and pipeline editors, they may trigger task and pipeline
runs or they may be responsible for monitoring the execution of runs,
via CLI, dashboard or a monitoring system.

Consider including folks that also work on CLI and dashboard.
-->

### Performance (optional)

<!--
Consideration about performance.
What impact does this change have on the start-up time and execution time
of task and pipeline runs? What impact does it have on the resource footprint
of Tekton controllers as well as task and pipeline runs?

Consider which use cases are impacted by this change and what are their
performance requirements.
-->

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.

If it's helpful to include workflow diagrams or any other related images,
add them under "/teps/images/". It's upto the TEP author to choose the name
of the file, but general guidance is to include at least TEP number in the
file name, for example, "/teps/images/NNNN-workflow.jpg".
-->

## Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->

## Design Evaluation
<!--
How does this proposal affect the reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

<!--
Why should this TEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (optional)

<!--
Use this section if you need things from the project/SIG.  Examples include a
new subproject, repos requested, github details.  Listing these here allows a
SIG to get the process for these resources started right away.
-->

## Upgrade & Migration Strategy (optional)

<!--
Use this section to detail wether this feature needs an upgrade or
migration strategy. This is especially useful when we modify a
behavior or add a feature that may replace and deprecate a current one.
-->

## References (optional)

<!--
Use this section to add links to GitHub issues, other TEPs, design docs in Tekton
shared drive, examples, etc. This is useful to refer back to any other related links
to get more details.
-->
