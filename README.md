# Updatecli policies for CRD version bumping

`updatecli-policies` contains policies usable by Updatecli to bump the version of new Kubernetes CRD schemas.

It is part 3 of a 3-part setup comprised of:

* [ContainerCraft/projen-pulumi-crd-sdks](https://github.com/ContainerCraft/projen-pulumi-crd-sdks) - A
  [Projen](https://projen.io/) project type that manages projects handling Pulumi SDKs for Kubernetes Custom Resources.
* [ContainerCraft/git-tag-replay](https://github.com/ContainerCraft/git-tag-replay) - A Github Action that
  replays Git tags from an upstream repository to a downstream repository.
* [ContainerCraft/updatecli-policies](https://github.com/ContainerCraft/updatecli-policies) - A repository
  containing [Updatecli](https://www.updatecli.io/) policies to update dependencies when new versions are
  detected by `git-tag-replay`. **(this repository)**
