# Provisioning Targets

This section details the pre-built provisioning targets that are available for Unison. In addition to these targets, custom targets may be created. Consult the Unison SDK for instructions on how to create a custom target.

All targets have a common interface for specifying mappings from Unison?s current user object and how attributes will be pushed to the target. Only mapped attributes will be utilized by a provisioning target.

| Source Type | Description                                                                                         | Source                                | Example                                 |
|-------------|-----------------------------------------------------------------------------------------------------|---------------------------------------|-----------------------------------------|
| user        | Map an attribute from the user’s directory object                                                  | Name of an attribute                  | givenName                               |
| static      | A static value that doesn’t change                                                                 | The static value                      | Myvalue                                 |
| custom      | A class that is used to determine the mapping                                                      | Class name, see the SDK for details on how to implement | com.mycompany.mapper.Mapper  |
| composite   | A composite of attributes and static values. Attributes are defined with `${attributename}`. Only attributes that exist before the mappings are run are available | Static and attribute data             | `${givenName}.${sn}@mydomain.com`       |

Note that if the source attribute is `TREMOLO_USER_ID` then the user object?s id is used. When `TREMOLO_USER_ID` is the target attribute it sets user object?s id.


## OpenShiftTarget

This target provides provisioning capabilities for Kubernetes and Red Hat OpenShift.  There are three ways this target can communicate with a cluster:

| Option | Description | Dependencies |
| ------ | ------------ | ----------- |
| Local `ServiceAccount` | When running in-cluster, this target will take OpenUnison's local `ServiceAccount` to communicate with the API server.  OpenUnison knows to reload the `ServiceAccount` as it gets rotated for expiration | Set `spec.params.url` to `https://kubernetes.default.svc` and `spec.params.useToken` to `true` |
| Projected token | Similar to the local option, but uses a non default location for the token.  This is useful if you're using a projected token for a shorter length in time | Set `spec.params.tokenType` to `tokenapi` and configure `spec.params.tokenPath` |
| Certificate | You can import a key pair into OpenUnison via a `Secret` and use that keypair. | Create a `Secret` with both a tls.key and tls.crt, add it to be imported by the helm chart and set `spec.params.tokenType` to `certificate`.  OpenUnison will determine the correct key pair to use while connecting to the cluster |
| oidc | OpenUnison can generate short lived tokens just-in-time so that there are no static shared tokens between OpenUnison and a remote cluster. This is the most secure way for OpenUnison to communicate with a remote cluster because you don't need to have a pre-shared static token. | Configure your remote cluster to trust an OpenUnison identity provider, set `spec.params.tokenType` to `oidc` and set `spec.params.oidc*` parameters below |
| Static token | This is the least secure option, as it requires a static token to be stored in your control plane cluster. | Set `spec.params.tokenType` to `static` and create a `Secret` to store the token |

