---
title: "Why jenkinsfile-runner's Docker Image Breaks When You Try to Lint (and What Actually Works)"
date: 2026-09-29
tags: [jenkins, jenkinsfile, ci-cd, docker, pipeline-linting, devops]
description: "jenkinsfile-runner is the official way to validate a Jenkinsfile offline, but its Docker image has three real, still-open bugs around the lint command. Here are the exact error messages, the linked issues, and what actually works around them."
---

# Why jenkinsfile-runner's Docker Image Breaks When You Try to Lint (and What Actually Works)

If you've ever wanted to catch a broken `Jenkinsfile` before you push it and wait on a real Jenkins server, you've probably found [`jenkinsfile-runner`](https://github.com/jenkinsci/jenkinsfile-runner) — the official, Jenkins-core-maintained tool for running (and, since 2021, linting) a declarative pipeline offline. It ships as a Docker image with a Jenkins WAR and a minimal plugin set already baked in, so in theory you never need a real server just to check syntax. In practice, the Docker image's `lint`/`declarative-linter` path has three separate, still-open bugs, and none of them are about your Jenkinsfile being wrong.

## Problem 1: the linter command silently does nothing, or rejects the exact syntax its own help text shows

[Issue #461](https://github.com/jenkinsci/jenkinsfile-runner/issues/461), "unable to use declarative-linter option with jenkins/jenkinsfile-runner image", is a report of two separate failures with the same command. Running `declarative-linter` on its own produces no console output at all — nothing to tell you whether it worked, failed, or is still doing something. Piping a file in the way the tool's own documentation implies, `declarative-linter < /workspace/Jenkinsfile`, fails outright with:

```
ERROR: No argument is allowed: <
```

The reporter's own read of the situation is the frustrating part: the help text for the command literally reads `java -jar jenkins-cli.jar declarative-linter - Validate a Jenkinsfile containing a Declarative Pipeline`, which looks like exactly the invocation that just failed. The issue is still open, with no maintainer response as of this writing — so if you've hit this and concluded you were holding it wrong, you're in good company; nobody has confirmed the right invocation.

## Problem 2: the feature has had zero documentation for four years, by the maintainer's own admission

[Issue #521](https://github.com/jenkinsci/jenkinsfile-runner/issues/521), "Create a new Docs page for Pipeline Verification", was opened on June 10, 2021 — by **oleg-nenashev**, a Jenkins core maintainer, not an outside user. His own description: "it is a new useful use-case for Jenkinsfile Runner users, and it deserves a documentation section." The acceptance criteria are modest — a dedicated linting/verification page, current Declarative Pipeline support status, optionally an example of a failing (negative) scenario. The feature itself had already shipped in [PR #511](https://github.com/jenkinsci/jenkinsfile-runner/pull/511). Four-plus years later, the issue is still open. If you've been guessing at the right flags and output format from trial and error instead of a docs page, that's not a gap in your research — there genuinely isn't one to find yet.

## Problem 3: the `latest` tag can't download its own dependency because of an outdated CA store

This is the one most likely to actually block you, because it doesn't ask you to get an invocation right — it fails immediately, before your Jenkinsfile is even read. [Issue #738](https://github.com/jenkinsci/jenkinsfile-runner/issues/738), "Docker image has outdated CA truststore (SSL handshake fails when downloading Jenkins WAR)", reports this exact stack:

```
javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed
unable to find valid certification path to requested target
```

thrown from `JenkinsLauncherCommand.getJenkinsWar()`. The root cause is a chain of two separate problems, not one: invoking the image with a custom command (like `lint`) drops the image's baked-in `-w`/`-p` flags that normally point it at the Jenkins WAR and plugins *already bundled inside the image* — so instead of using what's right there, it falls back to downloading a fresh WAR over HTTPS at container startup. That download then fails, because the `latest` tag's embedded Java runtime has an outdated CA trust store that can no longer validate the current certificate chain for the download host. Two bugs stacked on top of each other: a flag-dropping behavior, and a stale image nobody has refreshed. The reporter also asked for a `linux/arm64` build while they were at it — also still outstanding.

## What actually works

All three issues share one root behavior: the moment you hand `jenkinsfile-runner`'s Docker image anything other than its default invocation, you lose the flags that keep it self-contained, and you're on your own for figuring out the replacement syntax with no current documentation to check it against. The fix isn't a workaround inside `declarative-linter` itself — none of these issues have a merged fix yet — it's making sure you never hit the fallback path in the first place: explicitly re-supply `-w`/`-p` pointing at the WAR and plugins already inside the image whenever you override the default command, so the container never tries to reach the network at all.

That's the entire fix underneath [`jflint`](https://github.com/johndow-42/jflint) (`npm install -g jenkinsfile-lint`, installs the `jflint` command) — a thin wrapper I built specifically because I kept hitting issue #738 firsthand. It always calls the image with `-w`/`-p` set, so linting stays fully offline and never triggers the WAR download that fails on the outdated CA store, and you get one readable pass/fail result instead of `ERROR: No argument is allowed: <` or silence. It doesn't reimplement Jenkins' own validation logic — the engine underneath is the same Jenkins core your real server runs — and it doesn't fix issues #461 or #521 upstream, because those live in `jenkinsfile-runner` itself, not in anything a wrapper can patch. What it does is give you a command that works the same way every time, without needing to already know about three separate open GitHub issues first.

## What to actually do about it

1. **If you're calling the Docker image directly, don't drop `-w`/`-p`.** Re-supply them explicitly any time you pass a custom command like `lint` or `declarative-linter`, pointing at the WAR/plugins already baked into the image — that's what avoids the issue #738 download entirely.
2. **Don't expect console output from a bare `declarative-linter` call, and don't pipe a file in with `<`.** Issue #461 is still open; there's no confirmed working invocation from the maintainers yet.
3. **Don't go looking for an official linting docs page.** Issue #521 confirms there isn't one, direct from the maintainer who filed it.
4. **If you just want a working `jflint`-style command today**, `npm install -g jenkinsfile-lint` wraps all of the above so you don't have to reproduce it by hand.

Every error string and issue link above is quoted from the real, currently open GitHub issue — none of them have a merged fix as of this writing, so if you search your own error text and land here, you're not missing a setting, you've found the same three gaps everyone else runs into.
