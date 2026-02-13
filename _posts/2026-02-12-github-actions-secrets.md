---
layout: post
title: GitHub Actions Secrets - Your New Favorite Red Team Primitive
author: nopcorn
description: GitHub Actions secrets are designed to protect your credentials, but the same properties that make them secure make them a versatile tool for attackers.
summary: GitHub Actions secrets are designed to protect your credentials, but the same properties that make them secure make them a versatile tool for attackers.
tags: [cicd, github, github_token, workflow, redteam, actions, github_actions, secrets]
---

> - Anyone with write access to a repo can overwrite any secret, including environment secrets protected by required reviewers
> - Passing secrets through environment variables only protects against shell injection, not YAML injection, jq injection, CRLF, or other downstream parsing issues
> - Use GitHub Actions secrets as a swiss army knife when attacking CI/CD pipelines. It makes detection and response much more challenging.

*This post is a collaboration between myself and [jackhac](http://jackhacsecurity.com/)*

GitHub Actions secrets are encrypted variables designed to store sensitive information like API keys, tokens, and credentials. Their core premise is straightforward, once you set a secret its value is hidden from all GitHub interfaces. You cannot read it back through the UI, the CLI, or the API. Repository administrators can't retrieve a secret's plaintext value after it has been stored either. GitHub [encrypts secrets using Libsodium sealed boxes](https://docs.github.com/en/actions/concepts/security/secrets) before they even reach GitHub's servers, and the values are only decrypted at runtime when injected into a workflow.

This sounds great. And to be fair, the encryption and access control mechanisms are solid. The problem isn't with how secrets are stored, but rather with how they are *used*, who can *modify* them, and what the documentation *doesn't* adequately warn defenders about.

The GitHub documentation on secrets has a few significant blind spots that, taken together, create real opportunities for attackers during red team engagements. Their design as un-auditable values also provide avenues for dodging defenders. This post will walk through some blind spots regarding secrets and how you can leverage them when attacking GitHub Actions.

# Secrets for Shell Injection

GitHub's [security hardening guide](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) states:

>  Any user with write access to your repository has read access to all secrets configured in your repository.

The docs are a bit misleading. Here "read access" means users with write access can list secrets names. But the statement undersells the actual risk. Users with write access can also *overwrite* secret values. The [REST API for Actions secrets](https://docs.github.com/en/rest/actions/secrets) requires only collaborator access to create or update secrets, and the [Using Secrets documentation](https://docs.github.com/actions/security-guides/using-secrets-in-github-actions) confirms that you need write access to create secrets for an organization repository. This has been [confirmed as intentional behavior](https://github.com/orgs/community/discussions/159727) by GitHub, and there is a [long-standing docs issue](https://github.com/github/docs/issues/1087) about the confusing way this is documented.

In most organizations, write access is extremely common. Nearly every developer on a team has push permissions to at least some repositories. There is no separate permission for "can modify secrets" versus "can push code." If you can push a branch, you can overwrite a secret.

The surprising part is that **this applies to protected environment secrets too.** You might assume that environments with [required reviewers](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments) are protected from this kind of tampering. They are not. Required reviewers gate workflow *execution* only. The API endpoint for creating or updating environment secrets has the same permission requirement as repository secrets. No additional approval step, no reviewer notification. An attacker with write access can overwrite a production environment secret via `gh secret set SECRET_NAME --env production`, and the next time a legitimate deployment is approved by a reviewer, the poisoned value gets used.

If a workflow interpolates a secret directly into a `run` step, an attacker with write access can overwrite that secret with a value containing shell metacharacters and get arbitrary code running on the workflow runner.

The obvious problem is noise. Blindly overwriting a secret you don't know the value of will almost certainly break the pipeline, trigger an investigation, and announce your presence. That might be acceptable for a smash-and-grab, but there will be consequences.

Consider this vulnerable workflow step:

{% raw %}
```yaml
- name: Deploy to production
  run: |
    echo "${{ secrets.SSH_PRIVATE_KEY }}" > /tmp/deploy_key
    chmod 600 /tmp/deploy_key
    ssh -i /tmp/deploy_key -o StrictHostKeyChecking=no deploy@prod.example.com "deploy_script.sh"
```
{% endraw %}

The direct interpolation of `SSH_PRIVATE_KEY` makes this vulnerable to shell injection. But to exploit it cleanly, we first need to know the secret's current value so we can preserve it in our payload. That means we need to figure out where the secret lives: is it an organization secret, a repository secret, or an environment secret? The answer determines whether we can read the original value and whether we can override it at a different scope.

```bash
gh secret list --org <org>
gh secret list --repo <org>/<repo>
gh secret list --repo <org>/<repo> --env <env>
```

Assuming it is an organization secret, we can dump its value by pushing a throwaway workflow to a feature branch. Organization secrets are passed to all workflows in repositories that have access to them, regardless of whether any job explicitly references the secret by name.

{% raw %}
```yaml
# .github/workflows/malicious.yaml (pushed to a feature branch)
name: debug
on: push
jobs:
  dump:
    runs-on: ubuntu-latest
    steps:
      - name: nothing to see here
        run: echo '${{ toJSON(secrets) }}' | base64 | base64 # bypass log masking
```
{% endraw %}

This writes all available secrets to the workflow logs and bypasses GitHub's secret masking. In a real engagement you would blend the workflow name and step names into something ordinary, and you would probably exfiltrate the values to a server you control rather than leaving them in the logs.

Now that you know the real value, you can exploit the secret context hierarchy to inject a poisoned value without disrupting other repositories. GitHub resolves secret name collisions using a [fixed priority order](https://docs.github.com/en/actions/reference/security/secrets#naming-your-secrets): environment secrets take precedence over repository secrets, which take precedence over organization secrets. If the same secret name exists at multiple levels, the most specific scope wins.

This means you can shadow a broadly-scoped secret by creating one at a narrower scope. Set a repository-level secret with the same name as the organization secret, and only that repo picks up the poisoned value. Every other repo continues using the original. Delete the repository secret when you're done and the organization secret takes effect again, with no trace at the organization level.

The same principle applies one level deeper. If a workflow uses environment secrets, you can shadow a repository secret by setting an environment secret with the same name. The workflow job that references that environment will use your value; jobs that don't reference the environment will still see the original.

In this case, we set a repo secret to override the org secret and successfully take over the pipeline:

```bash
gh secret set SSH_PRIVATE_KEY --repo <org>/<repo> \
  --body "<real_SSH_PRIVATE_KEY_value>\" > /tmp/deploy_key; <injection payload>; #"
```

The pipeline still works. The SSH key is still valid. And your payload runs alongside it. When you are done, delete the repository secret and the original organization secret takes effect again.

# Injection Beyond the Shell

Sure, just don't write vulnerable workflows and you're fine. Right?

GitHub has an entire [dedicated page on script injections](https://docs.github.com/en/actions/concepts/security/script-injections) in the context of Actions workflows. It does a decent job of explaining how untrusted input from contexts like `github.event.pull_request.title` can lead to command injection when interpolated into `run` steps. The recommended mitigation is to pass values through environment variables rather than directly interpolating expressions. This is good advice. The shell treats the variable as a string, not as executable code.

But shell injection is not the only injection technique. Secrets get passed to all kinds of programs in a typical CI/CD pipeline, and many of those programs have their own interpretation of special characters. An environment variable stops the shell from expanding metacharacters. It does nothing to stop the *receiving program* from interpreting the secret's contents in dangerous ways.

This shifts the attacker's goal. Instead of exploiting a workflow that already exists, the aim is to get a workflow change merged that looks benign but is exploitable through a poisoned secret.

Consider a PR that adds a step like this:

{% raw %}
```yaml
- name: Add API key to local config
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: |
    sed "s/PLACEHOLDER_API_KEY/$API_KEY/" settings.conf
```
{% endraw %}

This looks fine. The secret is loaded as an environment variable, so shell injection is off the table. It uses `sed` to drop a sensitive API key into a settings file. A reviewer who has read GitHub's guidance on secrets will see the `env` block, nod approvingly, and hit approve.

The problem is that `sed` has its own injection surface. The `e` flag causes `sed` to execute the contents of the pattern space as a shell command. A poisoned secret like:

```bash
gh secret set API_KEY --body "<actual_api_key>/;e (cmd1 && cmd2) > /dev/null;#"
```

will cause the `sed` command to execute `cmd1` and `cmd2` on the runner without breaking the pipeline, assuming you were able to leak the original secret value as described earlier. The API key still ends up in the config file. The build still passes.

A reviewer would need to be intimately familiar with the parsing behavior of every program that touches a secret to catch something like this. Realistically, nobody outside of offensive security is thinking about `sed` command execution flags during code review. The combination of opaque secret values and non-obvious injection surfaces makes this a reliable way to slip malicious workflow changes past a reviewer who is doing everything GitHub tells them to do.

# A Note on Disclosure

All of the techniques in this post rely on documented, intentional behavior. Write access allowing secret modification is confirmed as by-design. Direct interpolation of secrets into shell commands is possible by design (the docs recommend against it, but do not prevent it). Secret values being opaque and unauditable is the entire point of the feature. There is nothing here to disclose to GitHub, because none of it is a bug. It is the logical consequence of design decisions that are individually reasonable but collectively create a trust model that attackers can exploit.

# Takeaways

GitHub Actions secrets do exactly what they promise: they store values securely and prevent them from being read back through GitHub's interfaces. The encryption is real, the access controls work as documented, and the log redaction is a thoughtful defense-in-depth measure.

The problem is that these same properties create assumptions. Developers assume that because a value comes from an encrypted store, it is safe to interpolate and requires higher permissions to modify. Reviewers assume that because they have followed GitHub's guidance, non-shell-based injection isn't something to worry about. And everyone assumes that environment protection rules protect the environment's secrets from tampering, when they only protect against unauthorized *consumption*.

None of these assumptions hold up under adversarial pressure. For red teamers, secrets are a versatile tool that provides injection vectors, social engineering aids for bypassing code review, and an invisible storage mechanism for payloads. 