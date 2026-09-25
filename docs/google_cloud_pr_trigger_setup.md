# Google Cloud UI: create a PR trigger for dev-to-main firmware builds

This guide explains how to set up a Cloud Build trigger in the Google Cloud Console so it runs when a pull request is raised to merge a dev branch into `main`.

This is the standard CI gate for the workflow described in the project guidance: a PR build compiles the firmware and produces an artifact, then a person tests that exact artifact on hardware before merging.

## Goal

Trigger a Cloud Build job whenever a pull request targets `main`, so the build validates the candidate firmware before it is approved for hardware testing.

---

## Before you start

Make sure all of the following are in place:

- A Google Cloud project exists
- Billing is enabled
- The Cloud Build API is enabled
- The Artifact Registry API is enabled
- Your GitHub repo is connected to Cloud Build
- The repo contains a build config such as `.cloudbuild/dev/dev.cloudbuild.yaml`
- The repo has a default branch or target branch named `main`

---

## Step 1: open Cloud Build

1. Go to the Google Cloud Console: https://console.cloud.google.com/
2. Select your project.
3. In the left navigation, open Cloud Build.
4. Click Triggers.
5. Click Create Trigger.

---

## Step 2: choose the repository source

1. In the trigger form, choose the repository source.
2. If this is the first time, click Connect Repository.
3. Choose GitHub and authorize Google Cloud to access the repo.
4. Select the repository you want to build.
5. Save the connection.

Once connected, the repo will appear in the trigger list.

---

## Step 3: set the trigger type

In the Create Trigger screen:

1. Give the trigger a name, for example:
   - `ugv-pr-main-build`
2. Choose the event type:
   - Pull request
3. Choose the repository you connected.

This means the trigger runs when a pull request is opened or updated, rather than on every branch push.

---

## Step 4: require manual approval before the PR build runs

When you select **Pull request**, Google Cloud may warn that anyone with read access to the repository can open a pull request and cause Cloud Build to execute code from that pull request.

For this project, enable the approval gate for the PR trigger so the build is created but does not run until a trusted maintainer approves it.

In the trigger configuration:

1. Find the approval option for the trigger.
   - In some Cloud Build UI versions this appears as **Require approval before build executes**.
   - In other versions it may appear under an **Approval** or **Advanced** section.
2. Enable the approval requirement.
3. Confirm the trigger uses a Cloud Build service account with only the permissions needed to compile firmware and write the build artifact.

With this enabled, opening or updating a PR will create a pending build. A trusted maintainer must approve that build before Cloud Build executes the PR code.

This is especially important for public repositories or repositories where contributors outside the trusted maintainer group can open pull requests.

---

## Step 5: restrict it to PRs targeting main

This is the key part.

Set the trigger so it only runs when the PR is targeting `main`.

If the UI gives you branch filters, use:

- Source branch: `dev` or `dev/*` if you want only dev-style branches
- Target branch: `main`

If the UI only asks for a branch regex, use:

- `^main$`

This means: only run the build when a PR is merging into `main`.

If you want to support only dev-to-main PRs, keep the branch pattern narrow and avoid triggering on unrelated branches.

---

## Step 6: choose the build configuration file

In the trigger configuration:

1. Set Build configuration to:
   - Cloud Build configuration file (yaml or json)
2. Set the file location to the repo path of your Cloud Build config, for example:
   - `.cloudbuild/dev/dev.cloudbuild.yaml`

This tells Cloud Build where the instructions are.

If the file is inside a subfolder, use the exact relative path from the repo root.

---

## Step 7: verify the build config path and variables

Before saving, confirm that the config file exists and the referenced variables are valid.

Your build config likely expects a file like:

- `.cloudbuild/dev/version.env`

Make sure the file path and environment values are correct.

If you use a separate environment file, the Cloud Build config should load it before build steps run.

---

## Step 8: save the trigger

1. Review the trigger settings.
2. Click Create.
3. The trigger will appear in the Cloud Build Triggers list.

At this point, it is ready to run when a PR targets `main`.

---

## Step 9: test it with a pull request

To confirm it works:

1. Create a branch from `dev` or a feature branch.
2. Commit a small change.
3. Open a pull request to merge into `main`.
4. Watch the Cloud Build page.
5. Confirm the trigger creates a build that is waiting for approval.
6. Have a trusted maintainer review the PR source changes and approve the build in Cloud Build.
7. Confirm the build starts only after approval.
8. Check the build logs for errors.

If the build fails, fix the config or dependency versions before merging.

---

## Step 10: require it in GitHub branch protection

This is strongly recommended.

In GitHub:

1. Open the repo settings.
2. Go to Branches.
3. Add or update a branch protection rule for `main`.
4. Require the status check from Cloud Build.
5. Save the protection rule.

This ensures a PR cannot be merged to `main` unless the Cloud Build check passes.

---

## Recommended trigger behavior for this project

For this repo, the best setup is:

- Trigger type: Pull request
- Target branch: `main`
- Approval: required before build execution
- Build config: `.cloudbuild/dev/dev.cloudbuild.yaml`
- Required check: Cloud Build status required for merge

That follows the recommended workflow in the CI design:

- PR builds the candidate firmware
- a trusted maintainer approves the build before PR code executes
- the exact binary is tested on hardware
- only then is the PR merged and the artifact promoted

---

## Notes and gotchas

- Do not trigger this on every push to every branch unless you want build spam.
- Keep the trigger focused on PRs that target `main`.
- Require approval before PR builds execute, especially for public repos or external contributors.
- If you are using external contributors, avoid exposing secrets in PR builds.
- Make sure the build uses pinned versions for the Arduino toolchain and libraries.
- The build should be deterministic so the same repo state gives the same artifact.

---

## Summary

The setup is:

1. Connect GitHub to Cloud Build
2. Create a Pull request trigger
3. Require approval before PR builds execute
4. Restrict it to PRs targeting `main`
5. Point it to `.cloudbuild/dev/dev.cloudbuild.yaml`
6. Require the status check before merge

That gives you a clean PR-based compile gate without adding unnecessary build noise.