```yaml
apiVersion: openunison.tremolo.io/v1
kind: Target
metadata:
  annotations:
    # Add this label to provide a label in the OpenUnison portal
    tremolo.io/cluster-label: sat6
  # all automation assumes Kubernetes targets start with "k8s-"
  name: k8s-sat6
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.openshiftv3.OpenShiftTarget
  params:
  # The URL of the cluster
  - name: url
    value: https://oumgmt-proxy.192-168-2-77.nip.io
  # should always be "true"
  - name: useToken
    value: "true"
  # A Base64 encoded PEM of the remote cluster's certificate, if needed
  - name: certificate
    value: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURFVENDQWZtZ0F3SUJBZ0lVYmtiS2ZRN29ldXJuVHpyeWdIL0dDS0kzNkUwd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0dERVdNQlFHQTFVRUF3d05aVzUwWlhKd2NtbHpaUzFqWVRBZUZ3MHlNakV4TURjeE5EUTFNakphRncwegpNakV4TURReE5EUTFNakphTUJneEZqQVVCZ05WQkFNTURXVnVkR1Z5Y0hKcGMyVXRZMkV3Z2dFaU1BMEdDU3FHClNJYjNEUUVCQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUUNucVZ3eVFvMjJyRzZuVVpjU2UvR21WZnI5MEt6Z3V4MDkKNDY4cFNTUWRwRHE5UlRRVU92ZkFUUEJXODF3QlJmUDEvcnlFaHNocnVBS2E5LzVoKzVCL3g4bmN4VFhwbThCNwp2RDdldHY4V3VyeUtQc0lMdWlkT0QwR1FTRVRvNzdBWE03RmZpUk9yMDFqN3c2UVB3dVB2QkpTcDNpa2lDL0RjCnZFNjZsdklFWE43ZFNnRGRkdnV2R1FORFdPWWxHWmhmNUZIVy81ZHJQSHVPOXp1eVVHK01NaTFpUCtSQk1QUmcKSWU2djhCcE9ncnNnZHRtWExhNFZNc1BNKzBYZkQwSDhjU2YvMkg2V1M0LzdEOEF1bG5QSW9LY1krRkxKUEFtMwpJVFI3L2w2UTBJUXVNU3c2QkxLYWZCRm5CVmNUUVNIN3lKZEFKNWdINFZZRHIyamtVWkwzQWdNQkFBR2pVekJSCk1CMEdBMVVkRGdRV0JCU2Y5RDVGS3dISUY3eFdxRi80OG4rci9SVFEzakFmQmdOVkhTTUVHREFXZ0JTZjlENUYKS3dISUY3eFdxRi80OG4rci9SVFEzakFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQQpBNElCQVFCN1BsMjkrclJ2eHArVHhLT3RCZGRLeEhhRTJVRUxuYmlkaFUvMTZRbW51VmlCQVhidUVSSEF2Y0phCm5hb1plY0JVQVJ0aUxYT2poOTFBNkFvNVpET2RETllOUkNnTGI2czdDVVhSKzNLenZWRmNJVFRSdGtTTkxKMTUKZzRoallyQUtEWTFIM09zd1EvU3JoTG9GQndneGJJQ1F5eFNLaXQ0OURrK2V4c3puMUJFNzE2aWlJVmdZT0daTwp5SWF5ekJZdW1Gc3M0MGprbWhsbms1ZW5hYjhJTDRUcXBDZS9xYnZtNXdOaktaVVozamJsM2QxVWVtcVlOdVlWCmNFY1o0UXltQUJZS3k0VkUzVFJZUmJJZGV0NFY2dVlIRjVZUHlFRWlZMFRVZStYVVJaVkFtaU9jcmtqblVIT3gKMWJqelJxSlpMNVR3b0ZDZzVlZUR6dVk0WlRjYwotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  # One of:
  # legacy - Use the local token
  # token - a static token
  # certificate - key pair for TLS authentication
  # oidc - use an openunison identity provider to generate short tokens just-in-time
  - name: tokenType
    value: token
  # If tokenType is oidc
  ## The name of the OIDC Application identity provider
  # - name: oidcIdp
  #   value: remotek8s
  ## The sub claim to use in the JWT
  # - name: oidcSub
  #   value: openunison-control-plane
  ## The audience for the JWT
  # - name: oidcAudience
  #   value: https://oumgmt-proxy.192-168-2-77.nip.io/
  
  # if tokenType is token
  ## Specifies the path to the token 
  # - name: tokenPath
  #   value: /path/to/token
  ## Path to the cluster certificate
  # - name: certPath
  #   value: /path/to/ca.crt
  
  # label for the target, used in the  OpenUnison portal
  - name: label
    value: sat6
  # List of queues to write objects to for DR
  - name: drqueues
    value: ""
  # When integrated with GitOps, the URL for git for the cluster repo
  - name: gitUrl
    values: ""
  # Set a timeout in millisenconds, default is 30,000
  - name: timeout
    value: "30000"
  secretParams: []
  ## if the tokenType is static
  #  - name: token
  #   secretName: orchestra-secrets-source
  #   secretKey: token
  
  
  targetAttributes:
  - name: fullName
    source: displayName
    sourceType: user
```

## K8sCrdUserProvider

This target will create User objects in Kubernetes.  Its meant to be used with K8sCrdInsert in the virtual directory.  Prior to using this insert the below CRD MUST be created.

```yaml
apiVersion: openunison.tremolo.io/v1
kind: Target
metadata:
  name: jitdb
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.providers.K8sCrdUserProvider
  params:
  # The name of the openshift provisioning target to use
  - name: k8sTarget
    value: k8s
  # The namespace to create users in
  - name: nameSpace
    value: openunison
  secretParams: []
  targetAttributes:
  - name: first_name
    source: first_name
    sourceType: user
  - name: last_name
    source: last_name
    sourceType: user
  - name: email
    source: email
    sourceType: user
  - name: sub
    source: sub
    sourceType: user
  - name: uid
    source: uid
    sourceType: user
```

