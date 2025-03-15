# Provisioning Tasks

## FilterGroups

This task can be used to limit the groups that are available to a target. For instance if a user could have the groups "Admin","Developer" and "User" but the target only has the groups "Admin" and "User" this task can be used to filter out "Developer". This way no "rogue" groups are presented to a target. This task should be used inside of a mapping task to make sure that other tasks are not effected.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.FilterGroups
  params:
    # A group name that should pass through this filter, case sensitive and can be listed multiple times
    name:
    - "User"
    - "Admins"
```

## LoadAttributes

This task will load attributes from a user's entry in the virtual directory. It's useful when a workflow is only being called with a user identifier or a subset of attributes and additional attributes are needed for reporting or decision making.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.LoadAttributes
  params:
    # An attribute name to load, case sensitive and can be listed multiple times
    name:
      - "givenName"
      - "sn"
    # The name of the attribute that identifies the user in the virtual directory
    nameAttr: "uid"
    # Optional - The directory base in Unison's ldap virtual directory to begin the search at.
    # base: "ou=db,o=Tremolo"
```

## MapGroups

The Map User Groups task will map group names from a "global" name to a target-specific name. For instance, if there is a generic group called "Administrator" but the target stores administrators in the group "SYS_ADMINS," this task can be used to create that mapping. It should be deployed inside of a mapping to make sure that global groups are not affected.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.MapGroups
  params:
    # A mapping of target from source. To map Admins → SYS_ADMIN the value should be SYS_ADMIN=Admins.
    # This attribute can be mapped multiple times.
    map:
      - "SYS_ADMIN=Admins"
```

## SetPassword

This task is useful in user registration scenarios where a user's password must be set but the email address needs to be verified. It triggers a password reset through the password reset authentication mechanism. In order for this task to work, it **MUST** have a password reset authentication mechanism configured where the workflow is configured.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.SetPassword
  params:
    # The name of the password reset mechanism as defined in the Auth Mechs section.
    mechName: "PasswordReset"
```

## Attribute2Group

This task takes the values of an attribute and adds them to a user's groups. This is useful when building generic workflows.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.Attribute2Groups
  params:
    # The name of the attribute to get the group values from. Once the values are added, the attribute is removed from the user.
    attributeName: "roles"
```

## JITIgnoreGroups

This task will allow for a group to be ignored during a just-in-time provisioning process. If the user is a member of the named group in the named target, the user's provisioning object is also given the group. This way, when the synchronization occurs, the group is ignored.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.JITIgnoreGroups
  params:
    # The name of the group to ignore
    groupName: "Administrators"
    # The name of the provisioning target to search
    targetName: "adUsers"
```

## LoadGroups

The Load Groups task will load all the groups a user is a member of in Unison's virtual directory. It can also optionally load the "inverse," meaning only the groups the user is **not** going to be a member of after this task. This can be useful when deleting a user from a group.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.LoadGroups
  params:
    # The attribute name to search for on the user's account
    nameAttr: "uid"
    # If set to true, only loads the groups from the virtual directory that the user's object is NOT already a member of
    inverse: "false"
```

## LoadGroupsFromTarget

The Load Groups from Target task will load all the groups a user is a member of from a specific provisioning target. It can also optionally load the "inverse," meaning only the groups the user is **not** going to be a member of after this task. This can be useful when deleting a user from a group.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.LoadGroupsFromTarget
  params:
    # The attribute name to search for on the user's account
    nameAttr: "uid"
    # If set to true, only loads the groups from the virtual directory that the user's object is NOT already a member of
    inverse: "false"
    # Name of the target
    target: "some-db-target"
```

## LoadAttributesFromTarget

The Load Attributes from Target task will load the named attributes for a user from a specific provisioning target.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.LoadAttributesFromTarget
  params:
    # Name of the target
    target: "drupaldb"
    # The attribute name to search for on the user's account
    nameAttr: "mail"
    # List of attributes to load, can be listed multiple times
    attributes:
      - "uid"
```

## JITBasicDBCreateGroups

The Just-In-Time Create Groups task can create groups in a database table if they aren't present. This is useful when using a database to store group information in a cloud situation where the list of groups is unknown at deployment time. It is used in conjunction with a database provisioning target that has a group table defined.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.JITBasicDBCreateGroups
  params:
    # The name of a database provisioning target
    targetName: "jitDB"
```

## PrintUserInfo

The Print User Info task is useful when developing and debugging workflows. It will print the user's attributes to the Unison log file.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
  params:
    # An optional label to add to the log message
    message: "After Approval"
