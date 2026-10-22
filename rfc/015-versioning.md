# RFC 015: Versioning Platform Mesh

| Status  | Collecting Feedback   |
|---------|------------|
| Author  | @xrstf |
| Created | 2026-10-22 |
| Updated | 2026-10-22 |

## Overview

This RFC proposes a new system to version and release Platform Mesh.

## Background

In ye olden days, Platform Mesh consisted of a large number of components, each homed in their own git repository, like the account-operator, security-operator, rebac-authz-webhook etc. To make a new Platform Mesh release, we would collect the most recent release of each component at the time of making the PM release.

Now after RFC 013 (Repo Consolidation) has been completed, most of these standalone repositories have been merged into a single monorepo, including all their existing versions. So for example account-operator v1.2.3 would have become the new git tag `account-operator/v1.2.3` in the monorepo.

Having all those different components share a single repository, but keep their own versioning, introduces a lot of complexity and difficulty for handling releases.

### Release Branch Model

This graph shows a very simple git graph, representing a repository that makes use of classic release branches. This is the same concept that most other open source projects follow, where around the time a new minor release is made, a branch is created to receive further fixes and improvements.

![Release Branch Git Graph](./assets/015-release-branches.png)

In such a model, new changes are generally made only on the `main` branch (exceptions exist, but the general flow should always be "commit to main, _then_ backport into the relevant release branches").

The goal of this RFC is to get us to this exact shape, where improvements and fixes can easily be backported and quickly released to consumers. For Platform Mesh, it could look like this:

![Git graph showing backport procedure](./assets/015-backporting.png)

In the graph above, we can see that after Platform Mesh v0.6.0 was released, a release branch was created. Later, multiple changes and new features and improvements for various components land on the `main` branch, from where they are cherrypicked into the release branch. Notably, this cherrypick is mostly automatable (Prow for example has a `/cherrypick` command to automatically create PRs) since _no content of the cherrypicked commit has to be adjusted_ (minus potential conflicts).

However, getting us to this structure has some obstacles to overcome.

### Intermittent Releases

Since we merged all those component repositories into one, we also merged all versioning information into a single repository. For example the account-operator's `v0.8.1` tag became `account-operator/v0.8.1` in the monorepo.

![Release branch model for Platform Mesh](./assets/015-platform-version-situation.png)

We can see that after Platform Mesh 0.5.2 was shipped, more changes happened on the `main` branch. Some of those changes even bump a component's version (like the 2nd commit after v0.5.2, where the account-operator was bumped to v0.8.0 (meaning the `v0.8.0` inside the circle stands for the Git tag `account-operator/v0.8.0`)). Not all changes increase the version though.

It doesn't matter here whether the version is written into a `Chart.yaml` or applied via Git tag or even both. The problem is the _linear_ situation.

Let's look closer and focus only on a single component. All other commits that change other components are simply ignored in the following graph:

![account-operator changing graph](./assets/015-account-operator-explicit.png)

Here the problem hopefully becomes a bit clearer: The commits on the main branch already "consume" version numbers, since we version each component independently. Now, after a few days and a few changes, we identified two commits to backport since we intent to ship them to consumers. At this point we run into 2 separate issues:

1. For Helm charts, the chart version is usually kept inside the `Chart.yaml`. This means that when doing a `git cherrypick`, that version is also copied over to the destination. Suddenly there are two commits (the original and the cherrypick) that claim to be the same Helm chart version. This clearly cannot be and would mean that everyone who does a cherrypick needs to manually edit the picked commit and assign a new version. Not only that, the person/agent who does it also first has to figure out which version is even correct to assign.

   This can be solved by not versioning the charts individually anymore. Then the `Chart.yaml` does not contain real versions anymore and can
   be blindly cherrypicked.

2. Even if no Helm charts are involved here and so no version is written down in the codebase, whenever a cherrypick happens, we need to determine the correct version to assign to the cherrypick. This is because a Platform Mesh release is an aggregate over the component versions, so to make sure a change is included in a Platform Mesh release, it itself needs to be released first, on its own. So the cherrypick _must_ eventually get a version assigned. The question just is: Which version?

