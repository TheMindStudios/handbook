# Gitflow

This document outlines our branching and merging strategy, which is primarily based on GitLab Flow with a few key adaptations to fit our development workflow. We prioritize a stable `main` branch representing production, a `staging` branch representing the next release, and a `development` branch for ongoing feature integration and testing.

## Main Assumptions

1.  **Branch Creation:**
    *   **Feature Branches:**  New feature branches are created from a stable branch, typically `staging`. In some project-specific cases, they might be created from `main`.  The goal is to branch off from the code that most closely resembles the environment where the feature will eventually be deployed.
    *   **Bug Fix Branches:** Bug fix branches are created from the branch where the bug is identified. This prioritizes fixing issues in the relevant environment:
        *   **Production Bugs:** Branch from `main`.
        *   **Staging Bugs:** Branch from `staging`.
        *   **Development (Dev) Bugs:** Ideally, fix the bug directly in the relevant feature branch.  If the bug is not specific to a feature, create a bug fix branch from *the feature branch* where the issue was discovered. This avoids contaminating `staging` or `main` with untested fixes.

2.  **Branch Naming:** Branch names should include the Jira task ID for traceability.  Example: `task/EA-967_website_updates`.  Allowed prefixes: `task`, `story`, `bug`.

3.  **Commit Messages:**  All commit messages *must* adhere to the [Conventional Commits specification](https://www.conventionalcommits.org/en/v1.0.0/). This ensures consistent and informative commit history, which is crucial for automated release notes.

4.  **Restricted Access:** Only Team Leads or designated lead project developers have direct push access to protected branches (`main`, `staging`). All other team members must use merge requests.

## Branches

| Branch                             | May branch off from            | Must merge back into        | Description                                                                                                                                                                                                      |
|------------------------------------|--------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `main`                             | —                              | —                           | Represents the live, production code. This branch is always stable and deployable.  Merges come only from `staging` (after a release) and occasionally from `bug/*` branches (hotfixes).                  |
| `staging`                          | `main`                         | `main`                      | Represents the next release candidate. All features planned for the upcoming release accumulate here.  This branch is merged into `main` after release confirmation, triggering the automated release process. |
| `development`                      | `main` at the start of development               | -             | Default branch for features testing and demonstration after merge request from features branches.                                                                                                                |
| `task/subject`, `task/xxx-subject` | `staging`, (occasionally `main`) | `development`, `staging`                        | Used for developing new features.  `xxx` is the Jira issue ID, and `subject` briefly describes the feature.  These branches are merged into `development` for integration testing.                        |
| `bug/subject`, `bug/xxx-subject`    | `main`, `staging`, feature branches | `main`,`staging`, `development`, feature branches         | Used for fixing bugs. The branching point depends on where the bug was found (see assumption #1). `xxx` = Jira ID, `subject` = short description. The merge target mirrors the branching point.      |

## Merge Request Steps

1.  **Pull Latest:**  Before starting work, pull the latest changes from the target stable branch (`staging` in most cases, or `main` if required by the project).

2.  **Create Branch:** Create a new branch with the appropriate prefix and Jira issue ID:  `task/UNIGHT-760_update_login_screen`.  Valid prefixes: `bug`, `task`, `story`.

3.  **Implement & Test:** Develop the feature/fix, and create corresponding unit and integration tests.

4.  **API Documentation:** Ensure all new requests are documented in the project's Postman collection or Swagger documentation.

5.  **Pre-Commit Checklist:**
    *   Thoroughly review your changes using `git diff` or your IDE's diff tools. Remove any unintentional changes (local testing configurations, unnecessary logs, etc.).
    *   Use `git status` or your IDE to confirm that all necessary files are staged for commit.

6.  **Merge and Resolve Conflicts:** Switch to the `development` branch, pull the latest changes from it, and merge it with your local branch, fixing merge conflicts if needed.

7.  **Local Testing:**  After merging with `development`, ensure the application runs locally without errors.  Run all relevant tests.

8.  **Create Merge Request:**  If tests pass, create a merge request targeting the `development` branch.

9.  **Notify Team:** Copy the merge request link and post it in the `#backend` channel, tagging `@channel` or directly notifying the Team Lead.

10. **Code Review:** A developer *other than the author* must review the merge request.

11. **Reviewer Checklist:** The reviewer should rigorously examine the code, address any checklist items, and provide feedback in the merge request.

12. **Address Feedback:** The merge request owner addresses any comments, resolves conflicts, and updates the merge request.  Post an updated link in the *same thread* as the original.

13. **Merge:** If the code review passes, the reviewer approves and merges the code.  The reviewer *must* check the "**Delete source branch**" option during the merge.

## Release Process (Automated via GitLab CI)

The release process is automated via GitLab CI/CD jobs triggered by merges into the `main` branch.  Here's a breakdown of the process:

1.  **Merge to Main:** The release process begins when `staging` is merged into `main`. No manual release creation is needed.

2.  **Automated CI Jobs:** To enable automated releases, include the following in your project's `.gitlab-ci.yml` file:

    ```yaml
    include:
      - project: 'web/ci-templates'
        ref: main
        file: '/templates/release.yml'
    ```
    This includes a predefined CI/CD template (`release.yml`) from a central repository (`web/ci-templates`). This template contains the necessary jobs:
    *   **`prepare_release_job`:**
        *   Reads the `VERSION` file to determine the new release version (and tag).
        *   Determines the previous tag (`PREV_TAG`) using Git commands and, if necessary, the GitLab Releases API. If no previous tag exists, it uses the first commit as a fallback.
        *   Generates a `release.description` file containing the changelog, using a custom Rake task (`changelog:generate`) that processes conventional commits between `PREV_TAG` and the current commit. This changelog is structured into sections based on commit types.
        *   Creates a `variables.env` file to pass `TAG` and `PREV_TAG` to the next job.

    *   **`release_job`:**
        *   Uses the `release-cli` to create a GitLab release.
        *   Sets the release name to "Release $TAG".
        *   Uses the generated `release.description` for the release notes.
        *   Creates a tag named `v$TAG` (automatically prepending "v" to the version from the `VERSION` file).
        *    References the commit SHA (`$CI_COMMIT_SHA`) of the merge commit.

**Key Differences from Standard Gitflow:**

*   **Development Branch:** Instead of being long-lived and merging back into `staging`/`main`, our `development` branch serves as a integration point for feature testing.
*   **Branching Point:** Feature branches typically originate from `staging` (the next release) rather than `development`.
*   **Release Process:** We leverage GitLab's *automated* release feature, creating releases directly from CI/CD jobs triggered by merges to `main` and using the `VERSION` file and automatically generated change log and git tag.
*   Feature branch are merged directly to dev and QA make testing based on development or dev branch.
*   Development or dev branch must be never merged into staging.