## RbacBindingsTarget

While it's best to assign users to groups and map bindings to groups, this isn't always possible.  The RbacBindingsTarget will provision users and service accounts directly to `ClusterRoleBindings` and `Rolebindings`.

***NOTE: This target ONLY provisions access to bindings.  It will not create a `ServiceAccount`.  If you need to create a `ServiceAccount`, do so with the standard `OpenShiftTarget`.

### Definig Groups

Since the concept of "groups" doesn't map directly to bindings, they need to be specified in a specific way:

| Scope | Definition | Example |
| ----- | ---------- | ------- |
| `ClusterRoleBinding` | `crb:crb-name` | `crb:cluster-admins` |
| `RoleBinding` | `rb:namspace:rb-name` | `rb:myns:admin` |

If a group doesn't have the correct format, it's ignored.

### Service Accounts

When provisioning Kubernetes `ServiceAccounts` the user's "id" must be in the form of `serviceaccountname:namespace`.  For instance, to provision the `ServiceAccount` `mysa` in the namespace `myns` the user id would be `mysa:myns`.

```yaml
apiVersion: openunison.tremolo.io/v1
kind: Target
metadata:
  name: k8s-rbac
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.targets.RbacBindingsTarget
  params:
  # The name of the openshift/kubernetes provisioning target to use
  - name: target
    value: k8s
  # if true, then the target is designed to work with ServiceAccounts.  If false, generic users.
  - name: forSa
    value: "true"
  secretParams: []
  targetAttributes: []
```

## LDAPProvider

This target provisions identities to a generic LDAPv3 directory.

```yaml
---
kind: Target
metadata:
  name: ldap2
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.core.providers.LDAPProvider
  params:
    # The object class for new user objects
    - name: objectClass
      value: inetOrgPerson
    # Host for the ldap server
    - name: host
      value: 127.0.0.1
    # The port to connect to
    - name: port
      value: "10983"
    # A DN for a user with administrator rights to create and update accounts
    - name: adminDN
      value: cn=admin,dc=domain,dc=com
    
    # The DN pattern for new users with user attributes in ${}
    - name: dnPattern
      value: uid=${uid},ou=internal,dc=domain,dc=com
    # The base that should be used for searching for users and groups
    - name: searchBase
      value: dc=domain,dc=com
    # The name of the attribute used to identify the user
    - name: userIDAttribute
      value: uid
    # If set to true SSL is used for the connection
    - name: useSSL
      value: "false"
    # Maximum number of connections to the directory
    - name: maxCons
      value: "10"
    # Maximum number of individual operations per connection
    - name: threadsPerCons
      value: "10"
    # Milliseconds an idle connection can stay open
    - name: idleTimeout
      value: "90000"
    # Allow for users from outside the directory to be provisioned into directory groups
    - name: allowExternalUsers
      value: "false"
  secretParams:
  # Credential passwords
  - name: adminPasswd
    secretName: orchestra-secrets-source
    secretKey: adminPasswd
  targetAttributes:
    - name: uid
      source: TREMOLO_USER_ID
      sourceType: user
    - name: sn
      source: sn
      sourceType: user
    - name: l
      source: l
      sourceType: user
    - name: cn
      source: cn
      sourceType: user
    - name: givenName
      source: givenName
      sourceType: user
```

***NOTE:*** To specify an alternate dnPattern in a workflows, specify it in the request object with the key `LDAPProvider.NEW_USER_DN` or `com.tremolosecurity.unison.provisioning.ldap.newUserDN`

## ADProvider

This target provisions identities to a Microsoft Active Directory. Note that unlike the Active Directory directory type, the provisioning target does NOT automatically map to an inetOrgPerson object class.

