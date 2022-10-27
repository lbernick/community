---
status: proposed
title: Workflows
creation-date: '2021-12-06'
last-updated: '2021-12-06'
authors:
- '@dibyom'
- '@lbernick'
---

# TEP-0098: Workflows

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases](#use-cases)
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
- [Implementation Pull request(s)](#implementation-pull-request-s)
- [References](#references)
<!-- /toc -->

## Summary

This TEP introduces the experimental Workflows API. The goal is to provide a way for users to set up and manage end to end CI/CD workflow configuration in a single place by relying on other Tekton primitives like Pipelines and Triggers.

## Motivation

An end to end CI/CD system consists of a number of pieces e.g. triggering the pipeline off of some events (git push), actually running the pipeline,  notifying the end users of the run status, storing the run results for later usage etc.

Tekton today contains a number of these pieces that can be combined together to build a full CI/CD system. This loosely coupled flexible approach allows users to only use the portions of Tekton that they need. For example, users can use Pipeline with their own triggering system, or they can use both Pipelines and Triggers but use their own CLI or visualization mechanism.

This flexibility comes with increased verbosity and configuration management burden. For a CI/CD system that uses Tekton end to end, users have to maintain multiple CRDs (sometimes with duplicate information). These files have to be managed and kept in sync manually. In many cases, changes in one will require changes in others e.g updating a pipeline with a new parameter would mean adding a new parameter to the TriggerTemplate as well as the TriggerBinding. Users will then have to ensure the cluster is updated with both the new pipeline as well as trigger configuration.

Most CI/CD workflows also require interactions with external systems such as GitHub. Tekton today does not provide a way to declare these in a standardized way making it harder to visualize and debug the entire end to end workflow in a single place e.g. it is hard to visualize all pipeline runs for a given repository.

Standardizing all of the configuration in one place can also enable some enterprise use cases. For instance, platform users can control the Workflow resource fields such as service accounts, secret configuration for various third party integrations while the app developer teams can still manage the actual pipeline definition in code

A Workflow resource can be the single entry point for the end to end CI/CD configuration and 
simplify the Tekton experience. It can provide a single place to declare all the pieces that are needed for the complete workflow including third party resources (secrets, repositories) that are used by multiple Tekton systems.

### Goals

* Provide a way to describe entire CI/CD configuration (not just pipeline definition) in a single place
* Make managing Tekton configuration less verbose and repetitive.
* Make it easier and simpler for end users to get started with a pure Tekton CI system
* Allow clear separation of concerns between platform developers and app teams
* Allow for clear separation between run configuration and pipeline definition?
* Platform builders can implement their own connectors (???)
* Define workflow API conformance?

### Non-Goals

* Non CI/CD pipelines
* Build support for all SCM providers or notification sinks

### Use Cases

1. End users can set up an entire end to end CI pipeline (from source back to run results) using only Tekton 
1. End users can visualize runs grouped by repository
1. End users can create an end to end CI/CD workflow e.g. no scripts to set up webhooks, GH Apps etc.

1. Platform teams can control or restrict access to portions of the CI/CD workflow e.g. service accounts, access to secrets etc.
1. A company has a centralized devops team rebuilding their CD system on Tekton components. They would like a way to abstract a lot of the boilerplate and setup involved when onboarding new teams to the system. Rather than write a new DSL they elect to use Tekton Workflows since it allows them to move more quickly than building and supporting their own solution would. 


The following use cases are targeted by the Workflows project, but may be implemented via several milestones.

Example 1: Github-based CI
- The end user would like to trigger a CI PipelineRun whenever a pull request is opened or a commit is pushed to the pull request,
without setting up any Github webhooks.
- If a new commit is pushed to the pull request, any CI PipelineRuns currently running for that branch should be canceled and
replaced by a new one.
- They may also want to trigger PipelineRuns based on pull request comments or labels.
For example, a test could be re-run if a user comments "/retest".
- The result of the CI PipelineRun is posted to Github Checks.
- CI PipelineRun results are archived and can be queried by Workflow.

Example 2: Builds triggered by branch changes
- The end user would like to build and release their project whenever changes are detected on the main branch.
  - They may want to do this using a "push" strategy, where the build occurs as soon as changes are pushed,
  or via a "polling" strategy, where a Tekton component polls the branch for updates at some regular interval.

Example 3: Nightly releases
- The end user would like to build and release their project on a scheduled cadence.
- If the build fails, they would like to receive a notification.
^ This is all possible with Tekton today-- what's the differentiating factor?
Instead of having a separate Tekton TaskRun at the end of your PipelineRun (or triggered by it), the notification would be built in.

Example 4: Platform builder
- A platform builder would like to support CI/CD workflows combining Tekton with their own products (???).
For example, they might want to allow secrets to be easily loaded from their secrets service,
blobs to be easily loaded into workspaces from their storage service, or to implement custom connection
logic to SCMs but still easily trigger PipelineRuns from SCM events.

Example 5: Deployment
- might include manual approval?

TODO:
- Why would you want to trigger an event for a branch regex, tag, or commit?
- support for multiple repos?
- standardized event definition? each provider might have a different event payload. still want access to full event payloads if needed
- authentication for SCM connections
- what should workflow status represent? can you have a workflow run?
- results grouped by workflow?
- manual triggering for easier testing/setup
- might want to allow platform builders to swap out where secrets are stored


## Requirements

<!--
Describe constraints on the solution that must be met. Examples might include
performance characteristics that must be met, specific edge cases that must
be handled, or user scenarios that will be affected and must be accomodated.
-->

- Must be possible to extend Workflows to support events coming from arbitrary sources (?)

## Background

### CI with various SCM providers
- different CI workflows
- different forms of authentication
- seems like we need a Github connector, but what if platform builders want to connect in different ways?

## Proposal

- Project has been created in experimental. Update dogfooding to use it.
- Initial implementation may require installing Tekton Triggers or other projects.
- TODO: Do we want to define some sort of conformance? What happens to Pipelines as Code and Nubank Workflows?

### API


### Implementation

#### Event Sources

If you specify

```yaml
spec:
  repos:
  - name: pipelines
    url: https://github.com/tektoncd/pipeline
    vcsType: github  # could be parsed from url, maybe
    accessToken:  # could workload identity be used somehow? probably not.
      secretKeyRef:
        name: knative-githubsecret
        key: accessToken
    secretToken:
      secretKeyRef:
        name: knative-githubsecret
        key: secretToken
  triggers:
  - eventSource:
      repo: pipelines
    eventTypes:
    - push
```

We would turn this into a knative event source:

```yaml
apiVersion: sources.knative.dev/v1alpha1
kind: GitHubSource
metadata:
  name: github-push
  namespace: dogfood
spec:
  eventTypes:
    - push
  ownerAndRepository: tektoncd/pipeline
  accessToken:
    secretKeyRef:
      name: knative-githubsecret
      key: accessToken
  secretToken:
    secretKeyRef:
      name: knative-githubsecret
      key: secretToken
  sink:
    ref:
      apiVersion: v1
      kind: Service
      name: el-workflows
      namespace: tekton-workflows
```

and trigger:

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
spec:
  interceptors:
  # Filter to only events coming from the Github EventSource
  template:
  # Template for a PipelineRun
```

(could also do extrafields as in customruns as opposed to params)
- How would the user specify that they want a polling Github source instead of a webhook based one?
- How would the user specify that they want to use a Github app?

Maybe we implement a Github app thing that can take any app???

Maybe we have our own GH event source; you can specify GH webhook, GH app, or polling as an option

We do not want to tie ourselves to knative event sources (i.e. they don't go in the api) but we don't want to reinvent the wheel, 
so our implementation can probably use them.

Since all of our event sources will have the same sink to start, we can just use a sinkbinding: https://knative.dev/docs/eventing/custom-event-source/sinkbinding/create-a-sinkbinding/

Maybe we can allow you to specify a repo separately, create a githubsource w/ all event types, and for each event type in the workflow trigger,
create an interceptor w/ a filter?

Things that we automagically take care of:
- setting the sink to be the eventlistener service
- filtering to events of interest?



Create our own Github App Knative EventSource (or update the existing one to allow specifying a Github app). Same for polling.

#### Notifications (Event Sinks)

We can set up a cloud event sink that listens for tekton cloud events and posts a body to the given URL.
- how to specify which events, where to post them, and what message body?
- some endpoints need authentication!

#### Results

#### Authentication



### Repo/Event Interface

Cloning a repo is easy. The problem is that we want to produce events from a repo. (Should also work for CronJobs.)
- Do we want to inject a step that automatically clones the repo?

Option 1: Repo interface (polling)

- This would probably look like the FluxCD GitRepository CRD.
- Might have to improve auth a bit.
- We would only need one implementation: a git clone. No need for platform builders to do anything differently.
- What would the event body look like? We would have to generate that ourselves.
- How would this work for a pull request?
- How about a cron job?

Example:
```yaml
kind: GitRepositoryBranch
metadata:
  name: pipelines-repo
spec:
  interval: 5m0s
  url: https://github.com/tektoncd/pipeline
  ref:
    branch: main
```

```yaml
kind: Workflow
spec:
  triggers:
  - eventSource:
      name: pipelines-repo
      kind: GitRepositoryBranch
```

We generate an event when the repo changes.

Option 2: Repo interface (pushing)

```yaml
kind: GitRepositoryEvents
metadata:
  name: pipelines-repo
spec:
  url: https://github.com/tektoncd/pipeline
  vcsType: github  # TODO: could potentially be parsed out of the URL
  eventTypes:  # This goes on the GitRepositoryEvents rather than on the workflow because different VCS providers
  # may have different event bodies so we can't really implement a generic filter 
  - check_suite
```

```yaml
kind: Workflow
spec:
  triggers:
  - eventSource:
      name: pipelines-repo
      kind: GitRepositoryEvents
```

- Here, we'd set up a webhook for the user on the repo.
- We would pass the event body from the SCM directly onwards, w/ filtering.
- For Github, we'd probably ask the user to install a Tekton github app on that repo.
- Would need separate implementation for separate SCMs.


Option 3: Repo interface. Implementation can be either pushing or pulling (??????)

```yaml
kind: GitRepositoryConnection
spec:
  url: https://github.com/tektoncd/pipeline
  connector: github-webhook-connector
  params:
  - name: eventTypes
    value: ["check_suite"]
```
OR

```yaml
kind: GitRepositoryConnection
spec:
  url: https://github.com/tektoncd/pipeline
  connector: github-poller
  params:
  - name: polling-interval
    value: 0m0s
```

```yaml
kind: GitRepositoryConnection
spec:
  url: https://github.com/tektoncd/pipeline
  connector: custom-connector-from-platform-builder
  params:
  - name: custom-param
    value: some-value
```

```yaml
kind: Workflow
spec:
  triggers:
  - eventSource:
      name: pipelines-repo
      kind: GitRepositoryConnection
```

- how to do filtering?

Option X: separate connection to Github from connection to repo.

```yaml
kind: SCMConnection
spec:
  connector: custom-connector-from-platform-builder
```

```yaml
kind: GitRepository
spec:
  url: https://github.com/tektoncd/pipeline
  connectionRef:
    name: my-connection
```


Option 4: Events interface
- allows us to be agnostic about event source and polling vs pushing
- could base this on event queues or k8s event spec, cloud event spec
https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#event-v1-events-k8s-io
https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md
- Need to be extremely clear about what parts of the event spec are optional vs required. Hard to add required fields
- Cloud events spec could work well since tekton already produces them
- How can user know what event body to expect?
- Could use knative event sources

TODO: Tekton implementation vs platform builder implementation






Repo CRD: Just has repo URL

RepoEvent: could be a poll event or a push event?


```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: GitRepository
spec:
  url: https://github.com/tektoncd/pipeline
  extraFields:
    pollingInterval: # optional
    webhookSecret: # optional
    appID: # optional
    # TODO: probably will also need some additional auth?
```

- how would this allow GCB to provide their own implementation?





We allow people to implement EventControllers.
- Repo EventControllers could be a subset of EventControllers?
- controller needs a source (repo?), event types, and a sink (???)


```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: Event
spec:
  source:
    repoRef:
      name: my-repo
  types:


```

Maybe instead of thinking about CRDs, what's the http api that this needs to implement?

- Needs a "create connection" api
- Needs "create repo" for a given connection?
- Create trigger for a given repo + sink URL?

******* Maybe the Repo just generates a bunch of CloudEvents and blasts them into space, and then other things can subscribe to them? ******

This would essentially work by using the eventlistener as the broker, and CEL filters as subscriptions.

Not sure if we'd want a separate SCMConnection CRD, or if this would just be part of installing the connector?

Repo should also provide an endpoint where it can accept CloudEvents generated by Tekton and post them back to the SCM.
Where does filtering happen?

If repo connector emits all possible events (instead of having the webhook subscribe to only some of them), we might slow
down the eventlistener because it has to process many more events and make many more requests to the CEL filter


TODO: Implement a piece of code that will accept an installation ID and PAT and send events to the given sink, and accept Tekton events
at a given URL.

TODO: implement a reverse event listener? Or an event listener for Tekton events?



## Prior Art

### Pipelines as Code: Connection via Github App

### Flux CD: polling model

### Knative EventSources: webhook





### Notes/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate. Think broadly.
For example, consider both security and how this will impact the larger
kubernetes ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside the WGs or subproject.
-->

- TODO: List out potential problems with Tekton Triggers
- Knative eventing used to require istio (large dependency that not everyone wanted to install). Now does not-- is it still feasible?
- may need to implement github app event source

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
How does this proposal affect the api conventions, reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

<!--
Why should this TEP _not_ be implemented?
-->

## Alternatives


### Implementation using Flux CD

TODO: Link POC

## Infrastructure Needed (optional)

<!--
Use this section if you need things from the project/SIG.  Examples include a
new subproject, repos requested, github details.  Listing these here allows a
SIG to get the process for these resources started right away.
-->

## Implementation Pull request(s)

<!--
Once the TEP is ready to be marked as implemented, list down all the Github
Pull-request(s) merged.
Note: This section is exclusively for merged pull requests, for this TEP.
It will be a quick reference for those looking for implementation of this TEP.
-->

## References

### Related Problems
- [TEP-0021: Results API](./0021-results-api.md)
- [TEP-0032: Tekton Notifications](./0032-tekton-notifications.md)
- [TEP-0083: Scheduled and polling runs](./0083-scheduled-and-polling-runs-in-tekton.md)
- [TEP-0095: Common repository configuration](./0095-common-repository-configuration.md)
- [TEP-0120: Canceling concurrent PipelineRuns](./0120-canceling-concurrent-pipelineruns.md)

### Related Projects
- [RedHat PipelinesAsCode](https://pipelinesascode.com/)
- [Flux CD](https://fluxcd.io/)
- [Knative Eventing](https://knative.dev/docs/eventing/)
