# Pulumi CRD SDK Policy

Uses a new version of an upstream Kubernetes CRD and updates the required files to generate a new version of 
the Pulumi SDK for the CRD.

## Overview

This policy runs a single Updatecli pipeline that:

- clones a source git repository (configured under `src`)
- Changes the version of the CRD in `.projenrc.json`
- reruns `npx projen` to update the required build files
- reruns `make clean build` to regenerate the Pulumi SDKs for the new version of the CRD YAML descriptors
- commits the changes to the source repository
- opens a pull request against the destination repository

This is useful for scaling the build setup of Pulumi Kubernetes CRDs that are generated
across multiple repositories from a single canonical source.

## Requirements

- `updatecli` CLI installed
- configuration of the upstream Kubernetes CRD project
- SCM credentials for the destination repository

## Supported SCM Backends

This policy always requires SCM configuration for the destination repository. It supports:

- `github`

It relies on the [default environment variables](https://www.updatecli.io/docs/plugins/scm/github/#_authentication) 
to be available for authentication against Github.

<!-- Still to review the content below -->

## Policy Configuration

### Available Input Values

The default `values.yaml` exposes these top-level inputs:

- `pr.automerge`: controls pull request automerge behavior (default: `false`)
- `pr.labels`: labels applied to pull requests
- `scm`: destination repository SCM configuration
    - `kind`: SCM provider (`github`)
    - `commitusingapi`: commit via the provider API rather than git (default: `true`, GitHub only)
    - `user`: git commit author name
    - `email`: git commit author email
    - `owner`: repository owner or organization
    - `repository`: repository name
    - `branch`: target branch
    - `username`: SCM username for authentication
    - `commitmessage`: structured commit message (`type`, `title`, `body`, `scope`, etc.)
- `src`: source repository
    - `url`: git clone URL of the source repository
    - `branch`: branch to read files from
- `upstream`: upstream repository
    - `kind`: source of releases (`githubrelease`)
    - `owner`: repository owner or organization
    - `repository`: repository name
    - `env_version`: environment variable containing the value for the next released version (`NEXT_VERSION`)

### Example Values

The following example syncs shared config files from the `updatecli/updatecli` repository into the destination repository:

```yaml
upstream:
  kind: github
  owner: cert-manager
  repository: cert-manager

scm:
  kind: github
  owner: experimentale
  repository: pulumi-crd-certmanager
  username: ringods
  branch: main
```

## How It Works

This policy renders a manifest with the following structure:

```yaml
name: 'Bump upstream CRD version'
pipelineid: <pipelineid>

scms:
  default:
    kind: github
    spec:
      owner: <scm.owner>
      repository: <scm.repository>
      username: <scm.username>
      branch: <scm.branch>

sources:
  upstream:
    kind: githubrelease
    spec:
      owner: <upstream.owner>
      repository: <upstream.repository>
      versionfilter:
        kind: semver
        pattern: <requiredEnv .upstream.env_version>
    transformers:
      - trimprefix: "v"

targets:
  apply-to-projen:
    name: "Update CRD version in .projenrc.json"
    kind: json
    sourceid: upstream
    scmid: default
    spec:
      engine: "dasel/v2"
      file: ".projenrc.json"
      key: "latestVersionOnBranch"
  regenerate-workflows-and-sdks:
    name: "Regenerate SDKs"
    dependson: ["target#apply-to-projen"]
    dependsonchange: true
    kind: shell
    scmid: default
    disablesourceinput: true
    spec:
      command: |
        mise trust
        npm install
        npx projen
        make clean build

actions:
  default:
    title: Bump upstream CRD version
    kind: github/pullrequest
    scmid: default
    spec:
      labels:
        - "dependencies"
```

The `default` SCM is used for writing and opening pull requests.

## Quick Usage

### Using from an OCI Registry

Consume the published bundle directly from a registry:

```sh
updatecli manifest show --values values.yaml ghcr.io/ContainerCraft/updatecli-policies/crd-pulumi-sdk@v0.1.0
```

```sh
updatecli pipeline diff --values values.yaml ghcr.io/ContainerCraft/updatecli-policies/crd-pulumi-sdk@v0.1.0
```

```sh
updatecli pipeline apply --values values.yaml ghcr.io/ContainerCraft/updatecli-policies/crd-pulumi-sdk@v0.1.0
```

## Authentication

Authenticate to your OCI registry before pushing or pulling bundles:

```sh
docker login "$OCI_REGISTRY"
```

Export the SCM token before running Updatecli:

```sh
export GITHUB_TOKEN="ghp_xxxx"
```

## Publish

Publish this policy bundle to an OCI registry. The `version` field in `Policy.yaml` defines the bundle tag:

```sh
updatecli manifest push \
  --config updatecli.d \
  --values values.yaml \
  --policy Policy.yaml \
  --tag "$OCI_REGISTRY/file" \
  .
```

After publishing, reference the bundle by tag:

```sh
updatecli manifest show "$OCI_REGISTRY/crd-pulumi-sdk:0.1.0"
```

## Troubleshooting

### Pull requests are not created

1. Verify that the authentication environment variables for Github App authentication is exported and has write 
   access to the destination repository.

2. Confirm that `scm.owner`, `scm.repository`, and `scm.branch` are set correctly.

3. Check that `scm.kind` matches your destination SCM provider.

## Related Documentation

- Updatecli docs: <https://www.updatecli.io>
- Compose docs: <https://www.updatecli.io/docs/core/compose/>
- Sharing and reuse: <https://www.updatecli.io/docs/core/shareandreuse/>
- Updatecli policies repository: <https://github.com/updatecli/policies>
