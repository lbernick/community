---
status: proposed
title: Workflows
creation-date: '2021-12-06'
last-updated: '2021-12-06'
authors:
- '@dibyom'
---

# TEP-0098: Workflows

<!--
**Note:** When your TEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the TEP with the
  appropriate Working Group.
- [ ] **Create a PR for this TEP.**
  Assign it to people in the SIG that are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the TEP clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a TEP is merged does not mean it is complete or approved.  Any TEP
marked as a `proposed` is a working document and subject to change.  You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing TEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused.  If you disagree with what is already in a document, open a new PR
with suggested changes.

If there are new details that belong in the TEP, edit the TEP.  Once a
feature has become "implemented", major changes should get new TEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/teps/NNNN-TEP-template/README.md).

-->

<!--
This is the title of your TEP.  Keep it short, simple, and descriptive.  A good
title can help communicate what the TEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a TEP and for
highlighting any additional information provided beyond the standard TEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

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
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [Implementation Pull request(s)](#implementation-pull-request-s)
- [References (optional)](#references-optional)
<!-- /toc -->

## Summary

<!--
This section is incredibly important for producing high quality user-focused
documentation such as release notes or a development roadmap.  It should be
possible to collect this information before implementation begins in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->
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

<!--
List the specific goals of the TEP.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

* Provide a way to describe entire CI/CD configuration (not just pipeline definition) in a single place
* Make managing Tekton configuration less verbose and repetitive.
* Support end to end CI workflows from source control
* Make it easier and simpler for end users to get started with a pure Tekton CI system
* Allow clear separation of concerns between platform developers and app teams
* Allow for clear separation between run configuration and pipeline definition?

### Non-Goals

<!--
What is out of scope for this TEP?  Listing non-goals helps to focus discussion
and make progress.
-->
* Non CI/CD pipelines
* Installation/deployment of other Tekton projects that Workflows needs as pre-requsities

### Use Cases

<!--
Describe the concrete improvement specific groups of users will see if the
Motivations in this doc result in a fix or feature.

Consider both the user's role (are they a Task author? Catalog Task user?
Cluster Admin? etc...) and experience (what workflows or actions are enhanced
if this problem is solved?).
-->

1. End users can set up an entire end to end CI pipeline (from source back to run results) using only Tekton 
1. End users can visualize runs grouped by repository
1. End users can create an end to end CI/CD workflow e.g. no scripts to set up webhooks, GH Apps etc.

1. Platform teams can control or restrict access to portions of the CI/CD workflow e.g. service accounts, access to secrets etc.
1. A company has a centralized devops team rebuilding their CD system on Tekton components. They would like a way to abstract a lot of the boilerplate and setup involved when onboarding new teams to the system. Rather than write a new DSL they elect to use Tekton Workflows since it allows them to move more quickly than building and supporting their own solution would. 

## Requirements

<!--
Describe constraints on the solution that must be met. Examples might include
performance characteristics that must be met, specific edge cases that must
be handled, or user scenarios that will be affected and must be accomodated.
-->

## Milestone 1 Proposal: Generating events from connected repositories


### CI Workflow with repo connections + eventing + triggers

### Workflows API

The initial experimental version of the workflows API will focus on connecting repo-generated events to the existing Tekton Triggers API.
After this experimental version is implemented, we can design syntactic sugar that reduces verbosity of the full setup. 

Example:
```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: Workflow
metadata:
  name: pipelines-ci
  namespace: tekton-ci
spec:
  repos:
  - name: pipelines
    ref:  # In the future, we may also support `spec`
      name: pipelines
      namespace: core
  - name: plumbing
    ref:
      name: plumbing
      namespace: ops
  triggers:
  - name: on-pr
    event:
      source:
        repo: pipelines  # Under the hood, this will use the github interceptor
      types:
      - check_suite
    filters:
      cel: "body.action in ['requested']"
  - name: on-pr-comment
    event:
      source:
        repo: pipelines
      types:
      - pull_request_review_comment
      - issue_comment
    filters:
      cel: "body.comment.body startswith '/retest'"  # fixme

```




### Repository connections

In order to allow platform builders to support connections to additional SCMs, or connections to existing SCMs in multiple ways, repository connections will be handled by
"connector" controllers that support authentication and event generation for a given SCM. <!-- FIXME: SCM vs VCS? -->
Tekton Workflows will ship with a "Github" connector and a polling connector; see [built-in connectors](#built-in-connectors) for more info.

VCS connections are done as an initial setup step, where a cluster operator provides credentials and the connector is responsible for creating a long-lived connection and
keeping it up-to-date over time. (For example, if a platform connects to a GitHub repo using deploy keys, this connection must be updated if the repo admin rotates their deploy keys.)
Repo connections can then be referenced in multiple Workflows. For example, this would allow a user to write a CI Workflow and a CD Workflow for the same repo, without specifying
repo credentials in each Workflow.

To create a new connection to a repo, the cluster operator creates an instance of a new CRD, a `GitRepository`. Each connector controller filters for the `GitRepository`s of
interest, and updates their status. <!--TODO: How to prevent multiple controllers from updating the status of one GitRepository?-->
Filtering is done based on the label "workflows.tekton.dev/connector".
Like `CustomRun`s, the `GitRepository` CRD contains spec fields where users may specify arbitrary configuration for repo connections, and status fields where controllers can provide
arbitrary information.


Examples of setup steps:




### Example: Webhook 

### Example: New GitHub App

### Example: Existing GitHub App

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: GitRepository
metadata:
  name: webapp
  namespace: ops
  labels:
    workflows.tekton.dev/connector: github-app
spec:
  url: https://github.com/small-startup/next-big-idea-app
  vcsType: github
  customFields:
    appID: 12345
    webhookSecretRef:
      name: github-webhook-secret
      key: secret-token
status:
  conditions:
  - type: Ready
    status: "True"
    reason: InstallationComplete
  customFields:
    installationID: 67890
```

In this example, the "github-app" connector is responsible for installing app "12345" on the next-big-idea-app repo. 


### Example: Deploy Keys

### Example: Polling

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: GitRepository
metadata:
  name: webapp
  namespace: ops
  labels:
    workflows.tekton.dev/connector: polling
spec:
  url: https://github.com/small-startup/next-big-idea-app
  customFields:
    pollingInterval: 5m

status:
  conditions:
  - type: Ready
    status: "True"
    reason: InstallationComplete
  customFields:
    installationID: 67890
```

### Org-level connections

In addition to repo connections, VCS connections may happen at the org level. For example, an organization may maintain multiple repos, and want to configure credentials
only once for the organization rather than once for each repo.

To create an organization-level connection, cluster operators create an instance of a `GitOrganization` CRD. <!-- TODO: Naming. -->
Connectors should indicate whether they support `GitRepository`s, `GitOrganization`s, or both.
This work is scoped for a follow-up milestone of the project.

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: SCMConnection
metadata:
  name: tekton-org
  namespace: ops
  labels:
    workflows.tekton.dev/connector: github-app
spec:
  orgName: tektoncd
  vcsType: github
  # robotAccount: tekton-robot
  repos:
  - pipeline
  - plumbing
  - chains
  customFields:
    personalAccessToken:
      secretName: github-pat
      secretKey: token
status:
  conditions:
  - type: Ready
    status: Unknown
    message: "Waiting for user to install app on organization"
  customFields:
    foo: bar
```

### Built-in connectors

### Referencing Repos in Workflows

Users should be able to declare one or more repos in their Workflow. Initially, Workflows will have references to `GitRepositories` which must already exist.
Later, we can add support for inline repository configuration, where the Workflow will create a `GitRepository` if it does not yet exist.
Workflows should be able to reference `GitRepositories` in other namespaces, to make it easier for cluster operators to set up the `GitRepository` while app developers reference it in their workflows.
Cross-namespace references are also beneficial for use cases where you might want to have (for example) a CI namespace and a CD namespace.

### Generating events from repositories

Event configuration should be separate from repo connection configuration, because the same repository can be used in multiple Workflows that trigger based on different events.
In addition to `GitRepositories`, repo connectors are expected to respond to `RepositoryEvents` CRDs. An example `RepositoryEvent` is as follows:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1
kind: RepositoryEvent
metadata:
  name: pull_request
spec:
  types:
  - check_suite

```

TODO: What happens if RepositoryEvents have different sinks?
TODO: What happens if sink is not a public URL?

When a user specifies Triggers in their Workflow, the controller will create an instance of an Event CRD

## Milestones

1. Event triggering
- User can create a connection to a repository and send events to an EventListener from this repository.
- An EventListener can be triggered by a CronJob. 
2. Notifications

3. Reduced verbosity for filters and interceptors
4. concurrency controls

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

## Implementation Pull request(s)

<!--
Once the TEP is ready to be marked as implemented, list down all the Github
Pull-request(s) merged.
Note: This section is exclusively for merged pull requests, for this TEP.
It will be a quick reference for those looking for implementation of this TEP.
-->

## References (optional)

<!--
Use this section to add links to GitHub issues, other TEPs, design docs in Tekton
shared drive, examples, etc. This is useful to refer back to any other related links
to get more details.
-->
