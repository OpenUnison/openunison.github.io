# Configuring OpenUnison Applications

The OpenUnison you deploy with your kubernetes implementation can be customized for several kinds of identity services.  For instance, if you're running a workflow in GitLab or GitHub, you can create a service in OpenUnison that allows you to get a kubectl configuration from the workflow using the token from your workflow!  There's an incredible amount of power in OpenUnison's capabilities to tightly integrate a strong identity into your environment.  Whether you're trying to integrate user facing applications or need additional services.  OpenUnison has the capability.  When looking at building identity services on OpenUnison, the first place to look is OpenUnison's `Application` object.

## Application Configuration

### Applications

`Application` objects are the core for building identity services and integrations with OpenUnison.  There are three kinds of `Application` objects:

* ***Reverse Proxy*** - OpenUnison forwards all requests to this `Application` to a downstream service.  For instance, if you're [deploying Prometheus](/documentation/custom-sso/#using-a-reverse-proxy-application) and AlertManager into your cluster, you'll use this method.
* ***Identity Provider*** - An `Application` can be configured to be either an OpenID Connect or SAML2 identity provider.  In this mode you have the ability to customize how a user's identity is presented to the relying party.  For instance, if you were to [deploy an integration with ArgoCD with a customized groups mapping](/documentation/custom-sso/#creating-an-openid-connect-identity-provider) you would use this method.
* ***Local Service*** - You can implement custom web services in an `Application` that aren't proxied back to another webapp or API.  This is really useful for implementing Security Token Services (STS) like we described above.

An `Application` is built on a set of `url`s.  Each `url` specifies:

* ***Hosts -*** Each URL can be associated with multiple host names, or all host names with `*`
* ***Authentication Chain -*** How to authenticate this URL
* ***Filters -*** A list of plugins that can manipulate both request and response headers.  This is useful for injecting identity information or adding data to responses.  [OpenUnison is bundled with several pre-built filters](/documentation/filters).
* ***Authorization Rules -*** A list of rules that defines who is able to access this URL
* ***Results -*** A set of authorization results, such as setting a cookie or adding a header
* ***Cookie Configuration -*** If a web application requires a session, this is how you configure how cookies are encrypted.  This allows you to isolate applications with separate keys, even if the cookie space covers the same scope.

If you're `Application` is meant to be an identity provider (IdP), the URL will have a special configuration as well.

