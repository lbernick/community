---
status: proposed
title: Common Repository Configuration
creation-date: '2021-11-19'
last-updated: '2021-11-29'
authors:
- '@sbwsg'
---

# TEP-0095: Common Repository Configuration
---

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
  - [Separate Host Connection and Repository Configuration](#separate-host-connection-and-repository-configuration)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [Implementation Pull request(s)](#implementation-pull-requests)
- [References (optional)](#references-optional)
<!-- /toc -->

## Summary

Several projects and designs currently in-flight in the Tekton ecosystem
are related to or directly working with source repository-related information.
These include:

- [Scheduled and Polling Runs in Tekton](0083-scheduled-and-polling-runs-in-tekton.md)
- [Workflows](../working-groups.md#workflows)
- [Pipeline-as-Code](https://github.com/openshift-pipelines/pipelines-as-code)
- [Remote Resource Resolution](0060-remote-resource-resolution.md)

Each of these projects are dependent on information from
repositories - either querying for metadata, fetching content, listening for
events, recording relatational data about them or publishing notifications to
them.

Given a shared interest in this data across multiple projects it might
make sense to define an object that allows an operator to specify common
repo configuration only once, rather than per-project. Over time other projects
and tools could also leverage this information.

## Motivation

Source repositories hold a unique position in CI/CD, not only as
versioned storage for code but also as a source of events, target of
notifications like test status updates, source of tasks and pipelines,
"single pane of glass" for developers, and so on. It therefore makes
sense that several Tekton components are designed to work directly with
source repos, either reading or writing to them in some way.

The core motivation for this TEP is to model source repos explicitly in
Tekton's domain so that components that work with them have a shared
set of configuration describing them.

### Goals

The precise scope of this TEP depends on results of experimenting with
this design prior to defining a proposal.

- Decide if there is a common subset of repository data that would be
  useful across multiple projects.
- Design a format to store and record that common data.
- Publish developer-facing docs for projects to use.
- Publish user-facing docs for operators/cluster admins/app teams to use.

### Non-Goals

- Mutable `status`. This TEP is not aiming to offer a way for concurrent
  processess to modify shared state or communicate about state changes.
  Components can read and respond to the shared Repository configuration
  data but shouldn't expect access to write back to it.
- A standard for inferring or storing provenance information related to
  source code or repositories. This kind of data modelling is kept to
  the domain of individual components rather than mandated by the common
  repository configuration.
- Defining interfaces between components for repository data, for
  example in Pipeline params. Instead this TEP can lean on the work from
  the dictionary params TEP, which sets the way for those interfaces to
  be defined.

### Use Cases

These use-cases were primarily taken from [the Workflows WG meeting
notes](https://docs.google.com/document/d/1di4ikeVb8Mksgbq4CzW4m4xUQPZ2dQMLvK1VIJw7OQg/edit#heading=h.i22qgrdiutdu)
on Nov 16, 2021.

- A component could read the type, provider and API endpoint info from
  the common repo config to determine how to convert `CloudEvents` from
  Pipelines into status updates on that repo.
- A component could record history related to a Repository object (e.g.
  the set of PipelineRuns that have been started as a result of events from
  that Repo) and then provide services based on that history (e.g. "clean up
  all the PipelineRuns started for this Pull Request on that Repo" or
  "return a summary of all PipelineRuns started for this repo in the
  past 7 days").
- A UI component could look up the website URL to link to in order
  to view detailed pull request information in that repo.
- A component could observe Repository objects and automate setting up
  `EventListeners` and `Triggers` for it.
- A component could keep a cache of the contents of a repo pointed at by
  a Repository object for use as a source of Pipelines and Tasks.
- A component could read info from the common config to decide which branches
  or commits to poll for updates on, and which API endpoints to hit.
  - Con: This might end up being too-specific data to include in common
    configuration since it is likely to be specific to individual
    triggers.

In all of these instances each of the services/components could
independently require configuration of specific repos. The value of
a shared repository object would be in having a single place for this
configuration info to live.

## Requirements

<!--
Describe constraints on the solution that must be met. Examples might include
performance characteristics that must be met, specific edge cases that must
be handled, or user scenarios that will be affected and must be accomodated.
-->

## Proposal

Create a new Repository CRD:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1  # TODO
kind: Repository
metadata:
  name: pipelines
  namespace: ops
spec:
  url: https://github.com/tektoncd/pipeline
  scmType: github
```

A user can also choose to specify a repo name and organization instead of a URL:

```yaml
apiVersion: workflows.tekton.dev/v1alpha1  # TODO
kind: Repository
metadata:
  name: pipelines
  namespace: ops
spec:
  org: tektoncd
  repo: pipeline
  scmType: github
```

### Repo connections

Repo connections aren't in scope for this proposal. There are many different ways of "connecting" to a given repo, including:
- Cloning it once as part of a CI/CD pipeline
- Regularly polling it for updates
- Setting up a webhook on it to subscribe to events
- Installing a GitHub App on it
- "Connecting" to the organization the repo is associated with, and adding the repo to the organization "connection"

This logic is better handled by the project using the Repository CRD.
This also means that configuration for events, files in the repo, or git revisions shouldn't be included in the repo CRD.

### Authentication

It's very likely that cluster operators would be interested in providing credentials for a repo only once, and then referring to that
repo many times in different Tekton resources. This makes authentication configuration a good candidate for including in a repo CRD.
However, there are some challenges:

1. Different SCMs have different ways of handling authentication, and authentication may differ based on how the repo is "connected".
For example:
- The repo could be cloned or polled using SSH keys.
- A user could set up a webhook using a personal access token for their own account or a robot account.
- A user could be directed through an OAuth flow to install a GitHub App on the repo.
- A user could connect the repo to an existing GitHub App using the app's private key.

2. Tekton functionality that uses repo connections will have to grapple with whether users that don't have access to these credentials
should be able to create Tekton resources referencing these credentials.

For example: a cluster operator could configure SSH keys (stored in a secret) for a repo that a developer doesn't have access to.
If the developer has access to the Kubernetes cluster, and creating a repo CRD with credentials means the repo could be used in PipelineRuns,
the developer could theoretically create a PipelineRun that uploads the contents of the repo somewhere they do have access to.

TODO: This might not actually be a problem-- there might be plenty of these situations already existing with Tekton that just aren't a problem
in practice. Need to think through scenarios.

3. We should ideally create a syntax for referring to secrets that doesn't rely on Kubernetes-isms, in keeping with Tekton's
design principles on [conformance](https://github.com/tektoncd/community/blob/master/design-principles.md#conformance).
This makes it easier to create non-Kubernetes-based Tekton implementations.

One option would be to add all the common SCM authentication methods to the repo CRD, as references to secrets.
This is similar to how the Pipelines [Git resolver](https://github.com/tektoncd/pipeline/blob/main/docs/git-resolver.md)
is configured. For now, authentication will be left out of scope for an initial version of this proposal
until we have some examples of projects using the Repository CRD.

### Organizations

Many SCMs have the concept of an organization to manage multiple repos for a team.
The following SCMs call this concept an "organization":
- GitHub and GitHub Enterprise
- GitLab
- Gerrit
- Gitea

The following SCMs have modified concepts of organizations:
- Bitbucket, which has "Teams"
- Azure Repos, Google Cloud Source Repositories, and AWS CodeCommit, which all manage organizations through their broader platforms rather than
an "organization" concept specific to their SCM products.

The following SCMs don't have concepts of "organizations":
- SourceHut
- SourceForge

In the [JetBrains State of the Developer Ecosystem 2019](https://www.jetbrains.com/lp/devecosystem-2019/team-tools/),
the most commonly used SCMs by far were GitHub, GitLab, and Bitbucket. 
In addition, Pipelines has already added support for repository name + organization name syntax (as an alternative to a URL) in its
[Git resolver](https://github.com/tektoncd/pipeline/blob/main/docs/git-resolver.md).
Therefore, the term "organization" is general enough for most use cases.

In the future, we could explore adding an "Organization" CRD or similar, to help in configuring connections to SCM organizations
instead of individual repos.

### Terminology

- "SCM" is preferred here over "VCS", because even though the terms are often used interchangeably, the intention here is to refer to the
source control manager hosting the repo rather than the version control system used to make changes to the repo.
- "Repository" is preferred here over "GitRepository", as not all source code repositories use Git as their version control system.
One potential downside is that "Repository" could be unintentionally interpreted to include artifact repositories.
(Expanding scope to artifact repositories is a potential avenue for future exploration.)

## Example Usage: Workflows

In this example, the CI pipeline is stored in a different repo than the repo to run CI on.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Workflow
spec:
  repos:
  - name: pipelines
  - name: plumbing
  events:
  - repo: pipelines
    type: pull_request
  pipelineRun:
    pipelineRef:
      resolver: git
      params:
      - name: url
        value: $(repos.plumbing.url)
      - name: revision
        value: 12345
      - name: pathInRepo
        value: tekton/pipelines/ci-pipeline.yaml
```

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


## Design Evaluation
<!--
How does this proposal affect the reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

The Repository CRD doesn't do very much on its own. It's not yet clear how much value it will add compared to an inline repo definition.

## Alternatives

### Separate Host Connection and Repository Configuration

- Maintain multiple schemas / config types to avoid repetition. E.g.
  host information is likely to be same across many repositories and
  might therefore be a good candidate to have its own schema that
  repository configuration can reference.

### Couple the Repo CRD to repo connections

One example of this pattern is FluxCD's [GitRepository](https://fluxcd.io/flux/components/source/gitrepositories/),
which polls a Git repository for changes:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
spec:
  interval: 5m0s  # Frequency to poll repo for updates
  url: https://github.com/tektoncd/pipeline
  ref:
    branch: master  # The branch to monitor for changes
  secretRef:
    name: git-ssh-creds
```

Another example is Knative's [GitHub EventSource](https://github.com/knative/docs/tree/main/code-samples/eventing/github-source),
which sets up a webhook on a GitHub repo and sends events to a connected sink:

```yaml
apiVersion: sources.knative.dev/v1alpha1
kind: GitHubSource
metadata:
  name: githubsourcesample
spec:
  eventTypes:
    - pull_request
  ownerAndRepository: tektoncd/pipeline
  accessToken:
    secretKeyRef:
      name: githubsecret
      key: accessToken
  secretToken:
    secretKeyRef:
      name: githubsecret
      key: secretToken
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: github-message-dumper
```

## Implementation Pull request(s)

<!--
Once the TEP is ready to be marked as implemented, list down all the Github
Pull-request(s) merged.
Note: This section is exclusively for merged pull requests, for this TEP.
It will be a quick reference for those looking for implementation of this TEP.
-->