```

## CreateOTPKey

Creates an OATH key, used with the Time-Based One-Time Password authentication mechanism.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.CreateOTPKey
  params:
    # The name of the attribute to store the token in
    attributeName: "l"
    # The host name of the service, used for identification in the authenticator
    hostName: "www.someplace.com"
    # The name of the key used to encrypt and decrypt the user's token.
    # Can be obtained from the TOTP Authentication Mechanism on your Authentication Chain.
    encryptionKey: "lastmile-enc-totp"
```

## AddRoleTask

When used with the OpenStack Keystone provisioning target, this task makes it easier to add (or remove) a role from the user's roles attribute. This task will generate the proper JSON. All of the configuration options are parameter-aware.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.openstack.AddRoleTask
  params:
    # The name of the role, if not specified defaults to $role_name$
    name: "$role_name$"
    # The scope of the role, may be project or domain
    scope: "project"
    # The name of the domain, defaults to $project_domain_name$ if not specified
    domain: "$project_domain_name$"
    # The name of the project, defaults to $project_name$
    project: "$project_name$"
    # Set to true if the role should be removed from the user's object
    remove: "false"
```

## CreateMongoGroups

This custom task can be used in a workflow to create groups in your Mongo database that don't exist. This is useful if you are letting users dynamically determine what groups are used for authorizing access using dynamic workflows.

```yaml
- taskType: customTask
  className: com.tremolosecurity.mongodb.unison.CreateMongoGroups
  params:
    # Collection to create groups in if not found
    collectionName: "groups"
    # The target to search and create groups in
    targetName: "mymongodb"
    # Check a request attribute for a group name, like what might be used in a dynamic workflow
    requestAttributes: "approvalGroup"
```

## CallRemoteWorkflow

Calls a remote OpenUnison to execute a workflow. Authentication is done via LastMile. This task is built to work with the `com.tremolosecurity.proxy.filters.CallWorkflow` filter configured on a URL with the OAuth2 authentication mechanism.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.CallRemoteWorkflow
  params:
    # Name of the workflow to call on the remote OpenUnison server
    workflowName: "updateLockout"
    # Static key used to encrypt the LastMile token
    lastMileKeyName: "lastmile-portal"
    # URL to call
    url: "https://openunison.domain.lan:8443/workflows/call"
    # Attribute from LastMile user to identify the LastMile account
    lastMileUid: "uid"
    # List of request variables to include
    staticRequestValues: "UNISON.EXEC.TYPE=UNISON.EXEC.SYNC"
    # Name of the user to use in the LastMile request
    lastMileUser: "system"
    # Skew to allow for time drift across servers in milliseconds
    timeSkew: "60000"
    # Name of the attribute in the workflow object that identifies the user
    uidAttributeName: "uid"
```

## AddGroupToStore

This task will add a group to a named data store. The data store **MUST** implement `com.tremolosecurity.provisioning.core.UserStoreProviderWithAddGroup`.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.AddGroupToStore
  params:
    # The name of the target to create the group in
    target: "fedora.tremolo.dev"
    # The name of the group(s) to add, may be listed multiple times. Values can use values from the request object.
    name: "created-$DYN_NAME$-workflow"
    # Parameters passed into addGroup, may be listed multiple times
    attributes:
      - "name=value"
```

## AddGroupToRole

The AddGroupToRole task will add a group to a project role. Useful when onboarding a new project.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.openshiftv3.tasks.AddGroupToRole
  params:
    # Target to run against
    targetName: "openshift"
    # Project to add to, supports request parameters in between dollar signs
    projectName: "$project$"
    # Group to add, supports request parameters in between dollar signs
    groupName: "view-$project$"
    # Role to add to, supports request parameters in between dollar signs
    roleName: "view"
```

## CreateProject

This task creates an OpenShift project.

```yaml
- taskType: customTask
  className: com.tremolosecurity.unison.openshiftv3.tasks.CreateProject
  params:
    # Target to run against
    targetName: "openshift"
    # The JSON template of a ProjectRequest object, supports request parameters in between dollar signs
    template: >
      {
        "kind": "ProjectRequest",
        "apiVersion": "v1",
        "metadata": {
          "name": "$project$",
          "creationTimestamp": null
        }
      }
```

## CopyFromUserToRequest

This task will copy attributes from a user object to the request object.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks
  params:
    # The name of the attribute to copy, may be listed multiple times
    attribute: "projectName"
    # If false, the attribute is removed from the user object
    keepInUser: "false"
```

## CleanLabels

Cleans request attributes that are meant to become labels on Kubernetes objects. Labels must be compliant with DNS host names. This task will replace invalid characters and inject an "x" before or after the label if the leading or trailing characters don't properly match.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.CleanLabels
  params:
    # Character that will replace invalid characters in the named attributes
    replacementCharacter: "-"
    # The suffix to use on the new request attribute.
    # For instance, if cleaning "rawlabel" and setting this to "_clean",
    # you would reference "rawlabel_clean" in the tasks for creating objects.
    newAttributeSuffix: "_clean"
    # Request attributes to be cleaned, may be listed multiple times
    attributes:
      - "raw label"
```

