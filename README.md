# ci-security-starter

Reusable security checks for the pull request, before it is merged. The same four jobs run on GitHub Actions and GitLab CI.

- **gitleaks** scans git history for secrets.
- **Trivy** scans the filesystem, and the container image built from a Dockerfile.
- **kubeconform** checks Kubernetes manifests against the schema for a pinned Kubernetes version.

[Security that arrives at the end of the sprint is already late](https://shafie.dev/notes/security-before-production/) is the longer argument for keeping the list this short, and for running it before production.

This repository runs those checks on [`sample/`](sample/). The workflow you copy is the workflow CI runs.

## Checks

| Job | What it looks at | The job fails when |
| --- | --- | --- |
| gitleaks | The whole git history | A secret matches |
| Trivy filesystem | Language packages and lockfiles in the worktree | A fixed HIGH or CRITICAL vulnerability |
| Trivy image | The image built in the job | A fixed HIGH or CRITICAL vulnerability |
| kubeconform | Manifests you point it at | The manifest does not match the pinned schema, including unknown fields (`-strict`) |

Unfixed Trivy findings do not fail the job. Pass `ignore_unfixed: false`, or a different `severity`, when you have agreed a stricter line. A line the team cannot meet gets waived, and then you do not have a line.

kubeconform checks schema. A Deployment can be valid for that schema and still be a bad idea. Pod security, network policy, and admission sit outside this job.

The scan runs the gitleaks CLI. gitleaks-action asks organization accounts for a license, so the workflow stays on the CLI. It is the same detector.

Trivy and the other tools are release tarballs, checked with sha256. `actions/checkout` is the only third-party action, pinned to a commit.

## GitHub Actions

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) calls the reusable workflows with a relative path, so a pull request to this repository tests the workflows in that pull request.

From another repository, call them by name and pin the ref. `@main` below is only so the example stays readable. It moves. Pin a tag or a commit when you depend on this.

```yaml
name: Security

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  secrets:
    uses: xoz-e/ci-security-starter/.github/workflows/secrets.yml@main

  trivy:
    uses: xoz-e/ci-security-starter/.github/workflows/trivy.yml@main
    with:
      dockerfile: Dockerfile
      context: .
      image: sample:ci

  manifests:
    uses: xoz-e/ci-security-starter/.github/workflows/manifests.yml@main
    with:
      manifests: deploy/
      kubernetes_version: "1.37.2"
```

Leave `dockerfile` empty to skip the image job. The filesystem scan still runs. `manifests` is required.

The reusable workflows are `workflow_call` only. On this repository the status check names are `secrets / gitleaks`, `trivy / filesystem`, `trivy / image`, and `manifests / kubeconform`. A failing check shows on the pull request. It blocks the merge once you mark it required in a ruleset or branch protection rule.

## GitLab CI

Include [`gitlab/security.gitlab-ci.yml`](gitlab/security.gitlab-ci.yml). [`.gitlab-ci.yml`](.gitlab-ci.yml) is the caller for this repository.

```yaml
include:
  - project: xoz-e/ci-security-starter
    file: /gitlab/security.gitlab-ci.yml
    ref: main
    inputs:
      dockerfile: Dockerfile
      image_context: .
      image_name: sample:ci
      manifests: deploy/
      kubernetes_version: "1.37.2"
```

Pin `ref` the same way you pin the GitHub workflow. An empty `dockerfile` skips the image job. An empty `manifests` skips kubeconform.

The image job builds with Docker-in-Docker (`docker:28.5.2`), so the runner has to allow a privileged service. The GitHub workflow does not need that: the hosted runner already has a Docker daemon. Jobs run in the `test` stage.

## Run the same commands locally

Versions are in the table below.

```bash
gitleaks detect --source . --redact --verbose --exit-code 1
trivy fs --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed --scanners vuln .
docker build -t ci-security-starter-sample:ci -f sample/Dockerfile sample
trivy image --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed --scanners vuln ci-security-starter-sample:ci
kubeconform -strict -summary -kubernetes-version 1.37.2 sample/k8s
```

## Pre-commit

[`.pre-commit-config.yaml`](.pre-commit-config.yaml) runs gitleaks 8.30.1 on staged changes, the same version CI runs.

