# SCIM 2.0 Gateway

SCIM, [System for Cross-domain Identity Management](https://scim.cloud/), is a popular RESTful API for integrating applications into identity management systems such as SailPoint.  Not all applications provide SCIM endpoints or require more complex integrations then a simple gateway can provide.  For instance, it's not unusual for applications to provide a simle CRUD (Create, Read, Update, Delete) API that would require custom integration into your identity system.  OpenUnison's SCIM gateway makes it easier to provide integration for these use cases by creating a simpler implementation.  OpenUnison handles the work of processing SCIM calls for you, while a [custom provisioning target](/customization/custom_provisioning_targets/) does the work of interacting with your application's APIs.  Since these components can be implemented on their own and tested on their own, it makes for quicker development and faster integrations.

## Creating a Custom Target

The first step is to create a [custom provisioning target](/customization/custom_provisioning_targets/) to interact with your application.  For the rest of this tutorial, we will work with the example created in our documentation.  Once your target is created, make sure to deploy it to your OpenUnison.

## Authenticating to the Gateway

Your identity management system will need to authenticate to your gateway.  You can use any of OpenUnison's [Authentication Mechanisms](/documentation/reference/authentication-mechanisms/) as an authentication chain.  Since most OpenUnisons are deployed on Kubernetes, we're going to use a `ServiceAccount` token that's not bound to a `Pod`. 

***NOTE*** This is not a good production use of Kubernetes ServiceAccount tokens.  We're using it as a simple example.  In production, it would be better to have an identity provider issue a token based on a credentials grant, certificate authentication, or some other form of strong API authentication.

Once you have OpenUnison deployed:

```
kubectl create serviceaccount scim-gateway -n openunison
kubectl create token scim-gateway -n openunison
```
Keep this token in a safe place, we'll use it when configuring our downstream system.  Next, create an authentication chain using the [Kubernetes JWT mechanism](/documentation/reference/authentication-mechanisms/#oauth2k8sserviceaccount):

***If you haven't deployed the orchestra-login-portal chart***
```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: oauth2k8s
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.OAuth2K8sServiceAccount
  uri: "/auth/oauth2k8s"
  init: {}
  secretParams: []
```

Then create our chain:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: scim-gateway
  namespace: openunison
spec:
  authMechs:
  - name: oauth2k8s
    required: required
    params:
      # The name of the Kubernetes target
      k8sTarget: "k8s"
      # If true, the user will be looked up in the virtual directory
      linkToDirectory: "false"
      # The OU in the DN to use if the user isn't found in the virtual directory
      noMatchOU: "oauth2"
      # If not linked to a user, the claim to use as the user identifier
      uidAttr: "sub"
      # LDAP filter to lookup the user, can include attributes from the JWT
      lookupFilter: "(uid=${sub})"
      # The objectClass to use if the user can't be found
      defaultObjectClass: "inetOrgPerson"
      # HTTP Realm
      realm: "kubernetes"
      # HTTP Scope
      scope: "auth"
    secretParams: []
  level: 20
  root: o=Tremolo
```

Once deployed, we can create our SCIM gateway.  

## Deploying Our Gateway

So far, we've setup OpenUnison and configured it to authenticate a `ServiceAccount` token.  Next, let's deploy an `Applicaiton` with our gateway configured.  Here's an example with incline comments for documentation:

```yaml
---
apiVersion: openunison.tremolo.io/v2
kind: Application
metadata:
  labels:
    app.kubernetes.io/component: openunison-applications
    app.kubernetes.io/instance: openunison-orchestra-login-portal
    app.kubernetes.io/name: openunison
    app.kubernetes.io/part-of: openunison
  name: scim-example
  namespace: openunison
spec:
  azTimeoutMillis: 3000
  cookieConfig:
    cookiesEnabled: false
    domain: '#[OU_HOST]'
    httpOnly: true
    keyAlias: session-unison
    logoutURI: /logout
    scope: -1
    secure: true
    sessionCookieName: tremolosession
    timeout: 900
  isApp: true
  urls:
  - azRules:
    # limit access to our ServiceAccount
    - constraint: "username=system:serviceaccount:openunison:scim-gateway,ou=oauth2,o=Tremolo"
      scope: dn
    filterChain:
    - className: com.tremolosecurity.proxy.filters.scim.Scim
      params:
        # determine if the lookups are done via the internal LDAP virtual directory, usually false
        lookupFromLDAP: "false"

        # LDAP search base if lookFromLDAP is true, otherwise leave empty
        searchBase: ""

        # the name of the target to lookup users from
        lookupTarget: "custom-target"

        # the user's login id attribute
        uidAttributeName: userName

        # if there is a separate id attribute
        idAttributeName: userName

        # attribute to lookup users by
        lookupAttributeName: userName

        # workflow for synchronizing a user
        workflowName: sync-user-to-target

        # workflow for deleting a user
        deleteUserWorkflowName: delete-from-target

        # workflow for adding a user to, or removing from, a group
        groupWorkflow: target-group-membership

        # workflow for deleting groups (if supported)
        groupDeleteWorkflow: delete-group-from-target

        # attribute to look groups up by
        groupLookupAttributeName: name

        # attribute that stores a group's unique id, if supported
        groupIdName: cn

        # group display name attribute
        groupDisplayName: displayName

        # group's membership attribute
        groupMemberAttributeName: uniqueMember

        # list each attribute for your target that will be mapped to by scim attributes
        scim2tremolo:
          - username
          - mail
          - first
          - last

        # for each attribute, create a mapping from the SCIM attributes
        # user will come from the userName SCIM attribute
        scim2tremolo.username.source: userName
        scim2tremolo.username.sourceType: user

        # mail will come from emails, which is a complex type.  the JsonPathMapping
        # uses JSON Path syntax to extract the value
        scim2tremolo.mail.source: com.tremolosecurity.mappers.JsonPathMapping|emails,$[*].value
        scim2tremolo.mail.sourceType: custom

        # first will come from name, which is a complex type.  the JsonPathMapping
        # uses JSON Path syntax to extract the value
        scim2tremolo.first.source: com.tremolosecurity.mappers.JsonPathMapping|name,$.givenName
        scim2tremolo.first.sourceType: custom

        # last will come from name, which is a complex type.  the JsonPathMapping
        # uses JSON Path syntax to extract the value
        scim2tremolo.last.source: com.tremolosecurity.mappers.JsonPathMapping|name,$.familyName
        scim2tremolo.last.sourceType: custom

        # list each SCIM attribute your target will support
        tremolo2scim:
          - userName
          - emails
          - name
          - groups
        
        # for each SCIM attribute, configure how OpenUnison will map to them
        # userName will come from the user
        tremolo2scim.userName.source: username
        tremolo2scim.userName.sourceType: user

        # since emails is a json complex type (array of json objects), a special custom mapping will help.  The base 64 encoded parameter
        # is [{"type":"work","primary":true,"value":"${email}"}].  It's base64 encoded because custom parameters parameters to custom mappings
        # are comma separated
        tremolo2scim.emails.source: com.tremolosecurity.mappers.CompositeNoReturn|W3sidHlwZSI6IndvcmsiLCJwcmltYXJ5Ijp0cnVlLCJ2YWx1ZSI6IiR7ZW1haWx9In1d
        tremolo2scim.emails.sourceType: custom

        # since name doesn't require an array we can use generic custom mappings
        tremolo2scim.name.source: "{\"givenName\": \"${first}\", \"familyName\" : \"${last}\"}"
        tremolo2scim.name.sourceType: composite

        # groups is a read only attribute built from the user's group memberships.  The GenerateGroups mapper's first parameter should match 
        # lookupFromLDAP above, then include the URL prefix for your scim gateway, finally the name of the target you're loading
        # users from
        tremolo2scim.groups.source: com.tremolosecurity.mappers.GenerateGroups|false,https://#[OU_HOST]/api/scim-gateway,custom-target
        tremolo2scim.groups.sourceType: custom
        

        # mapping from a SCIM attribute to an LDAP filter attribute for users, comma separated
        userFilterScim2Ldap: emails.value=mail

        # mapping from a SCIM attribute to an LDAP filter attribute for groups, comma separated
        groupFilterScim2Ldap: description=cn,id=cn

        
    hosts:
    - '#[OU_HOST]'
    # the name of the authentication chain we created to authenticate from Kubernetes
    # service account tokens
    authChain: scim-gateway
    results: {}
    uri: /api/scim-gateway
```

With your `Application` and `AuthenticationChain` deployed, we can verify that we can load a user:


```sh
curl -H "Authorization: Bearer $(kubectl create token scim-gateway  -n openunison)" https://ouscim.domain.com/api/scim-gateway/Users/mmosley
```

will result in the JSON:

```json
{
  "id": "mmosley",
  "schemas": [
    "urn:ietf:params:scim:schemas:core:2.0:User"
  ],
  "emails": [
    {
      "type": "work",
      "primary": true,
      "value": "mmosley@nodomain.io"
    }
  ],
  "name": {
    "givenName": "Matt",
    "familyName": "Mosley"
  },
  "groups": [
    {
      "value": "group1",
      "$ref": "https://ouscim.domain.com/api/scim-gateway/Groups/group1"
    },
    {
      "value": "group2",
      "$ref": "https://ouscim.domain.com/api/scim-gateway/Groups/group2"
    },
    {
      "value": "group3",
      "$ref": "https://ouscim.domain.com/api/scim-gateway/Groups/group3"
    }
  ],
  "userName": "mmosley",
  "meta": {
    "resourceType": "User",
    "location": "https://ouscim.domain.com/api/scim-gateway/Users/mmosley/Users/mmosley"
  }
}
```

We have a working read-only gateway that can be called with curl or even integrated into a remote identity system.  If your remote identity system is schema aware, then it can poll the gateway to get just the attributes that you have configured in your gateway, making integration easier.  With search access working, next we'll create the workflows needed for creating, updating, and deleting users.

## User Management Workflows

So far, we have configured a gateway that allows us to search for users and groups.  The next step is to create `Workflow` objects that will do the work of updating users.  `Workflow` objects in OpenUnison are built around creating a user identity as it should exist, with the `Target` being responsible for making that a reality.  SCIM, on the other hand, is a CRUD (Create, Read, Update, Delete) API which means that it expects to perform individual operations on each object.  The gateway does the work of making these two different concepts come together.  There are three workflows we configured in our gateway that we now need to implement:

| Configuration Name | Value | Description |
| ------------------ | ----- | ----------- |
| `workflowName`       | `sync-user-to-target` | Synchronizes the user's attributes or creates the user.  This workflow is ***NOT*** responsible for group memberships |
| `groupWorkflow` | `target-group-membership` | Responsible for adding users to groups, not attributes |
| `deleteUserWorkflowName` | `delete-from-target` | Responsible for deleting a user |

OpenUnison's [workflow system](/documentation/configuring-openunison/#workflows-and-automation) has several built in tasks and can be extended with [custom tasks](/documentation/reference/provisioning-tasks/) or you can implement your own [custom task in javascript](/customization/custom_provisioning_tasks/) to provide more flexibility in mapping.  For our gateway, the workflows are pretty simple.

First, let's create the workflow for syncing our user:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: sync-user-to-target
  namespace: openunison
spec:
  description: update user
  inList: false
  label: update user
  orgId: internal-does-not-exist
  tasks: |-
    # useful for debugging
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
      params:
        message: enter workflow
    
    # provision to our target.  OpenUnison does all the work to build your input
    - taskType: provision
      sync: true
      target: "custom-target"
      attributes:
      - username
      - first
      - last
      - email
```

Once created, we can test adding a new user using curl:

```bash
curl -X POST \
  --insecure \
  -H "Content-Type: application/scim+json" \
  -H "Accept: application/scim+json" \
  -H "Authorization: Bearer $(kubectl create token scim-gateway  -n openunison)" \
  https://ouscim.domain.com/api/scim-gateway/Users \
  -d '{
    "schemas": [
      "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "jjackson",
    "name": {
      "givenName": "Jennifer",
      "familyName": "Jackson"
    },
    "emails": [
      {
        "value": "jjackson@nodomain.io",
        "type": "work",
        "primary": true
      }
    ],
    "active": true
  }'
```

In the OpenUnison logs, you'll see:

```
[[2026-01-19 15:45:18,852][XNIO-1 task-2] INFO  PrintUserInfo - enter workflow - jjackson - {mail=mail : 'jjackson@nodomain.io' , last=last : 'Jackson' , first=first : 'Jennifer' , username=username : 'jjackson' } / []
[2026-01-19 15:45:18,939][XNIO-1 task-2] INFO  custom-target - In find user
[2026-01-19 15:45:18,990][XNIO-1 task-2] INFO  custom-target - user: undefined
[2026-01-19 15:45:18,991][XNIO-1 task-2] INFO  custom-target - No user found
[2026-01-19 15:45:19,074][XNIO-1 task-2] INFO  ProvisioningEngineImpl - target=custom-target entry=true Add user=jjackson workflow=sync-user-to-target approval=0 username='jjackson'
[2026-01-19 15:45:19,076][XNIO-1 task-2] INFO  ProvisioningEngineImpl - target=custom-target entry=false Add user=jjackson workflow=sync-user-to-target approval=0 last='Jackson'
[2026-01-19 15:45:19,077][XNIO-1 task-2] INFO  ProvisioningEngineImpl - target=custom-target entry=false Add user=jjackson workflow=sync-user-to-target approval=0 first='Jennifer'
[2026-01-19 15:45:19,078][XNIO-1 task-2] INFO  ProvisioningEngineImpl - target=custom-target entry=false Add user=jjackson workflow=sync-user-to-target approval=0 username='jjackson'
```

The first log line from `PrintUserInfo` is from the `PrintUserInfo` custom task and is useful for debugging your workflows.  Next we see the `jjackson` object being created.  Finally, we see all the attributes being set.

At this point with can use HTTP PATCH SCIM api calls to change the user's attributes, but not their memberships.  In order to add our new user to a group, we need to create a group membership workflow:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: target-group-membership
  namespace: openunison
spec:
  description: group members
  inList: false
  label: group members
  orgId: internal-does-not-exist
  tasks: |-
    # print out the current user object
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
      params:
        message: in group member

    # load the existing groups from our target to make updates
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.LoadGroupsFromTarget
      params:
        nameAttr: username
        inverse: "false"
        target: "custom-target"

    # add or remove the group, depending on what the gateway determines needs to happen
    # the $groupname$ is the group being manipulated, while $removegroup$ will be true if
    # the group is being removed from the user, and false if the group is being added.
    # these are entries in the request Map object and are generated by the gateway
    - taskType: addGroup
      name: "$groupname$"
      remove: "$removegroup$"

    # the new user once groups are manipulated
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
      params:
        message: added group
    
    # synchronize our user.  Only act on groups and the username attribute
    # to make sure none of our other attributes are changed
    - taskType: provision
      sync: true
      target: "custom-target"
      attributes:
      - username
```

With our workflow deployed, we can now test adding our user to a group:

```bash
  curl -X PATCH \
  --insecure \
  -H "Content-Type: application/scim+json" \
  -H "Accept: application/scim+json" \
  -H "Authorization: Bearer $(kubectl create token scim-gateway  -n openunison)" \
  https://ouscim.domain.com/api/scim-gateway/Groups/group2 \
  -d '{
    "schemas": [
      "urn:ietf:params:scim:api:messages:2.0:PatchOp"
    ],
    "Operations": [
      {
        "op": "add",
        "path": "members",
        "value": [
          {
            "value": "jjackson",
            "type": "User"
          }
        ]
      }
    ]
  }'