```yaml
---
kind: Target
metadata:
  name: activedirectory
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.core.providers.ADProvider
  params:
    # Host for the LDAP server
    - name: host
      value: ad.tremolo.lan
    # The port to connect to
    - name: port
      value: "636"
    # A DN for a user with administrator rights to create and update accounts
    - name: adminDN
      value: cn=Administrator,cn=Users,dc=enterprise,dc=domain,dc=com
    # The DN pattern for new users with user attributes in ${}
    - name: dnPattern
      value: cn=${cn},cn=Users,dc=enterprise,dc=domain,dc=com
    # The base that should be used for searching for users and groups
    - name: searchBase
      value: dc=enterprise,dc=domain,dc=com
    # The name of the attribute used to identify the user
    - name: userIDAttribute
      value: samAccountName
    # If set to true SSL is used for the connection
    - name: useSSL
      value: "true"
    # Maximum number of connections to the directory
    - name: maxCons
      value: "10"
    # Maximum number of individual operations per connection
    - name: threadsPerCons
      value: "10"
    # If set to true a shadow account is created
    - name: createShadowAccount
      value: "true"
    # Milliseconds an idle connection can stay open
    - name: idleTimeout
      value: "90000"
    # Allow for users from outside the directory to be provisioned into directory groups
    - name: allowExternalUsers
      value: "false"
  secretParams:
  # Credential passwords
  - name: adminPasswd
    secretName: orchestra-secrets-source
    secretKey: adminPasswd
  targetAttributes:
    - name: uid
      source: TREMOLO_USER_ID
      sourceType: user
    - name: lastname
      source: sn
      sourceType: user
    - name: givenName
      source: staticValue
      sourceType: static
    - name: cn
      source: com.tremolosecurity.test.provisioning.CreateCN
      sourceType: custom
    - name: login
      source: "${firstname}.${sn}@TEST"
      sourceType: composite
```

### Dynamic Group Creation

The ADProvider supports creating groups dynamically.  When calling AddGroupToTarget the `base` parameter must be added to tell the provider where to create the new group.

## SugarCRM

The SugarCRM target can be used to update contacts inside of SugarCRM. It does not, at present support the creating of users.

```yaml
---
kind: Target
metadata:
  name: sugarcrm
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.core.providers.TremoloTarget
  params:
    # The SugarCRM web services URL
    - name: url
      value: http://sugarcrm.domain.com/sugarcrm/service/v2/rest.php
    # Administrative username
    - name: adminUser
      value: admin
  secretParams:
  # The user’s password
  - name: adminPwd
    secretName: orchestra-secrets-source
    secretKey: adminPwd
  targetAttributes:
    - name: login
      source: uid
      sourceType: user
    - name: last
      source: sn
      sourceType: user
    - name: first
      source: givenName
      sourceType: user
```

## FreeIPATarget

