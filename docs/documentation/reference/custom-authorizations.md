# Custom Authorizations

This section details the pre-built Unison custom authorization rules. These rules can be used in your deployments without change. Consult the Unison SDK for instructions on how to create custom custom authorization rules.

All custom authorization rules have a common interface for specifying configuration options. Each rule can take any number of name/value pairs. A single configuration option can have multiple values by listing the name/value pair for each value.

## ManagerAuthorization

This rule provides a mechanism for allowing a user's supervisor or manager to approve a request. The approver is authorized based on the user's data in Unison's internal virtual directory.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: manager-authorization
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.az.ManagerAuthorization
  params:
    # Use this to define what "distance" of a manager is authorized. For instance to allow the user's manager's manager to perform the authorization specify "2". This allows for escalations to be processed with multiple tiers of managers.
    numLevels: "1"

    # The name of the attribute on the user's directory object that will tell Unison who their manager is.
    managerID: manager

    # If true, OpenUnison will assume that the attribute defined in "managerID" is the distinguished name of the user's manager. If not checked, then OpenUnison will use a filter built from the the user id attribute
    managerIDIsDN: "true"

    # Used when the number of levels is greater then one, allowing all the managers between the user and the current step to approve. For instance of an approval is escalated to a user's manager's manager checking this option will allow both the user's manager AND the user's manager's manager to approve.

    allowLowerManagers: "false"
  secretParams: []
```

### Example azRule

There are no configuration options on the constraint.

```yaml
azRules:
- scope: custom
  constraint: manager-authorization
```

## GithubTeamRule

This rule is a convinience for authorizing multiple GitHub orgnaizations and teams without listing multiple LDAP filters.  In the rule, list each team as `org/team` and each org as `org/`.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: github
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.proxy.auth.github.GithubTeamRule
  params: {}
  secretParams: []
```

### Example azRule

After naming the custom authorization object, list each team as `org/team` and each org as `org/`.

```yaml
azRules:
- scope: custom
  constraint: github!MyOrg/!MyOrg/MyTeam
```

## GithubTeamRule

This rule is a convinience for authorizing multiple GitHub orgnaizations and teams without listing multiple LDAP filters.  In the rule, list each team as `org/team` and each org as `org/`.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: github
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.proxy.auth.github.GithubTeamRule
  params: {}
  secretParams: []
```

### Example azRule

After naming the custom authorization object, list each team as `org/team` and each org as `org/`.

```yaml
azRules:
- scope: custom
  constraint: github!MyOrg/!MyOrg/MyTeam
```

## RBACBindingAuthorization

Use a `ClusterRoleBinding` or a `RoleBinding` in Kubernetes to make an authorization decision.


```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: rbacaz
  namespace: openunison
spec:
  className: com.tremolosecurity.az.RBACBindingAuthorization
  params: {}
  secretParams: []
```

### Example azRule

In addition to the name of rule, you'll need to provide the following in order:

| Parameter | Description | Example |
| --------- | ----------- | ------- |
| Target | The name of the `Target` that connects to the Kubernetes cluster where the binding is configured | k8s |
| Scope | Will the binding be against a `ClusterRole` or a `Role` | `ClusterRole` or `Role` |
| Name | The name of the binding | admin |
| Lookup attribute | The name of the user attribute provided by the binding's `name` for each subject | mail |
| Search Base | Where in OpenUnison's internal directory to search for users to match | `ou=users,ou=shadow,o=Tremolo` |
| (Optional) Namespace | If the binding is a `RoleBinding`, the namespace to find it in | mynamespace |

```yaml
approvers:
- scope: custom
  constraint: rbacaz!k8s!ClusterRole!admin!mail!ou=users,ou=shadow,o=Tremolo!$namespace$
```

## UserHasSessionAz

Authorizes a user based on having an active session stored in a Kubernetes cluster.  Used to authorize access to web applications used for managing a cluster.  For instance, logging a user out of the dashboard when the user's kubectl sessions are destroyed.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: userhassession
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.az.UserHasSessionAz
  params:
    # the Target that points to the Kubernetes cluster
    target: k8s
    # The namespace to search
    namespace: openunison
  secretParams: []
```

### Example azRule

There are no additional parameters on the constraint.

```yaml
azRules:
- scope: custom
  constraint: userhassession
```

## AlwaysFail

Will always fail authorization.  Useful when escalating past the point of an acceptable limit to close out a request.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: alwaysfail
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.az.AlwaysFail
  params: []
  secretParams: []
```

### Example azRule

There's no additional configuration besides the constraint name.

```yaml
azRules:
- scope: custom
  constraint: alwaysfail
```

## JavaScriptAz

Use JavaScript to create a custom authorization rule.