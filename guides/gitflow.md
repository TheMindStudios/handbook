# Gitflow

## Main assumptions

1. At first all feature branches are created from **development** (can be dev or develop). Branches that contains bug fixes can be created from **staging** or **main** branches.

2. Name of branch should contain task from Jira. For example, **task/EA-967_website_updates**. Possible prefixes for branch names: task, story, bug.

3. You commit messages should follow [Conventional commit guidelines](https://www.conventionalcommits.org/en/v1.0.0/).

4. Only Team Lead or lead project developer can have access to deploy branches (dev, staging or master). Other team members shouldn't try to push directly, only through merge request process.

## Branches

Branch                                   | May branch off from       | Must merge back into          | Description
---------------------------------------- | ------------------------- | ----------------------------- | -----------
`main`                                   | —                         | `staging`, `development`      | The latest stable version of the code base.
`development`                            | `main`                    | `staging`                     | Default branch to create new branches from and target branch for Merge requests.
`task/subject`, `task/xxx-subject`       | `staging`, `development`  | `development`, `staging`      | Use for developing new features. Merged into `development` or `staging` depending on your release process. When all features developed in the sprint go to release, then you merge `task` branch into `development`. When each task should be released independently, you merge  `task` branch into `staging` after testing on `development`. `xxx` - issue ID in task tracker like JIRA, `subject` - issue subject.
`bug/subject`, `bug/xxx-subject`         | `main`                    | `main`, `development`         | Maintenance or "hotfix" branches are used to quickly patch production releases. Where `xxx` - issue ID in task tracker like JIRA, `subject` - issue subject.
`staging`                                | `development`             | —                             | Represents the current staging server state. development, task/\*, bug/\* branches may be merged here directly for testing and demonstration.

## Steps to create merge request

1. The developer will pull latest changes from **development** branch

2. The developer will create a branch with Jira issue id followed by issue type and title eg. **task/UNIGHT-760_update_login_screen**. Available types: **bug, task, story**.

3. The developer will fix the issue.

4. The developer will create unit and integration tests as applicable.

5. The developer will check that all new requests were added to Postman collection or Swagger for current project.

6. The developer will go through the checklist before creating merge request.

7. Before commit the developer will check changes made using `git diff`, or IDE tools. Unnecessary changes should be removed from code (local changes make in secrets/credentials for testing, unnecessary logging etc...)

8. Before commit the developer will check the status of changed files using git status or IDE tools, to ensure that all necessary files are added to commit.

9. The developer will switch to the **development** branch, pull latest changes from it and will merge it with his local branch fixing merge conflicts if needed.

10. The developer will ensure that the server runs locally without errors after the merge process.

11. If the unit and integration tests run successfully will create a merge request.

12. The developer will then copy the link of merge request and paste it into the **#backend** channel and tag **@channel** or to Team Lead direct messages.

13. The developer other than the one that owns the merge request will review it.

14. The developer that reviews will go through the checklist again and then review the merge request.

15. If there are any suggestions or any of the points of the checklist that are applicable but missed, the reviewer will comment on the merge request.

16. The owner of the merge request then performs the update again and resolves conflicts if any and again puts a fresh link in the same thread as the original one.

17. If the code review passes the reviewer approves and then merges the code and deletes the branch by keep the checkbox **Delete source branch** checked when merging.