For detailed configuration information for each configuration option, see the [Application CRD](/documentation/openunison-crds/#application) documentation.

With a general idea of how to configure an application in OpenUnison, the next concept we'll work on is how authentication is built in OpenUnison.

### Authentication

Every URL in an `Application` has exactly one `authChain`.  Authentication chains are a sequence of authentication mechanisms that have a level associated with it.  This gives you the ability to string together different kinds of authentication and provides for some limited branching.  For instance, you can create a chain that accepts an OIDC token, then just-in-time provisions an account into a database.  By assigning an authentication level to each chain, you can force re-authentication.  As an example, you have an application that only needs a single factor, except for the admin system.  Assigning an authentication chain with a higher level for the admin system will force a re-authentication.

The following mechanisms are pre-configure with OpenUnison:

| Mechanism | Description |
| --------- | ----------- |
| login-form | Login with a username and password |
| jit | Just-in-time provisioning |
| genoidctoken | Generates an OIDC token from an oidc identity provider and puts it in the user's session |
| login-service | Allows a user to choose from multiple authentication chains |
| oidc | Authenticate using a remote OIDC identity provider |
| saml2 | Authenticate using a remote SAML2 identity provider |
| oauth2jwt | Authenticate with a JWT |
| oauth2k8s | Authenticate with a Kubernetes `ServiceAccount` and validate it using a `TokenReview` |
| map | Maps attributes in an authenticated identity |
| github | Authenticates to GitHub |
| az | Allows for authorization rules to be executed during the authentication process |
| js | Write custom authentication mechanisms in JavaScript | 
| include | Includes another authentication chain in the current chain |

Additional [Authentication Mechanisms](/documentation/authentication-mechanisms) can be configured by creating the appropriate [Kubernetes AuthenticationMechanism CRD](/documentation/openunison-crds/#authenticationmechanism).

Once authentication has been configured for an `Application`, the next question to answer is how to authorize each request.

### Authorization

OpenUnison's built in authorization system is common across multiple processes.  Each authorization is made up of a series of authorization rules.  Authorization succeeds if ***ANY*** of the rules pass.  There are four authorization options for any given rule:

| Authorization Mode | Description | Constraint Example |
| ------------------ | ----------- | ------------------ |
| group | The DN of a group the suer must be a member of in OpenUnison's virtual directory | cn=group-name,ou-orgs,dc=domain,dc=com |
| dynamicGroup | The DN of an LDAP dynamic group the suer must be a member of in OpenUnison's virtual directory | cn=ldap-dynamic-group,ou=groups,dc=domain,dc=com |
| dn | The DN of an area of the virtual directory where all users under that DN are authorized | ou=users,dc=domain,dc=com |
| custom | The name of a CustomAuthorization. Can be followed by a ! with parameters. For instance SomeCustomAZ!param1!param2 will call the custom authorization SomeCustomAZ with the parameters param1 and param2 | SomeCustomAZ!param1!param2 |
| filter | An LDAP filter that will be applied to a logged in user | (objectClass=*) |

Since authorization rules can incur a significant performance impact, each `Application` has an option called `azTimeoutMillis` that caches authorization decisions for a user for a very limited time.  Usually a few seconds.

OpenUnison comes with several [Custom Authorizations](/documentation/authentication-mechanisms) that you can leverage in your configurations.  You can also write your own using JavaScript.

Once your authorizations are configured, the last choice is to decide if there will be any actions performed as a result of an authentication or an authorization decision.

### Result Groups

Every URL can trigger an event as the result of one of four possible events:

| Event | Description |
| ----- | ----------- |
| auSuccess | Successful Authentication, happens once per session |
| auFail | Failed authentication, happens when an authentication chain fails |
| azSuccess | When a URL access is authorized, triggers on each request |
| azFail | When a URL access is denied, triggers on each request |

Each event can be configured with a [ResultGroup](/documentation/openunison-crds/#resultgroup), which can in turn be configured with multiple results.  Results can be a cookie, a request header, or a redirect.  The value of a result can come from a static value, and attribute value, or from a [custom result](/documentation/custom-results)

## Deployment and Customization

Once your `Application` object is deployed, you can immediately start testing.  OpenUnison is an operator, so it will automatically load the objects you create and process them.  You may want to consider building your objects into a Helm chart.  This makes it easy to deploy for both a gitops scenario or using the ouctl utility.

In addition to any values configured into your objects as parameters, you can also include some custom variables by using `#[variable_name]` in your configuration.  For instance, `#[OU_HOST]` will include the value for OpenUnison's host (`network.openunison_host` in your values.yaml).  Any of the entries in the `orchestra` `OpenUnison` object's `non_secret_data` can be included this way.  You can get a list of non secret entries with the command:

```sh
 k get openunison orchestra -n openunison -o json | jq -r '.spec.non_secret_data'
[
  {
    "name": "K8S_URL",
    "value": "https://192.168.2.130:6443"
  },
  {
    "name": "SESSION_INACTIVITY_TIMEOUT_SECONDS",
    "value": "900"
  },
  {
    "name": "K8S_DASHBOARD_NAMESPACE",
    "value": "kubernetes-dashboard"
  },
  {
    "name": "K8S_DASHBOARD_SERVICE",
    "value": "kubernetes-dashboard"
  },
  {
    "name": "K8S_CLUSTER_NAME",
    "value": "openunison-cp"
  },
  {
    "name": "OPENUNISON_PROVISIONING_ENABLED",
    "value": "false"
  },
  {
    "name": "K8S_IMPERSONATION",
    "value": "false"
  },
  {
    "name": "PROMETHEUS_SERVICE_ACCOUNT",
    "value": "system:serviceaccount:monitoring:prometheus-k8s"
  },
  {
    "name": "OU_SVC_NAME",
    "value": "openunison-orchestra.openunison.svc"
  },
  {
    "name": "K8S_TOKEN_TYPE",
    "value": "legacy"
  },
  {
    "name": "K8S_DB_SSO",
    "value": "oidc"
  },
  {
    "name": "PROMETHEUS_SERVICE_ACCOUNT",
    "value": "system:serviceaccount:monitoring:prometheus-k8s"
  },
  {
    "name": "SHOW_PORTAL_ORGS",
    "value": "false"
  },
  {
    "name": "K8S_OPENUNISON_NS",
    "value": "openunison"
  }
]
```