This problem is not solved by jumping to new minor versions, as shown in the following example:

![account-operator changing graph](./assets/015-account-operator-explicit-minor.png)

The same problems arise again.

Likewise, using tags does not resolve this problem. It certainly makes it less harsh, since cherrypicks can be automated more, but the problem of figuring out which versions to change to what remains:

![account-operator changing graph](./assets/015-account-operator-tagged.png)

### Per-Component Release Branching

The "solution" to this problem is apparent: release branches. Just a lot more of them. A working model could look like this, where every single component gets its own release branches:

![account-operator changing graph](./assets/015-per-component-branching.png)

This way, changes from the `main` branch can cleanly be separated out. You can see that the first change after version 0.6.0 was released would immediately bump the account-operator (on `main`) to 0.8.0. This is because the 0.7 lineage is now "owned" by the release branch `account-operator-0.7`. In effect, every component would need to make a minor version jump whenever a new Platform Mesh minor version is released (since at that point in time, we "freeze" the components' minor versions and open release branches).

Do note that still, during a cherrypick, the author/agent would have to adjust the contained version (in the `Chart.yaml` from 0.8.0 to 0.7.1) by hand. This requires them to know what the most recent patch release on 0.7.x is. As before, this requirement to change the `Chart.yaml` goes away if we decided not to have versions in the `Chart.yaml`, _but_ the manual process of figuring out what version to assign remains, even if Git tags are used.

When a new Platform Mesh release is then to be made, _all_ component release branches are merged back into Platform Mesh's release branch (i.e. `account-operator-0.7` is merged into `release-0.6`, then `security-operator-0.15` is merged into `release-0.6` etc.).

## Summary

We can clearly see that the current situation we maneuvered ourselves into is not working anymore. We are as of now incapable of making patch releases for older Platform Mesh releases and the approach of versioning each component/chart independently is causing a lot of churn and complexity.

## Why version something in the first place?

At this point, we should step back and ask ourselves a question: In this example that we've already seen:

![account-operator changing graph](./assets/015-account-operator-explicit.png)

When we committed what would become account-operator v0.7.1, someone decided that this commit would be tagged with a version. For what reason though?

Clearly: We tag something so an artifact gets built and someone else out there can consume it if they choose to do so.

So by tagging we acknowledge that something is "blessed", "good to be used". But how is it now going to be used?

Look at Keith, who installed Platform Mesh today. He sees that version 0.5.2 is the latest, so he installs it, but then he notices that there is already account-operator 0.7.1 (his Platform Mesh 0.5.2 still has account-operator 0.7.0). So now Keith is wondering "Can I use 0.7.1? Is it compatible? Surely, right?"

This leads to even more uncertainty:

a) Can someone use it? @brsmnv already brought up the topic of compatibility matrixes: If we randomly ship random components, how will anyone figure out which of these are even applicable to them? _Especially_ since Platform Mesh 0.5 ships with components in versions 0.7, 0.59, 1.5.2, etc. -- all over the place.

b) If we decided to ship account-operator 0.7.1, why did we not actually release it? There is no changelog, no release notes, nothing. Just a tag and a hardy "Yolo, have fun with it!" from us.

c) If we decided that a commit is generally useful, stable and consumable by admins, why would we not simply ship it as a new Platform Mesh release? Shipping it as a new release has all the benefits: A changelog, a nice GitHub release, visibility, all the good stuff.

Just randomly tagging components will build artifacts. But ... then nobody can really make use of them.

## Proposal

Having discussed all of this, there can be only one solution: Version Platform Mesh as one coherent application. One version number to rule them all. In detail this would mean:

* Platform Mesh releases are made by tagging a commit with `vX.Y.Z`.
* Individual components are not tagged or shipped on their own anymore.
* To make a change available in older Platform Mesh releases, they just need to be cherrypicked with git. Since no version numbers are included in the codebase, this is trivial.

