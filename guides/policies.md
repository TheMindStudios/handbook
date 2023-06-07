# Policies

Policy objects are plain old Ruby classes that encapsulate complex read operations. One definition says that policy objects are similar to service objects, but the difference is that service objects are used for write operations and policy objects for reads. Also, they are different from query objects because query objects focus on SQL reads, while policy objects operate on data already loaded in memory.

Policy objects should not contain any logic that is not directly related to policy.

[Pundit](https://github.com/varvet/pundit) is the most commonly used gem based on policy objects.

On old projects you can see that we are using [CanCanCan](https://github.com/CanCanCommunity/cancancan). It uses authorization rules defined in one place (Ability class) to check if a user is allowed to perform an action. It is recommended to use **Pundit** gem in new projects.