This target is for FreeIPA / Red Hat Identity Management (http://www.freeipa.org/page/Main_Page / https://access.redhat.com/products/identity-management-and-infrastructure). The provisioning target allows for the creation of users, updating of attributes and groups as well as setting passwords. In addition, the target can generate "shadow objects" designed to work with SSO and constrained delegation where a password shouldn't be known.

```yaml
---
kind: Target
metadata:
  name: rhelent.lan
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.freeipa.FreeIPATarget
  params:
    # The protocol and host of the FreeIPA IPA-Web server. Do NOT include any path information
    - name: url
      value: https://freeipa.rhelent.lan
    # The user name (uid attribute) of a member of the admins group
    - name: userName
      value: adminx
    # If true, when a user is created a random password is generated so that the account is active and ready for use, but not usable with a password
    - name: createShadowAccounts
      value: "true"
    # If true, this FreeIPA target will work with cross-domain trusts. The user ID will need to be the user's userPrincipalName, not uid
    - name: multiDomain
      value: "false"
    # If multiDomain is true, this will tell the target the primary domain
    - name: primaryDomain
      value: rhelent.lan
    # Determines the Trust View to use with multiDomain, default to Default Trust View
    - name: trustView
      value: Default Trust View
  secretParams:
  # The password of the service account used to create accounts
  - name: password
    secretName: orchestra-secrets-source
    secretKey: password
  targetAttributes:
    - name: sn
      source: sn
      sourceType: user
    - name: givenname
      source: givenname
      sourceType: user
    - name: mail
      source: mail
      sourceType: user
    - name: uid
      source: uid
      sourceType: user
    - name: cn
      source: cn
      sourceType: user
    - name: displayname
      source: displayname
      sourceType: user
    - name: gecos
      source: gecos
      sourceType: user
```

## MongoDBTarget

The MongoDB provisioning target provides the capability to provision users and user groups in a single
database.  It is meant to work with the virtual directory insert.  Both users and groups can be in
any collection.  While MongoDB has no sense of an "objectClass", this attribute is used to distinguish
between users and groups.  Group memberships are stored as an attribute value on a group with an identifier,
NOT a distinguished name.

```yaml
---
kind: Target
metadata:
  name: mymongodb
  namespace: openunison
spec:
  className: com.tremolosecurity.mongodb.unison.MongoDBTarget
  params:
    # The MongoDB connection URL per https://docs.mongodb.com/manual/reference/connection-string/
    - name: url
      value: mongodb://dbs.tremolo.lan:27017
    # The name of the database to use
    - name: database
      value: unisonprov
    # The value of the "objectClass" attribute for users
    - name: userObjectClass
      value: inetOrgPerson
    # The RDN attribute for users
    - name: userRDN
      value: uid
    # The user identifier attribute
    - name: userIdAttribute
      value: uid
    # The group identifier attribute
    - name: groupIdAttribute
      value: cn
    # The group objectClass
    - name: groupObjectClass
      value: groupOfUniqueNames
    # The RDN of group objects
    - name: groupRDN
      value: cn
    # Group attribute that stores members
    - name: groupMemberAttribute
      value: uniqueMember
    # The user attribute used as the value for group membership
    - name: groupUserIdAttribute
      value: uid
    # If true, groups may point to users in the virtual directory that are NOT in MongoDB
    - name: supportExternalUsers
      value: "true"
    # The name of the attribute to store the object's collection in
    - name: collectionAttributeName
      value: collection
  secretParams: []
  targetAttributes:
    - name: sn
      source: sn
      sourceType: user
    - name: givenname
      source: givenname
      sourceType: user
    - name: mail
      source: mail
      sourceType: user
    - name: uid
      source: uid
      sourceType: user
    - name: cn
      source: cn
      sourceType: user
    - name: collection
      source: collection
      sourceType: user
    - name: objectClass
      source: objectClass
      sourceType: user
```

### Creating Groups

To create a group in MongoDB make sure the following attributes are added:

* unisonRdnAttributeName - Tells OpenUnison what the rdn will be
* The attribute named in unisonRdnAttributeName
* objectClass - To identify groups

## Drupal8Target

The Drupal8 target will work with the json API for JSON.

```yaml
---
kind: Target
metadata:
  name: drupal
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.drupal.drupal8.provisioning.Drupal8Target
  params:
    # The URL for your Drupal site
    - name: url
      value: http://192.168.56.102
    # Administrative credentials
    - name: user
      value: admin
  secretParams:
  # administrator credentials password
  - name: password
    secretName: orchestra-secrets-source
    secretKey: password
  targetAttributes:
    - name: name
      source: name
      sourceType: user
      targetType: string
    - name: mail
      source: mail
      sourceType: user
      targetType: string
    - name: first_name
      source: first_name
      sourceType: user
      targetType: string
    - name: last_name
      source: last_name
      sourceType: user
      targetType: string
    - name: status
      source: "true"
      sourceType: static
      targetType: string
```

## OktaTarget

The Okta target will provision objects to your Okta service.  It will provision users and group memberships. 

```yaml
---
kind: Target
metadata:
  name: okta
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.okta.provisioning.OktaTarget
  params:
    # Your Okta domain
    - name: domain
      value: xyz.okta.com
  secretParams:
  # The token from the "API" section of your Okta domain administration console
  - name: token
    secretName: orchestra-secrets-source
    secretKey: token
  targetAttributes:
    - name: login
      source: uid
      sourceType: user
      targetType: string
    - name: email
      source: mail
      sourceType: user
      targetType: string
    - name: firstName
      source: givenName
      sourceType: user
      targetType: string
    - name: lastName
      source: sn
      sourceType: user
      targetType: string
```

## AzureAD 

The AzureAD target can provision and accounts and groups to AzureAD tennants via the Graph API.  It can create users, assign groups and invite guest accounts.  In order to use this target, you'll need to register an application in AzureAD and generate an access secret.  You'll also need to request the following roles:

* Group.Create
* Group.ReadWrite.All
* GroupMember.ReadWrite.All
* User.Invite.All
* User.ManageIdentities.All
* User.Read
* User.ReadWrite.All

```yaml
---
kind: Target
metadata:
  name: azuread
  namespace: openunison
spec:
  className: ""
  params:
    # The AzureAD Client ID
    - name: clientId
      value: sdfsdfsdfsd
    # The tenant ID
    - name: tenantId
      value: sfgsdfsfd
  secretParams:
  # The secret key
  - name: clientSecret
    secretName: orchestra-secrets-source
    secretKey: clientSecret
  targetAttributes:
    - name: displayName
      source: displayName
      sourceType: user
      targetType: string
    - name: givenName
      source: givenName
      sourceType: user
      targetType: string
    - name: surname
      source: surname
      sourceType: user
      targetType: string
    - name: userPrincipalName
      source: userPrincipalName
      sourceType: user
      targetType: string
    - name: id
      source: id
      sourceType: user
      targetType: string
    - name: mail
      source: mail
      sourceType: user
      targetType: string
    - name: accountEnabled
      source: accountEnabled
      sourceType: user
      targetType: boolean
```

| Request Attribute                     | Description                                          | Example |
|---------------------------------------|------------------------------------------------------|---------|
| tremolo.azuread.external              | If true, the creation is an invitation of an external guest |         |
| tremolo.azuread.invitation.redirect   | For invitations, the URL to redirect users to        |         |
| tremolo.azuread.invitation.message    | For invitations, the message to be included in the email |         |

## GitlabUserProvider

The GitlabUserProvider can provision users and groups to GitLab.  It's capable of creating both local users and "identities" for integration with SSO.  It can also provision
users into groups.  Any attribute that is part of the User object in GitLab can be set.  Use the java bean notation.  When creating an "identity", make sure to add it to the
request object.  All group memberships are assumed to be Developer level unless otherwise mapped as shown below.

```yaml
---
kind: Target
metadata:
  name: gitlab
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.gitlab.provisioning.targets.GitlabUserProvider
  params:
    # The URL of your GitLab service
    - name: url
      value: "#[GITLAB_URL]"
  secretParams:
   # A personal access token for an administrator account
   - name: token
     secretName: orchestra-secrets-source
     secretKey: token
  targetAttributes:
    - name: username
      source: username
      sourceType: user
    - name: email
      source: email
      sourceType: user
    - name: name
      source: name
      sourceType: user
    - name: isAdmin
      source: isAdmin
      sourceType: user
    - name: skipConfirmation
      source: "true"
      sourceType: static
    - name: projectsLimit
      source: "100000"
      sourceType: static
```

#### Create Identity

```javascript
GitlabFedIdentity idToCreate = new GitlabFedIdentity();
idToCreate.setExternalUid("mmosley2");
idToCreate.setProvider("openid_connect");

ArrayList<GitlabFedIdentity> idsToCreate = new ArrayList<GitlabFedIdentity>();
idsToCreate.add(idToCreate);

request.put(GitlabUserProvider.GITLAB_IDENTITIES,idsToCreate);
```

#### Map Groups 

```javascript
HashMap<String,Integer> groupMap = new HashMap<String,Integer>();
groupMap.put("gitlab-group-1",AccessLevel.GUEST.toValue());
groupMap.put("gitlab-group-2",AccessLevel.GUEST.toValue());

request.put(GitlabUserProvider.GITLAB_GROUP_ENTITLEMENTS,groupMap)
```

## ArgoCDTarget

The ArgoCDTarget is primarily a place-holder for working with ArgoCD.  Most operations can be handled directly by provisioning
objects into the argocd namespace, but this target can be used for direct API integration.  An example is provisioning new git
repositories.  Use a token provisioned through the CLI.

```yaml
---
kind: Target
metadata:
  name: argocd
  namespace: openunison
spec:
  className: com.tremolosecurity.argocd.targets.ArgoCDTarget
  params:
    # The URL for your ArgoCD service
    - name: url
      value: "#[ARGOCD_URL]"
  secretParams:
  # API token to use
  - name: token
    secretName: orchestra-secrets-source
    secretKey: token
  targetAttributes: []
```

## MatterMostProvider

Onboard users into MatterMost chat platform.  Delete "disables" users, not deletes them fully.

```yaml
---
kind: Target
metadata:
  name: mattermost
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.core.providers.MatterMostProvider
  params:
    # URL for Mattermost
    - name: matterMostUrl
      value: https://mattermost-url.domain.com
  secretParams:
  # Access token for Mattermost
  - name: accessToken
    secretName: orchestra-secrets-source
    secretKey: accessToken
  targetAttributes:
    - name: email
      source: email
      sourceType: user
      targetType: string
    - name: username
      source: username
      sourceType: user
      targetType: string
    - name: first_name
      source: first_name
      sourceType: user
      targetType: string
    - name: last_name
      source: last_name
      sourceType: user
      targetType: string
    - name: auth_service
      source: auth_service
      sourceType: user
      targetType: string
    - name: auth_data
      source: auth_data
      sourceType: user
      targetType: string
```