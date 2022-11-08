---
status: proposed
title: Workflows
creation-date: '2021-12-06'
last-updated: '2022-11-03'
authors:
- '@dibyom'
- '@lbernick'
---

# TEP-0098: Workflows

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Future Work](#future-work)
  - [Non-Goals](#non-goals)
  - [Use Cases](#use-cases)
- [Requirements](#requirements)
- [Design Considerations](#design-considerations)
  - [Extensibility and Conformance](#extensibility-and-conformance)
- [Prior Art](#prior-art)
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
- [Implementation Pull request(s)](#implementation-pull-requests)
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
* Allow clear separation of concerns between platform developers and app teams,
including making it easier to separate notifications and other infrastructure out of the Pipelines being run
* Create starter workflows?? https://docs.github.com/en/actions/using-workflows/using-starter-workflows

### Future Work

* Create a Workflows conformance spec

### Non-Goals

* Non CI/CD pipelines
* Installation/deployment of other Tekton projects that Workflows needs on behalf of the cluster operator
  * We can explore using the Operator project to simplify installation, but for an initial implementation
  the cluster operator will just be directed to install the necessary projects
* In-tree support for all possible SCMs, cloud storage systems, secrets storage systems, etc.

### Use Cases

The following example use cases are targeted by the Workflows project, but may be implemented via several milestones
(i.e. not all of these features need to be supported in an initial implementation). These use cases are examples, not a comprehensive list of features;
for example, we want to support CI in multiple SCMs, not just Github.

1. Github-based CI

- The end user would like to trigger a CI PipelineRun whenever a pull request is opened or a commit is pushed to the pull request,
without setting up any Github webhooks or Apps.
- If a new commit is pushed to the pull request, any CI PipelineRuns currently running for that branch should be canceled and
replaced by a new one.
- They may also want to trigger PipelineRuns based on pull request comments or labels.
For example, a test could be re-run if a user comments "/retest".
- They may want to run certain checks only if certain files have changed.
For example, they may want to skip unit tests if only markdown files are modified.
- The result of the CI PipelineRun is posted to Github Checks by Tekton, not by a finally Task owned by the PipelineRun user.
- CI PipelineRun results are archived and can be queried by repository.

2. Builds triggered by branch changes and cron jobs

- The end user would like to build and release their project to their dev cluster whenever changes are detected on the main branch.
- They don't want to set up an ingress on their cluster for security reasons, so they'd like to repeatedly poll the main branch for changes.
- They would like build failures to be posted to Slack, but don't want their app developers to have to include this notification logic in their build Pipeline,
especially since several other teams' build Pipelines should post to Slack too.
- They would like to kick off a daily release of their project to the prod cluster based on the changes pushed to the dev cluster a week prior.

3. CD system from scratch

A company has a centralized devops team rebuilding their CD system on Tekton components.
They would like a way to abstract a lot of the boilerplate and setup involved when onboarding new teams to the system.
Rather than write a new DSL they elect to use Tekton Workflows since it allows them to move more quickly than building and supporting their own solution would.

4. Platform builder

A company would like to make it easier for end users to run Tekton on its platform.
They would like to be able to extend Tekton Workflows to allow end users to trigger PipelineRuns from repositories connected
via their platform, view logs via their log storage, load secrets from their secret storage system, etc, even though
first-class support for their products is not built directly into Tekton Workflows.

## Requirements

- End users can set up an entire end to end CI pipeline (from source back to run results) using
only Tekton APIs, even if the implementation uses other infrastructure
- End users can configure a repo connection once and use this configuration for multiple Workflows. For example, they might want both a CI and a CD Workflow for a given repo.
- End users can define multiple repos in a workflow. For example, the release pipeline configuration might
exist in a different repo than the code being released.
- Platform teams can control or restrict access to portions of the CI/CD workflow e.g. service accounts, access to secrets etc.

## Design Considerations

### Extensibility and Conformance

Different SCMs have different APIs, authentication methods, and event payloads.
While Workflows should support the most commonly used SCMs out of the box, our goal is to avoid building in support for all SCMs.
In addition, platform builders might want their own logic for connecting to SCMs.
For example, the default implementation of a Github connection could allow the user to send events that trigger PipelineRuns from a Github App they create,
but a platform builder might want to use their own Github App and include other custom connection logic.
Therefore, Workflows must provide an extensibility mechanism to allow platform builders to create their own controllers for connecting to repos
and sending events from these repo connections, similarly to how users can define their own resolvers or Custom Tasks.

In addition, there are already several implementations of wrappers on top of Tekton Pipelines that allow end users to specify end-to-end CI/CD Pipelines
from version control. Extensibility mechanisms for Workflows could allow these projects to partially migrate to Workflows without a full rewrite,
leveraging Workflows for some common functionality but still relying on their own logic where necessary.

In addition to extensibility mechanisms, we should explore creating a Workflows conformance spec. This will allow existing (and new!) CI/CD projects
to become Workflows conformant if they don't want to use the default Workflows implementation, giving end users more flexibility in where to run their Workflows.

## Prior Art

- [RedHat Pipelines As Code](https://pipelinesascode.com/) is an E2E CI/CD system that allows users to kick off Tekton PipelineRuns based on events
from a connected git repository.
- [Flux CD](https://fluxcd.io/) allows users to define their app configuration in a git repository.
Whenever this configuration changes, Flux will automatically bring the state of their Kubernetes cluster in sync
with the state of the configuration defined in their repository.
  - A [POC of Workflows based on FluxCD](https://github.com/tektoncd/experimental/pull/921) found that the `GitRepository` CRD is a close analogue of the repo polling functionality
  described in [TEP-0083](./0083-scheduled-and-polling-runs-in-tekton.md), and is
  well suited for CD use cases.
  - The Flux `GitHub` receiver can be used to trigger reconciliation between a repo
  and a cluster when an event is received from a webhook, but the event body is not
  passed on to other components; it simply triggers an earlier reconciliation than
  the poll-based `GitRepository`. In addition, there's not a clear way to post statuses
  back to GitHub. Therefore, it's not well suited to CI use cases.
  - In addition, there's no plugin mechanism, and event bodies are controlled by FluxCD.
- [Knative Eventing](https://knative.dev/docs/eventing/) is a generic project for connecting event sources to event sinks.
  - There's a catalog of Knative EventSources, including GitHub, GitLab, and Ping EventSources, which could serve the use cases identified.
  Users can also create their own EventSources.
- Tekton experimental [Commit Status Tracker](https://github.com/tektoncd/experimental/tree/main/commit-status-tracker) creates a GitHub Commit Status
  based on the success or failure of a PipelineRun.
- Tekton experimental [GitHub Notifier](https://github.com/tektoncd/experimental/tree/main/notifiers/github-app) posts TaskRun statuses to either
  GitHub Statuses or GitHub Checks.

## Proposal

Initially, the cluster operator will install Tekton Workflows onto their Kubernetes cluster, and connect an individual repo
or connect a set of repos associated with the same organization in their SCM. This part of the user journey is discussed in
[Repo Connections](#repo-connections).

After a cluster operator installs Workflows, a Workflow can be set up in three ways:
1. Committing the Workflow directly to the repo it operates on, as discussed in [In-Repo Configuration](#in-repo-configuration).
2. Committing the Workflow to a separate repo that is reserved for configuration only, similar to the Tekton org's plumbing repo,
as discussed in [Configuration in Separate Repo](#configuration-in-separate-repo).
3. Applying the Workflow definition directly to the Kubernetes cluster, as discussed in [Configuration on Cluster](#configuration-on-cluster).
Most users are expected to interact with Workflows by committing their Workflow definitions directly to the repo being operated on,
and features related to this user journey are higher priority than features related to the other modes of interacting with Workflows.

Trusted developers can test changes to Workflows committed directly to a repo by opening a pull request with these changes.
See [Testing Workflows](#testing-workflows) for more information.

The [API](#api) section goes into more detail on the proposed API, with examples.
A Workflow definition encodes a PipelineRun and the events that trigger it, plus some built-in functionality for the most common use cases.
This section also includes default Workflow templates that users can create with the Tekton CLI.

Lastly, the [Milestones](#milestones) section proposes a roadmap for implementing the changes in this proposal.

### Repo Connections

The cluster operator or DevOps engineer needs some way to connect their repo to the project.
The initial version of the project should support connecting a single repo to Workflows.
Later, we will add support for connecting an organization to Workflows with
a single set of credentials, so that any repo in the organization can be used with Workflows.

Milestones:
1. Connect to a single GitHub repo via a webhook that is created on behalf of the user.
2. Connect to a single GitHub repo by creating a new GitHub App.
3. Connect to a GitHub organization by creating a new GitHub App, and install the app for a set of repos specified for that organization.
4. Support for existing GitHub Apps and generating events via polling Git repos.
6. Support for platform builders to add their own SCM connections.

#### Repo CRD

#### Ingress setup

#### Option 1: All Configuration exists in the CRD

##### Github webhook

Cluster operator creates this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: personal-access-token
  namespace: my-namespace
data:
  token: LS0tLS1C...
---
apiVersion: workflows.tekton.dev/v1alpha1
kind: RepositoryConnection
metadata:
  name: pipelines-repo
  namespace: my-namespace
spec:
  repo: pipelines
  connector: github-webhook
  accessToken:
    secretKeyRef:  # TODO: kubernetes-ism
      name: personal-access-token
      key: token
```

Workflows will create a kubernetes secret with a random string for validating the webhook payload.
TODO: security re: random string
It will issue a request to the GitHub API to create a webhook on the repo, and subscribe to "push" events initially.


TODO: When you subscribe to events, you need to update the connection -> when a workflow is created,
you need to update the webhook. When a workflow is deleted, you look at all other workflows to see if it's OK
to remove the event.

##### Github App

Cluster operator creates this:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: SCMConnection
metadata:
  name: tektoncd
  namespace: my-namespace
spec:
  organization: tektoncd
  repos:
  - pipelines
  - plumbing
  connector: github-app
```

Workflows will create a new GitHub App, subscribe to the "push" event, and create Kubernetes secrets for storing
the webhook secret and the app private key. It will then direct the user to install the app on the repo.

The SCMConnection could also create a RepositoryConnection for each repository listed.

##### Status

When a RepositoryConnection or SCMConnection is created, Workflows will update it with the following status:

```yaml
conditions:
  - type: Succeeded
    status: Unknown
    reason: ConnectionInProgress
```

After connection is complete, the status will be:

```yaml
conditions:
  - type: Succeeded
    status: "True"
    reason: ConnectionComplete
```

#### Option 2: Configuration in CRD, w/ extensibility

Cluster operator creates this:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: RepositoryConnection
metadata:
  name: pipelines-repo
  namespace: my-namespace
  labels:
    workflows.tekton.dev/connector: github-app
spec:
  url: https://github.com/tektoncd/pipeline
  customFields:
    appID: 12345
    appPrivateKey:
      name: github-app-private-key
      key: secret-token
```

Flow is the same as in option 1.

TODO: How does extensibility work?

#### Option 3: Repo connection is set up manually or via a CLI tool

In this solution, the cluster operator would connect their repo to Workflows using the tkn CLI,
similar to how setup is performed in Pipelines as Code. The operator could perform webhook or app creation
manually if they did not want to use the CLI.

TODO: Where is connection info stored?

### Workflow Definitions

#### In-Repo Configuration

Users create a /tekton folder in their repository for storing Tekton YAML files. Their Workflows are stored in /tekton/workflows,
although this path is configurable during installation. Our examples will show Tasks stored in /tekton/tasks and Pipelines stored in
/tekton/pipelines, but users may store Tasks and Pipelines anywhere in the repo and refer to them using remote resolution syntax,
for example:

```yaml
kind: Workflow
spec:
  pipelineRun:
    pipelineRef:
      resolver: git
      params:
      - name: name
        value: my-special-task-in-same-repo
      - name: pathInRepo
        value: /tekton/tasks/special-task.yaml
```
In this example, the Git default URL is set by the cluster operator.

TODO: We should avoid introducing any variable substitutions related to a Workflow's path in a repo, since these substitutions
wouldn't work for Workflows defined on a cluster.
TODO: How does variable substitution work for repo owner and git revision?

If the Workflow defines events, the event source will default to the repo the Workflow is defined in.
In the following example, the Workflow will trigger on "pull_request" events on the repo it's defined in:

```yaml
kind: Workflow
spec:
  events:
  - type: pull_request
    # Event source is not configured: defaults to repo where this is defined.
```

#### Configuration in Separate Repo

If a Workflow is defined in a separate repo than the one it operates on, it must define a [`repos` section](#repo-connections)
and specify which repos are sources for events. For example:

```yaml
kind: Workflow
spec:
  repos:
  - name: pipelines
  events:
  - type: pull_request
    source:
      repo: pipelines
  pipelineRun:
    pipelineRef:
      resolver: git
      params:
      - name: name
        value: my-special-task-in-same-repo
      - name: url
        value: $(repos.pipelines.url)
      - name: pathInRepo
        value: /tekton/tasks/special-task.yaml
```

The repo where the Workflow is defined must be connected to Workflows in order to run the Workflow if changed in a pull request.
The repo the Workflow operates on must also be connected, in order to run the Workflow when events occur on that repo.

#### Configuration on Cluster

Configuration directly on a cluster uses the same syntax as a Workflow defined in a separate repo.
We can create a CLI command to "export" a Workflow defined in-repo by making the defaults explicit.
For example, the command `tkn workflows export <workflow_name> --repo-name=<repo_name>` would add a repos section
and event sources where they are not present.

### Testing Workflows

When a trusted user opens a pull request that changes any Workflows, all changed Workflows will be run.
Any Workflows checked into the main branch that trigger on pull request events will run as well.
If such a Workflow is modified in the pull request, only the version in the pull request will run.

For an initial version of this proposal, a trusted user is defined as an organization member.
A pull request's Workflows can also be run if a trusted user comments "/ok-to-test" on the pull request.
Later, we should make this behavior configurable.

### API

This TEP introduces the Workflow CRD, a [runtime type](../design-principles.md#flexibility) used to configure
an end-to-end CI/CD process for a repository, including the events that will trigger a CI/CD PipelineRun and where
the PipelineRun's status will be posted.

A Workflow has 4 components:
- [Repositories](#repositories)
- [Events and Filters](#events-and-filters)
- A [PipelineRun](#pipelinerun)
- [Status](#status)

See [Alternatives: Workflow Definitions](#alternatives-workflow-definitions) for other ways this CRD could be defined.

#### Repositories

A developer can reference connected repositories in their Workflow definition via the `repos` field.
These could be repos to clone and run CI/CD on, or repos where configuration is stored. (These may be the same repo.)
When the Workflow is created, the controller will validate that these Repository CRDs exist in the same namespace and are connected.
Repos are optional, as they're not needed for cron-based Workflows.

Repos can be used in parameter substitutions. Supported substitutions are:
- `$(repo.<reponame>.url)`
- `$(repo.<reponame>.name)`
- `$(repo.<reponame>.owner)`

TODO: syntax for referencing repo in Workflow defined in-repo
TODO: If syntax for Workflow is different when it's defined in repo, you can't use existing tools to apply the definition to the cluster

#### Events and Filters

A Workflow also defines events that will cause a PipelineRun to be generated.

Events have a source, which can be a repo or a schedule (for cron jobs). For example:

```yaml
kind: Workflow
metadata:
  name: cd-pipeline
spec:
  repos:
  - name: pipelines
  events:
  - name: nightly
    source:
      cron: "0 0 * * *"
  - name: on-push
    source:
      repo: pipelines
```

Events with a `repo` source must define a field `types`, an array of strings such as "push" or "pull_request".
Supported event types depend on the repo's SCM. For example, GitHub has "pull_request" events, while GitLab has "merge_request" events.
This means Workflows must have logic for extracting an event type from different SCMs' event payloads.

As described in [Repo Connections](#repo-connections), SCM connections must be updated when Workflows are created or deleted,
in order to subscribe to the events of interest.

Events with a `repo` source may also define `filters`. Filters provide built-in ways of filtering events that correspond to the most
common CI/CD use cases, unlike Trigger Interceptors, which are more expressive.
Filters depend on the SCM, VCS, and event type, and the same filter might also be interpreted differently for different event types.
(For example, a Git branch filter would be interpreted as the branch changes are pushed to for a "push" event,
and the branch a pull request is opened against for a "pull_request" event.)
Built-in filters will be supported for only a subset of events and a subset of SCMs.

Built-in filters should include:
- a CEL filter (all event types)
- a filter for files changed (for "push" events and CI-related events)
- a filter for a Git branch (for "push" events and CI-related events)

TODO: Spell out filter syntax and supported event types for filters

#### PipelineRun

The PipelineRun to create when an event occurs.
The [original TEP-0098 proposal](https://docs.google.com/document/d/1CaEti30nnq95jd-bD1Iep4mEnZ5qMEBHax0ixq-9xP0) proposed
specifying a Pipeline rather than a PipelineRun. However, now that remote resolution has been implemented in Pipelines,
it's preferred to specify a PipelineRun with a remote reference to a Pipeline.

A Workflows installation will include a service account used to create PipelineRuns (unlike Triggers, where the user must define their
own service account for an EventListener, bound to existing roles).
The PipelineRun will be created with a label specifying the name of the workflow: `tekton.dev/workflow: workflow-name`.
For the initial implementation of this proposal, the PipelineRun will run in the same namespace as the Repo CRD
(and the Workflow CRD, if applicable).

TODO: details on parameter substitution from events

#### Status

A Workflow's status reflects any validation errors that aren't suitable for detecting in an admission webhook
(i.e. those that block and/or require API calls).

For example, a Workflow that declares repositories that aren't connected, or a Workflow where updates to a repo connection fail,
would have the following status:

```yaml
status:
  conditions:
  - type: Ready
    status: "false"
    reason: ValidationFailed
    message: "human-readable reason for validation error"
```

A Workflow with no errors would have the following status:

```yaml
status:
  conditions:
  - type: Ready
    status: "true"
```

A Workflow's status doesn't include information about PipelineRuns generated from the Workflow, as it's intended to be long-lived
and the status would only grow over time. Embedding PipelineRun status information in a Workflow may cause problems similar to those
created by embedding TaskRun statuses in PipelineRuns, as described in TEP-0100, and provides little value over querying
PipelineRuns using a label selector based on the Workflow. In the future, a Workflow status may include a reference to the Results API,
where PipelineRun statuses can be retrieved.

### Examples

#### CI Workflow

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: Workflow
spec:
  repos:
  - name: pipelines
  - name: plumbing
  events:
  - repo: pipelines
    type: pull_request
    filters:
      # TODO: opened by someone with permission to test, or has an ok-to-test comment from someone with permission to test
```

#### CD Workflow

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: Workflow
spec:
  repos:
  - name: pipelines
  - name: plumbing
  events:
  - repo: pipelines
    types:
    - push
    filters:
      gitRef: main  # Only runs on pushes to the main branch. If omitted, runs against pushes to any branch
      changedPaths: regex-goes-here
```

### Implementation

TODO

### Milestones

TODO: A lot of this depends on whether we adopt Pipelines as Code, but this section can also include details
on which features to prioritize and milestones for user experience.

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

TODO: Details on testing integrations with different Git providers

## Design Evaluation
<!--
How does this proposal affect the api conventions, reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

<!--
Why should this TEP _not_ be implemented?
-->

## Alternatives: Repo Connections

### Repo Configuration in Workflow Definition
- include URL in repos section

### Workflow can only be defined in-repo

### Workflow must always include repos section

## Alternatives: Workflow Definitions

### Embed Triggers

In this option, a Workflow definition includes a list of Triggers (with the same spec as the Triggers project)
that should fire when the events occur, and would not include `filters` or a PipelineRun.
Each event defined in `events` would be passed to each Trigger, and each Trigger would require an Interceptor
to filter to only the events of interest.

Here's what the CI EventListener would look like as a Workflow with Triggers:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: Workflow
metadata:
  name: github-ci
spec:
  repos:
  - name: pipelines
  events:
  - source:
      repo: pipelines
    types:
    - check_suite
  triggers:
    - name: github-listener
      interceptors:
        # No need for a GitHub interceptor.
        # Payload validation and event type filtering is handled by Workflows
        - name: "only when a new check suite is requested"
          ref:
            name: "cel"
          params:
            - name: "filter"
              value: "body.action in ['requested']"
      bindings:
      - name: revision
        value: $(body.check_suite.head_sha)
      - name: repo-url
        value: $(repos.pipelines.clone_url) # Variable replacement by Workflows instead of event body
      template:
        ref: github-template
  # No need to define KubernetesResources such as a load balancer and service account
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: github-template
spec:
  params:
    - name: repo-full-name
    - name: revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: github-ci-run-
      spec:
        pipelineSpec:
          tasks:
          # No Tasks for creating a GitHub API token or creating/updating a new Check.
          # Workflows will do this for pull request and check suite events.
          - name: clone
            taskRef:
              resolver: hub
              - name: git-clone
            workspaces:
            - name: output
              workspace: source
            params:
            - name: url
              value: $(params.repo-url)
            - name: revision
              value: $(params.revision)
          - name: tests
            taskRef:
              name: tests
            workspaces:
            - name: source
              workspace: source
            runAfter:
            - clone
        serviceAccountName: container-registry-sa
        params:
        - name: repo-url
          value: $(tt.params.repo-url)
        - name: revision
          value: $(tt.params.revision)
        - name: source
          volumeClaimTemplate:
            spec:
              accessModes:
              - ReadWriteOnce
              resources:
                requests:
                  storage: 1Gi
```

Pros:
- Easy to turn existing Tekton Triggers into Workflows
- Very flexible compared to proposed solution
- Easy to trigger TaskRuns and other resources from an event, instead of just PipelineRuns
- Starting with this solution allows us to focus on a great design for repo connections, event generation, and
notifications, instead of expanding scope to include simpler syntax for Triggers
- Don't need to bake in logic for filtering specific events based on SCM

Cons:
- It may be confusing to have to include some types of interceptors (e.g. CEL)
but not others (e.g. GitHub)
- May be too verbose and hard to understand compared to proposed solution

### Events API only

In this solution, we would create only an Events API for use with Triggers, instead of a Workflows API.

For example:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: Event
metadata:
  name: webhook
  namespace: my-namespace
spec:
  source:
    repo: pipelines
  types:
  - pull_request
  - issue_comment
  sink:
    eventListener: github-ci-eventlistener
```

When combined with repo connections, this solution allows users to map events coming from a repo (a source)
to existing EventListeners (sinks).

Pros:
- Very flexible
- Makes it easier to use Triggers for E2E workflows
- Don't need to bake in logic for filtering specific events based on SCM

Cons:
- Knative EventSources may be better suited for a proposal like this.
It's still very verbose to configure a CI/CD workflow.
- Doesn't easily allow for defining more complex configuration, such as concurrency
- Event configuration is separate from PipelineRun configuration, making it harder to understand

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

## Implementation Pull request(s)

<!--
Once the TEP is ready to be marked as implemented, list down all the Github
Pull-request(s) merged.
Note: This section is exclusively for merged pull requests, for this TEP.
It will be a quick reference for those looking for implementation of this TEP.
-->

## References

- [TEP-0021: Results API](./0021-results-api.md)
- [TEP-0032: Tekton Notifications](./0032-tekton-notifications.md)
- [TEP-0083: Scheduled and polling runs](./0083-scheduled-and-polling-runs-in-tekton.md)
- [TEP-0095: Common repository configuration](./0095-common-repository-configuration.md)
- [TEP-0120: Canceling concurrent PipelineRuns](./0120-canceling-concurrent-pipelineruns.md)
