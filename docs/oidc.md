# Short-lived credentials

The checks in this repository scan git, the worktree, and a local image. They do not need a cloud account.

A later job often does. It pushes an image, runs a plan, or calls a cloud API. That job needs a credential. How you issue it matters more than which scanner failed last week.

## Long-lived keys

An access key in a CI secret is a password that outlives the job. It is copied onto the runner. It shows up in a debug log the day someone turns on trace. It keeps working after the person who minted it has left. Rotation is a reminder on a calendar. If the key leaks, the blast radius is whatever that key could do, until someone notices and disables it.

Deleting the key from the workflow is not the same as disabling it at the cloud provider.

## OIDC

OpenID Connect flips the direction. The CI system mints a short-lived token that says which repository, ref, and workflow the job is. The cloud account is configured to trust that issuer, and only for a subject you name. The job exchanges the token for cloud credentials that expire in minutes or hours. Nothing you would call an access key is stored in CI.

The trust condition is the control. A token for one repository on `main` should not be accepted from a pull request, and a job that pushes to one registry should not assume a role that can administer the account.

## What to configure

1. Register the CI issuer in the cloud account. GitHub's issuer is `https://token.actions.githubusercontent.com`. GitLab.com's is `https://gitlab.com`. A self-managed GitLab uses its own URL.
2. Bind that trust to a subject. Do not trust every repository in the organization, and do not trust every ref if only one ref deploys.
3. Give the role the API calls that job makes, and nothing around them.
4. On GitHub Actions, the job needs `id-token: write` or it cannot request a token. It still does not need a cloud key:

   ```yaml
   permissions:
     contents: read
     id-token: write
   ```

5. On GitLab, declare an `id_tokens` entry whose audience is the one the cloud provider expects, and pass that token into the exchange.
6. Remove the long-lived key after the OIDC path works. Leaving both in place means the key is still the real access path.

A pull request, including one from a fork, should not be able to assume the role that deploys. Keep cloud credentials out of the scan jobs in this repository. They have nothing to do with a cloud API.

## AWS

Register the CI issuer as an IAM OIDC identity provider. Add a role whose trust policy allows `sts:AssumeRoleWithWebIdentity` only when the token's `sub` matches the repository and ref that may deploy. In GitHub Actions, `aws-actions/configure-aws-credentials` with `role-to-assume` performs the exchange. Pin that action to a commit SHA, the same way this repository pins `actions/checkout`.

GitHub tokens use the audience `sts.amazonaws.com`. A subject for a branch looks like `repo:OWNER/REPO:ref:refs/heads/main`. A pull request subject is a different string. Trust the ref you deploy from.

Do not create an access key for the pipeline.

- [About security hardening with OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect)
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [Create an OpenID Connect identity provider in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)

## GCP

Use Workload Identity Federation. A workload identity pool trusts the CI issuer, an attribute condition limits which subjects can use it, and the job impersonates a service account. There is no JSON service-account key to download into a CI variable.

- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)

## Azure

Add a federated credential to a user-assigned managed identity or an app registration. The issuer and the subject match the CI token. The job requests a token from Microsoft Entra ID as that identity. Do not store a client secret for the pipeline.

- [Workload identity federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)

## What this page does not decide

OIDC answers how a job proves who it is. It does not answer what that identity should be allowed to reach, how a workload inside the cluster gets its own identity, or what happens when the token's subject is wider than you thought. Those are design decisions, and they belong earlier than the pipeline. The longer argument is in [Security that arrives at the end of the sprint is already late](https://shafie.dev/notes/security-before-production/).
