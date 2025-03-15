# GitLab


## Provisioning Tasks

### AddGitlabExternalIdentity

This task makes it easier to add an external identity without writing code. Useful in JIT workflows.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.gitlab.provisioning.tasks.AddGitlabExternalIdentity
  params:
    # Which omni_auth provider to use
    provider: "openid_connect"
    # The attribute in the user object to map the user's identity to
    userAttribute: "username"
```

### AddGroupToProject

This task will add an existing group to a project and set its entitlements on that project. The group will be added to the last project created by the CreateProject task.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.gitlab.provisioning.tasks.AddGroupToProject
  params:
    # Group to add
    groupName: "approvers-k8s-$nameSpace$"
    # GitLab provisioning target
    targetName: "gitlab"
    # Entitlement level for the group in the project
    accessLevel: "MAINTAINER"
```

### CreateDeploymentKey

CreateDeploymentKey will create a deployment key on a project, making the key and its base64-encoded value available in the workflow's request object.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.gitlab.provisioning.tasks.CreateDeploymentKey
  params:
    # GitLab provisioning target
    targetName: "gitlab"
    # Project namespace
    namespace: "$nameSpace$-production"
    # Project name
    project: "$nameSpace$-application"
    # Label for the key
    keyLabel: "tekton_pull"
    # If the key is writeable or read-only
    makeWriteable: "false"
    # The name of the request object where the key is stored, base64 encoded
    privateKeyReuestName: "tektonPullecret"
    # The name of the request object where the key is stored, plain text
    privateKeyReuestNamePT: "tektonPullSecretPT"
```

### CreateGitFile

Creates a file in the named project.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.gitlab.provisioning.tasks.CreateGitFile
  params:
    # GitLab target
    targetName: "gitlab"
    # Project namespace
    namespace: "$nameSpace$-production"
    # Project name
    project: "$nameSpace$-application"
    # Branch to commit against
    branch: "master"
    # Path and file (excluding "/")
    path: "README.md"
    # Content of the file to create
    content: |
      # $nameSpace$-application
      
      Fork this project to create your application. Create a pull request to trigger a build and deployment to development.
    # Commit message
    commitMessage: "initializing the repository"
```

### CreateProject

Creates a GitLab project. Can optionally create a webhook and generate a deployment key.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.gitlab.provisioning.tasks.CreateProject
  params:
    # Project namespace
    namespace: "$nameSpace$-production"
    # Project name
    name: "$nameSpace$-application"
    # Project description
    description: "Application project"
    # Enable issues
    issuesEnabled: "true"
    # Enable merge requests
    mergeRequestsEnabled: "true"
    # Enable wiki
    wikiEnabled: "true"
    # Enable snippets
    snipitsEnabled: "true"
    # Visibility level
    visibility: "2"
    # GitLab provisioning target
    targetName: "gitlab"
    # Git SSH Host
    gitSshHost: "#[GITLAB_SSH_HOST]"
    # Optionally create a webhook
    createWebhook: "true"
    # Webhook suffix
    webhookSuffix: "#[GITLAB_WEBHOOK_SUFFIX]"
    # Webhook branch filter
    webhookBranchFilter: "master"
    # Webhook secret request name
    webhookSecretRequestName: "appProjectWebhook"
```

### ForkProject

Forks a GitLab project into another namespace.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.gitlab.provisioning.tasks.ForkProject
  params:
    # Source project name
    sourceProjectName: "$nameSpace$-operations"
    # Source project namespace
    sourceProjectNamespace: "$nameSpace$-production"
    # Destination namespace
    destinationNamespace: "$nameSpace$-dev"
    # GitLab provisioning target
    targetName: "gitlab"
    # Git SSH Host
    gitSshHost: "#[GITLAB_SSH_HOST]"
```