## ClearGroups

Deletes all groups from a user's object.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.ClearGroups
  params: {}
```

## Env2Req

Copies environment variables to the workflow's request object. The name of the parameter is the name of the object in the workflow request to create, and the value is the name of the environment variable to get the value from.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.Env2Req
  params:
    # Maps environment variables to request attributes
    for_request: "from_environment"
```

## ClearPasswordResets

This task will clear out all password reset requests for the user.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.ClearPasswordResets
  params:
    # The name of the password reset mechanism
    mechName: "passwordReset"
```

## CopyGroupMembers

CopyGroupMembers will copy the members from one group to another. This is useful when dynamically generating access control groups from a workflow.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.CopyGroupMembers
  params:
    # A workflow for performing the copy (see example below)
    copyWorkflow: "addApproverUsers"
    # The name of the group to copy members to inside of a provisioning target
    copyTo: "add-approver-user"
    # The group in the virtual directory that is the source for members
    copyFrom: "cn=administrators,ou=groups,ou=shadow,o=Tremolo"
    # The name of the user id attribute
    uidAttributeName: "uid"
    # The requester for the audit trail
    requestor: "system"
```

### Example copyWorkflow

```yaml
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: add-approver-user
  namespace: openunison
  annotations:
    argocd.argoproj.io/sync-wave: "40"
spec:
  description: Add new approval users
  inList: false
  label: Add approver users
  orgId: 63ada052-881e-4685-834d-dd48a3aa4bb4
  tasks: |-
      - taskType: mapping
        strict: true
        map:
        - targetAttributeName: sub
          sourceType: user
          targetAttributeSource: uid
        onSuccess:
          - taskType: provision
            sync: false
            target: jitdb
            setPassword: false
            onlyPassedInAttributes: false
            attributes:
            - sub
```

## DoesGroupExist

This task will check to see if a group exists in a target and put the result in a request parameter.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.DoesGroupExist
  params:
    # The provisioning target to check
    target: "jitdb"
    # The name of the group to check
    groupName: "approvers-openshift-$name$"
    # The name of the request attribute to create
    attributeName: "tremolo.approval.group.exists"
```

## GenUUIDAttribute

A useful way to generate a unique ID.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.com.tremolosecurity.provisioning.customTasks
  params:
    # Name of the attribute to put the UUID in
    attributeName: "uuid"
```

## MapJitGroups

It is often useful to map from an external group to an internal group when just-in-time provisioning group access. This task provides a static map between groups.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.customTasks.MapJitGroups
  params:
    # The name of the attribute that stores the user's groups
    attributeName: "memberOf"
    # Each mapping is of the form internalgroup=externalgroup
    groupMap:
      - "k8s-cluster-administrators=CN=jit-k8s-admin,CN=Users,DC=ent2k12,DC=domain,DC=com"
      - "administrators=CN=ouadmins,CN=Users,DC=ent2k12,DC=domain,DC=com"
```

## CreateGitRepository

Creates a Git repository in ArgoCD, registering an SSH private key.

```yaml
- taskType: customTask
  className: com.tremolosecurity.argocd.tasks.CreateGitRepository
  params:
    # Type of repository
    type: "git"
    # Name of the repository
    name: "$nameSpace$-build"
    # SSH URL for the repository
    repoUrl: "$gitSshInternalURL$"
    # Plain text encoded SSH private key to register
    sshPrivateKey: "$gitPrivateKey$"
    # ArgoCD target name
    target: "argocd"
```

## AddtoRBAC

Appends to the RBAC ConfigMap in ArgoCD.

```yaml
- taskType: customTask
  className: com.tremolosecurity.argocd.tasks.AddtoRBAC
  params:
    # Kubernetes target
    k8sTarget: "k8s"
    # Rules to add
    toAdd: |
      p, role:$nameSpace$-operations, applications, get, $nameSpace$/*, allow
      p, role:$nameSpace$-operations, applications, override, $nameSpace$/*, allow
      p, role:$nameSpace$-operations, applications, sync, $nameSpace$/*, allow
      p, role:$nameSpace$-operations, applications, update, $nameSpace$/*, allow

      p, role:$nameSpace$-operations, projects, get, $nameSpace$, allow

      g, k8s-namespace-operations-$nameSpace$, role:$nameSpace$-operations

      p, role:$nameSpace$-dev, applications, get, $nameSpace$/*, allow
      p, role:$nameSpace$-dev, projects, get, $nameSpace$, allow
      g, k8s-namespace-developer-$nameSpace$, role:$nameSpace$-dev
```