```

In our logs, we'll see:

```
[2026-01-19 16:04:56,761][XNIO-1 task-2] INFO  PrintUserInfo - in group member - jjackson - {userName=userName : 'jjackson' , username=username : 'jjackson' } / []
[2026-01-19 16:04:56,791][XNIO-1 task-2] INFO  custom-target - In find user
[2026-01-19 16:04:56,794][XNIO-1 task-2] INFO  custom-target - user: {"username":"jjackson","last":"Jackson","first":"Jennifer","groups":[]}
[2026-01-19 16:04:56,795][XNIO-1 task-2] INFO  custom-target - Returning user with attributes [last, first, email, username]
[2026-01-19 16:04:56,860][XNIO-1 task-2] INFO  PrintUserInfo - added group - jjackson - {userName=userName : 'jjackson' , username=username : 'jjackson' } / [group2]
[2026-01-19 16:04:56,891][XNIO-1 task-2] INFO  custom-target - In find user
[2026-01-19 16:04:56,894][XNIO-1 task-2] INFO  custom-target - user: {"username":"jjackson","last":"Jackson","first":"Jennifer","groups":[]}
[2026-01-19 16:04:56,895][XNIO-1 task-2] INFO  custom-target - Returning user with attributes [username]
[2026-01-19 16:04:56,945][XNIO-1 task-2] INFO  ProvisioningEngineImpl - target=custom-target entry=false Add user=jjackson workflow=target-group-membership approval=0 group='group2'
```

We can see that in our workflow first the user jjackson had no groups, then had group2, and then group2 was added to the user.  Finally, we'll want to delete our user:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: delete-from-target
  namespace: openunison
spec:
  description: delete user
  inList: false
  label: delete user
  orgId: internal-does-not-exist
  tasks: |-
    # debug to print who the user is that's getting deleted
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.PrintUserInfo
      params:
        message: in delete workflow

    # delete the user
    - taskType: delete
      target: custom-target
```

Now let's test deleting our `jjackson` user:

```bash
curl -X DELETE \
  --insecure \
  -H "Accept: application/scim+json" \
  -H "Authorization: Bearer $(kubectl create token scim-gateway  -n openunison)" \
  https://ouscim.tremolo.dev/api/scim-gateway/Users/jjackson
```

and in our logs:

```
[2026-01-19 16:10:46,586][XNIO-1 task-2] INFO  PrintUserInfo - in delete workflow - jjackson - {last=last : 'Jackson' , userName=userName : 'jjackson' , first=first : 'Jennifer' , username=username : 'jjackson' } / []
[2026-01-19 16:10:46,637][XNIO-1 task-2] INFO  ProvisioningEngineImpl - target=custom-target entry=true Delete user=jjackson workflow=delete-from-target approval=0 username='jjackson'
```

You now have a functioning gateway that can be plugged into your remote identity management system!

## Running Multiple Gateways

Now that you've deployed your first gateway, you don't need to redeploy OpenUnison for each additional gateway!  Create and deploy an appropriate `Target`, create and deploy the correct `Workflow` objects, then deploy a new gateway `Application` with a new name and 