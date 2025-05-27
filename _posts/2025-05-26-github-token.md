---
layout: post
title: GITHUB_TOKEN isn't that Ephemeral
author: nopcorn
description: Compromising CI/CD pipelines can be very fruitful
summary: Compromising CI/CD pipelines can be very fruitful
tags: [cicd, github, github_token, workflow, redteam, actions]
---

> - GITHUB_TOKEN is an ephemeral (cough) API key used for workflows to authenticate to the Github API
> - If you leak the token and make it available prior to the end of the workflow run, bad things can happen
> - Github states that the token is destroyed at the end of a workflow run, but it isn't immediately invalidated
> - You can race Github's invalidation process for fun and profit

*This post is a collaboration between myself and [jackhac](http://jackhacsecurity.com/)*

Every time a [GitHub Actions](https://docs.github.com/en/actions) workflow is triggered, GitHub automatically provisions a token called [`GITHUB_TOKEN`](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication). This ephemeral credential allows jobs in the workflow to authenticate to the GitHub API without requiring manually managed secrets or personal access tokens. The token is injected into the workflow environment and is scoped to the repository context in which the workflow is running.

GITHUB_TOKENs can have [multiple different default permission sets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#permissions-for-the-github_token) though these permissions can be customized within the workflow or job itself. Job-level scoping is prefered as it gives fine-grained control over access. 

# Dude where's my token

Despite the ephemeral nature of GITHUB_TOKEN, misuse or mishandling during the workflow execution window can expose projects to significant risk. Many teams assume that temporary credentials are inherently safe, but this assumption breaks down in environments where artifacts, logs, or third-party actions introduce uncontrolled surfaces.

One of the most overlooked issues with GITHUB_TOKENs comes from a subtle behavior in [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact). In this version, artifacts [become downloadable as soon as they're uploaded]((https://github.com/actions/upload-artifact?tab=readme-ov-file#improvements)), even if the workflow that created them is still running. This creates a narrow but potent attack window: if the artifact contains a copy of the GITHUB_TOKEN, an attacker can retrieve it and execute arbitrary API calls in the context of the job until the workflow terminates.

That said, most workflows that upload artifacts tend to do it at the very end. So, in theory, there shouldn’t be much of a window for an attacker to abuse the GITHUB_TOKEN, right? The workflow usually wraps up right after the upload, which should invalidate the token almost immediately. Right? Right!?

# Use after free (ish)

While the official documentation states that the GITHUB_TOKEN is revoked immediately after the workflow ends, we discovered that this isn't strictly true... *surprised pickachu*

During a series of tests conducted in collaboration with my research partner [jackhac](http://jackhacsecurity.com/), we observed that the token often remains valid for a short period (typically between one and two seconds) after the workflow reports as completed. In other words, the revocation process is not instantaneous. This creates a brief but real post-execution window during which a previously retrieved token can still be used to interact with the GitHub API.

We confirmed the racing by setting up a dummy vulnerable workflow in `nopcorn/artifact-exploit-poc` that would purposefully write the GITHUB_TOKEN to an artifact using `actions/upload-artifact@v4`. We then used a script to manually kick off the workflow, monitor for the artifact, download it, extract the value of GITHUB_TOKEN, and use it to authenticate to the Github API out of band for as long as it was valid. Afterwards we compared timestamps:

```
$ python exploit.py --repo nopcorn/artifact-exploit-poc --polling-interval 0.5
[*] Manually triggering the workflow to start...
[*] Waiting for the workflow run to be detected...
[+] Active workflow run found! Run ID: 15263730946, Status: queued
[*] Waiting for run 15263730946 for artifact 'secret-file'...
[*] Downloading artifact secret-file...
[+] Successfully extracted GITHUB_TOKEN from the artifact -> ghs_PsmNtQuBpes3u9x8MxQx22gE6C7Oc42QTsLj
[+] Monitoring GITHUB_TOKEN validity...
[+] 2025-05-27T00:01:42.289469Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:42.980345Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:43.668557Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:44.360745Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:45.023419Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:45.688257Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:46.353344Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:47.046626Z: Valid GITHUB_TOKEN! (status code 200)
[+] 2025-05-27T00:01:47.754317Z: Valid GITHUB_TOKEN! (status code 200)
[!] 2025-05-27T00:01:48.423765Z: Invalid GITHUB_TOKEN: Bad credentials
```

If we take a look at the [raw run logs for the workflow](https://github.com/nopcorn/artifact-exploit-poc/commit/79cadb723caa718616c04661e32ee628bf53c1b5/checks/42925720356/logs), you can see the workflow ends around `2025-05-27T00:01:46`, but we're able to successfully use the GITHUB_TOKEN until at least `2025-05-27T00:01:47` - a whole second later. While this doesn't seem like a lot there are several techniques a threat actor could use to compromise a repository with only a single API call.

While this is a contrived example, a scan of all Github repositories using [SourceGraph](sourcegraph.com) proves this is more common than you might think.

# Disclosure

We responsibly disclosed this finding to GitHub through their vulnerability reporting program on HackerOne. The issue was triaged and investigated over the course of a month. Ultimately, however, GitHub classified the behavior as "informational" and chose not to pursue it further. Their position was that since the token is eventually revoked, the brief extension does not constitute a vulnerability under their current threat model.

![](/assets/img/github_this_is_fine.jpg)

While we respect their decision, it underscores a philosophical gap between how ephemeral secrets are documented and how they behave in practice. For red teamers, this matters. If you can automate artifact retrieval and parse secrets before the revocation race completes, you have a viable short-term credential to pivot or persist.

We approached many of the vulnerable projects we identified during our scans (via bug bounty programs and Github Security disclosures) and we're happy to say that the vast majority of them were receptive to the exploit vector and promptly fixed their vulnerable repositories. 