The hook is there so you see the failure before you push. `git commit --no-verify` skips it, and so does any clone where the hook was never installed. The pull request job scans the history that would actually be merged. That job is the one to enforce.

Install [pre-commit](https://pre-commit.com), then run `pre-commit install` in the clone.

## Sample

[`sample/`](sample/) is a Go process with no third-party modules, a multi-stage Dockerfile whose runtime image is `scratch`, and one Service plus Deployment. CI builds `ci-security-starter-sample:ci` and scans that image. The image is not pushed anywhere.

The manifest sets a non-root user, a read-only root filesystem, and dropped capabilities so the sample is not a bad habit. kubeconform does not check any of that. It checks that the YAML is valid for Kubernetes 1.37.2.

## Credentials

These jobs do not talk to a cloud API. When a later job has to, prefer a short-lived credential from OIDC over a long-lived access key. [docs/oidc.md](docs/oidc.md) is the vendor-neutral note, with brief pointers for AWS, GCP, and Azure.

## Pinned versions

Bump a tool by changing its version and checksum together. The defaults live in the workflow files and in `gitlab/security.gitlab-ci.yml`. Callers can override the inputs.

| Tool | Version | Pinned as |
| --- | --- | --- |
| gitleaks | 8.30.1 | release tarball, sha256 |
| Trivy | 0.76.2 | release tarball, sha256 |
| kubeconform | 0.8.0 | release tarball, sha256 |
| Kubernetes schema | 1.37.2 | `kubernetes_version` input |
| actions/checkout | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
| Go (sample build) | 1.27.4 | `golang:1.27.4-alpine3.24` |

## What this does not do

These checks are a gate on the pull request. They are not a security program.

- gitleaks reads git. It does not watch build logs, a registry, or a secrets manager, and it does not rotate the credential it finds. If a secret has already been committed, rotate it. Deleting the line leaves it in history.
- Trivy, as wired here, reports known vulnerabilities in the worktree and in the image just built. It is not a reachability review, it does not sign the image, and it does not scan a running cluster.
- kubeconform checks a manifest against one Kubernetes schema. It will accept a manifest that is valid and still wide open.
- Nothing here is application SAST, a review of infrastructure changes, threat modeling, or runtime detection.
- pre-commit is local. Anyone can skip it. The server-side job is the one that can block a merge, and only after you make it a required status check.

A longer list of scanners, turned on at once, is how findings turn into noise. The reason to stop at this set is in [Security that arrives at the end of the sprint is already late](https://shafie.dev/notes/security-before-production/).

## License

[MIT](LICENSE). Copyright (c) 2026 Hamidreza Shafie.

## فارسی

این مخزن یک شروع کوچک است: چند security check که قبل از merge شدن pull request اجرا می‌شوند.

gitleaks تاریخچه git را برای secret می‌گردد. Trivy فایل‌های repo و container image را scan می‌کند. kubeconform مانیفست Kubernetes را با schema همان نسخه‌ای مقایسه می‌کند که در workflow مشخص شده، و بیشتر از schema را نگاه نمی‌کند. روی GitHub Actions به‌صورت reusable workflow صدا زده می‌شوند و در GitLab CI با include وارد pipeline می‌شوند.

pre-commit برای gitleaks اینجاست تا همان خطا را قبل از push ببینید. با `git commit --no-verify` می‌شود از hook رد شد. merge را job سمت سرور متوقف می‌کند، به شرطی که required status check باشد.

این workflowها به cloud وصل نمی‌شوند. وقتی job باید image را push کند یا API صدا بزند، access key را داخل secretهای CI نگذارید. با OIDC همان job یک credential کوتاه‌عمر می‌گیرد که به repo و ref محدود است. جزئیات در [`docs/oidc.md`](docs/oidc.md) است.

سبز بودن این چک‌ها یعنی secret داخل git نمانده، image روی آستانه HIGH و CRITICAL گیر نکرده، و مانیفست از نظر schema معتبر است. SAST، runtime، policy روی cluster، و image signing اینجا نیستند. این یک برنامه امنیتی کامل نیست. استدلال بلندتر را اینجا نوشته‌ام: <https://shafie.dev/notes/security-before-production/>

حمیدرضا شفیعی
