# Application URL Filters

Unison provides the capability to make changes to each request. For an identity provider this typically means adding additional attributes to an assertion. For a reverse proxy, this generally means adding headers or executing workflows based on the user's choices. The below filters come standard in OpenUnison.

## Group2Attribute

This filter allows for an attribute to be added to an assertion if the user is a member of a particular group in your directory. This could be useful when providing service providers entitlement information. This filter can be added multiple times and if the user is a member of the specified group AND the attribute already exists the specified value is added to the attribute, it does not replace it.

```yaml
- className: com.tremolosecurity.prelude.filters.Group2Attribute
  params:
    # The name of the attribute to create if the user is a member of this group
    attributeName: roles
    # The value to be added or set if the user is a member of the specified group
    attributeValue: admin
    # The full LDAP DN of the group being checked. This DN must be the mapped DN from inside of Unison. The ??? button next to this option may be used to search for the group based on it?s CN.
    groupDN: cn=staticgroup,ou=internal,ou=GenericLDAP,o=Tremolo
  secretParams: []
```

## DNBase2Attribute

This filter allows for an attribute to be added to an assertion if the user's DN in the virtual directory is a child of the specified DN. This could be useful when providing service providers entitlement information. This filter can be added multiple times and if the user is a member of the specified DN AND the attribute already exists the specified value is added to the attribute, it does not replace it.

```yaml
- className: com.tremolosecurity.prelude.filters.DNBase2Attribute
  params:
    # The name of the attribute to create if the user is a member of this group
    attributeName: roles
    # The value to be added or set if the user is a member of the specified group
    attributeValue: admin
    # The full LDAP DN of the base being checked. This DN must be the mapped DN from inside of Unison.
    dn: cn=Users,ou=MyEnterprise,O=Tremolo
  secretParams: []
```

## LoginTest

This filter will echo the attributes of the currently logged in user. It's a convenient way to test the login process without having to have an application to proxy or an identity provider configured. Configure this filter on a URL and that URL will use this filter to provide content back to the web browser. No filters configured after this filter are executed.

```yaml
- className: com.tremolosecurity.prelude.filters.LoginTest
  params:
    # The URI of the JSP that will echo the information
    jspURI: "/auth/forms/loginTest.jsp"
    # The path of the logout URI
    logoutURI: "/logout"
  secretParams: []
```

## XForward

The X-Forward headers (X-Forwarded-For, X-Forwarded-Host and X-Forwarded-Proto), are a defacto standard for supplying down-stream servers with information as a reverse proxy would see it. This filter will create these attributes for use as headers.

```yaml
- className: com.tremolosecurity.proxy.filters.XForward
  params:
    # Determine if plain HTTP Headers or Secure Headers should be used. If checked, standard headers are created, if not checked then attributes are created with the same names that can be added as Secure Headers to the LastMile filter.
    createHeaders: "false"
  secretParams: []
```

## CreateAWSRoleAttribute

This filter is designed to be added to an identity provider for the AWS console to create an attribute that is acceptable to AWS out of human readable role names.

```yaml
- className: com.tremolosecurity.proxy.filters.custom.CreateAWSRoleAttribute
  params:
    # The name of an attribute on the user object that contains all of the names of roles to be included in the assertion. NOTE: each role should exist in IAM or the authentication will fail.
    sourceAttribute: roleNames
    # The AWS account number that should be included in the role mapping.
    idpName: "1234567"
    # The name of the identity provider in your AWS IAM configuration to bind to.
    accountNumber: MyIdp
  secretParams: []
```

## StopProcessing

The `Stop Processing` filter will stop all processing, not executing any filters configured after it or sending a request to the proxied server. This filter has no configuration options.

```yaml
- className: com.tremolosecurity.prelude.filters.StopProcessing
  params: {}
  secretParams: []
```

## ExecuteWorkflow

This insert will execute a workflow using the user loaded by the authentication process.

```yaml
- className: com.tremolosecurity.proxy.filters.ExecuteWorkflow
  params:
    # The name of the workflow to execute
    workflowName: jitdbwf
    # The name of the attribute on the user object used to identify the user
    uidAttributeName: uid
  secretParams: []
```

## UserToJSON

This filter will create a JSON object based on the user?s attributes. The class com.tremolosecurity.proxy.auth.AuthInfo is serialized into JSON into the attribute UserJSON. This attribute can then be used as a header in a result or a LastMile attribute.

```yaml
- className: com.tremolosecurity.proxy.filters.UserToJSON
  params:
    # If set to “true” the request is continued to be proxied. If set to false, the request completes
    doProxy: "false"
  secretParams: []
```

## AzFilter

If an application is configured to not use a session, the user?s context may be set in a filter but the authorization process will not be executed. This filter will execute authorization rules and execute result groups in this scenario. There are no configuration options for this filter, as it uses the rules configured on the application.

```yaml
- className: com.tremolosecurity.proxy.filters.AzFilter
  params: {}
  secretParams: []
```

## RemoteBasic

This filter will authenticate users by executing a basic authentication request against a remote server using the Authorization header inbound from the browser. The filter will then set the user?s context. Its designed to work with an application?s session disabled.

```yaml
- className: com.tremolosecurity.proxy.filters.RemoteBasic
  params:
    # The name of the realm on the remote web server the authentication is against
    realmName: Tremolo Basic Realm
    # The URL to use to test the authentication
    url: http://localhost.localdomain:9090/echo/echo4
  secretParams: []
```

## LastMile

The last mile security filter generates the token utilized to validate the request by the Last Mile system deployed on the application. This filter can be configured to add attributes, roles and other information to the last mile token. It also supplies the configuration needed for the application?s last mile configuration.

```yaml
- className: com.tremolosecurity.proxy.filters.LastMile
  params:
    # The key used to encrypt the last mile header
    encKeyAlias: lastmile-userreg
    # The number of milliseconds that the last mile token is valid
    timeScew: "1000"
    # The name of the header
    headerName: tremoloHeader
    # Static text to put in front of the token, ie for OAuth2 bearer tokens
    headerPrefix: ""
    # Each attrib lists a mapping from a user attribute to the attribute in the last mile token
    attribs:
      - uid=displayName
      - uid=mail
  secretParams: []
```

## CheckADShadowAccounts

When integrating with an AD environment that is used for both shadow accounts and real accounts this filter is used to transition from real account in an external forest to a shadow account.

```yaml
- className: com.tremolosecurity.proxy.filters.CheckADShadowAccounts
  params:
    # The UPN suffix for AD forest storing shadow accounts
    nonShadowSuffix: shadows.ad.local
    # The attribute that’s used for the source of the shadow account’s UPN
    upnAttributeName: mail
    # Attribute to store a flag value if the account is to be a shadow account
    flagAttributeName: description
    # Flag value if the user will become a shadow account
    flagAttributeValue: shadow
  secretParams: []
```

## BasicAuth

This filter is used in conjunction with an application with a disabled session. It will perform a basic authentication against the internal virtual directory.

```yaml
- className: com.tremolosecurity.proxy.filters.BasicAuth
  params:
    # The name of the realm to be presented to the browser
    realmName: Tremolo Basic Realm
    # The name of the attribute to use to lookup the user
    uidAttr: uid
  secretParams: []
```

## AnonAz

This filter is used in conjunction with an application with a disabled session. It will create an AuthInfo object based on an anonymous user.

```yaml
- className: com.tremolosecurity.proxy.filters.AnonAz
  params:
    # RDN of the anonymous user
    userName: uid=Anonymous
  secretParams: []
```

## HideCookie

This filter will remove all cookies set by the proxied applications prior to being sent to the client. The cookies are stored in an internal cookie jar in the user?s session. There are no configuration options.

```yaml
- className: com.tremolosecurity.proxy.filters.HideCookie
  params: {}
  secretParams: []
```

## LastMileJSON

Used in conjunction with the OAuth2 Last Mile Bearer Token authentication scheme, this filter will create an HTML page with an OAuth 2 access token inside of a div called "json".

```yaml
- className: com.tremolosecurity.proxy.filters.LastMileJSON
  params:
    # Last Mile encryption key
    encKeyAlias: sessionkey
    # The number of seconds the returned token is valid
    secondsScew: "300"
    # The number of seconds to adjust for if clocks are not synced
    secondsToLive: "100"
  secretParams: []
```

## PreAuthFilter

Some applications do not work well with a reverse proxy and require an explicit "login" step. In these scenarios the Pre-Authentication filter can be used to create a session prior to the first time a user accesses the website. This filter does a Last Mile login to the url and can optionally generate a SAML2 assertion and perform an IdP initiated SSO. Once the login is complete the cookies from the request are added to the user's cookie jar.

```yaml
- className: com.tremolosecurity.proxy.filters.PreAuthFilter
  params:
    # Determines if a SAML assertion should be generated and posted to the Pre-Auth URL.
    # If checked, an Identity Provider MUST be created to supply the configuration information to generate the assertion.
    postSAML: "true"
    # The fully qualified domain name (FQDN) and URI of the URL in Unison configured with a LastMile filter, only needed if postSAML is true
    url: http://mysite.company.com/myapp/login
    # The name of the identity provider that has the configuration information for generating the assertion
    idpName: saml2
    # The host name that should be in the issuer
    issuerHost: localhost.localdomain
    # If the issuer is on a non-standard port, it can be specified here. This field is optional
    issuerPort: "6060"
    # If the issuer should be https (checked) or http (not checked)
    issuerSSL: "false"
    # If the application expects a RelayState
    relayState: "/echo/echo"
  secretParams: []
```

## Groups2Attribute

This filter will create an attribute with the names of groups the user is a member of. An optional regular expression can be used to specify only a certain number of groups.

```yaml
- className: com.tremolosecurity.prelude.filters.Groups2Attribute
  params:
    # The base in the virtual directory to begin searching for groups.
    base: o=Tremolo
    # Name of the attribute to create
    attrName: roles
    # A regular expression to filter out group memberships
    pattern: groups-(.*)
    # If a pattern is specified, the group from that pattern to add to the attribute
    groupNum: "1"
  secretParams: []
```

## CookieFilter

This filter will stop all cookies, except those configured, from being sent to downstream applications. This can be used to stop third party cookies, attempts to spoof or cookie collisions.

```yaml
- className: com.tremolosecurity.proxy.filters.CookieFilter
  params:
    # List of cookie names (or regular expressions) to filter
    ignore: cookie2
    # If checked, the values to filter are treated as Java regular expressions.
    # (http://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)
    supportRegex: "false"
  secretParams: []
```

## SetNoCacheHeaders

This filter will set response headers that will tell browsers not to cache the content.

```yaml
- className: com.tremolosecurity.proxy.filters.SetNoCacheHeaders
  params: {}
  secretParams: []
```

## AddHttpsToRedirect

This filter will replace http:// with https:// in the Location response header.  Useful when overriding the Host header is disabled but the proxy transitions from https to http.

```yaml
- className: com.tremolosecurity.proxy.filters.AddHttpsToRedirect
  params: {}
  secretParams: []
```

## CheckSession

This filter will check to see if the user's session is active, and if so how much longer the user's session is.  This filter should be configured on its own application, seperate from the application being tested
and with its own sesison cookie.  This way the user's session is not reloaded when checked.

```yaml
- className: com.tremolosecurity.proxy.filters.CheckSession
  params:
    # The name of the application whose session cookie data to check
    applicationName: ScaleJS
  secretParams: []
```

## RemovePrefix

The RemovePrefix filter will remove the beginning of the URI and put it into an attribute that can be included in the proxyTo.

```yaml
- className: com.tremolosecurity.proxy.filters.RemovePrefix
  params:
    # The part of the URI to trim off
    prefix: /scale
    # The attribute to set with the new URI
    attributeName: trimmedURI
  secretParams: []
```

## RetrieveIdToken

Kubernetes does not retrieve your id_token using the authorization flow, it assumes that the bearer token provided is an id_token with a short expiration date.  In this filter provides a simple service that will generate the
id_token for an authenticated session based on the session's refresh_token.  Configure this filter on an anonymous URL and pass a single parameter, refresh_token, to get a proper id_token:

```bash
$ kubectl --token=$(curl https://openunison.server/service/idtoken?refresh_token=....) get nodes
```

***NOTE:*** this will not generate a new token once the current one has expired.  That requires going to ScaleJS and refreshing the token screen to get a new id_token.

```yaml
- className: com.tremolosecurity.proxy.filters.RetrieveIdToken
  params:
    # The name of the identity provider to pull the id_token from
    idpName: oidc
    # The name of the trust to pull the id_token from
    trustName: kubernetes
  secretParams: []
```

## CallWorkflow

Allows for a remote OpenUnison to call a workflow by calling a PUT of a workflow request.  Recommended to use with an OAuth2 authentication chain.

```yaml
- className: com.tremolosecurity.proxy.filters.CallWorkflow
  params:
    # Determines which workflows can be called, can be listed multiple times
    allowedWorkflow: updateLockout
  secretParams: []
```

## HeaderFilter

Removes an inbonud header from a request.  Header names are NOT case sensistive

```yaml
- className: com.tremolosecurity.proxy.filters.HeaderFilter
  params:
    # May be listed multiple times for multiple headers
    headerName: ToBeRemoved
  secretParams: []
```

## LdapOnJson

The LdapOnJson filter will provide LDAP like functionality via a RESTful JSON interface.  The filter provides search and bind capabilities only.  Unlike LDAP, searches and binds are NOT tied together.  This is a convenience for accessing OpenUnison's internal virtual directory without using the LDAP protocol.  It is not a replacement for LDAP.

```yaml
- className: com.tremolosecurity.proxy.filters.LdapOnJson
  params: {}
  secretParams: []
```

### Swagger Definition

```yaml
---
swagger: '2.0'
info:
  version: "1.0.17"
  title: OpenUnison Ldap2Json

definitions:
  Attributes:
    type: object
    additionalProperties:
      type: array
      items:
        type: string
    description: Map of attribute name --> multiple values

  LdapJsonBindRequest:
    type: object
    properties:
      password:
        type: string
        description: Secret for authentication
    description: in java package com.tremolosecurity.ldapJson

  LdapJsonError:
    type: object
    description: in java package com.tremolosecurity.ldapJson
    properties:
      responseCode:
        type: integer
        description: LDAP Response code
      errorMessage:
        type: string
        description: response message

  LdapJsonEntry:
    type: object
    description: in java package com.tremolosecurity.ldapJson
    properties:
      dn:
        type: string
        description: The distinguished name of the entry
      attrs:
        $ref: '#/definitions/Attributes'

paths:
  /{base_dn}/{search_scope}:
    get:
      description: performs an LDAP Search
      parameters:
        -
          name: base_dn
          in: path
          description: Search base
          required: true
          type: string
        -
          name: search_scope
          in: path
          description: One of base, sub, one
          required: true
          type: string
        -
          name: filter
          in: query
          description: LDAP Filter
          required: true
          type: string
        -
          name: attributes
          in: query
          description: each value is an attribute to be returned
          type: string
          minimum: 0
      responses:
        200:
          description: list of entries
          schema:
            type: array
            items:
              $ref: '#/definitions/LdapJsonEntry'
        500:
          description: error
          schema:
            $ref: '#/definitions/LdapJsonError'

  /{user_dn}:
    post:
      description: performs an LDAP Bind to verify the user's credentials
      parameters:
        -
          name: user_dn
          in: path
          description: The distinguished name of the user to check
          type: string
          required: true
        -
          name: body
          in: body
          schema:
            $ref: '#/definitions/LdapJsonBindRequest'
          description: Bind request
      responses:
        200:
          description: valid credentials
          schema:
            $ref: '#/definitions/LdapJsonError'
        500:
          description: error
          schema:
            $ref: '#/definitions/LdapJsonError'
```

## MetricsFilter

This filter will generate Prometheus metrics for OpenUnison.  It includes metrics for:

* StandardExports
* MemoryPoolsExports
* GarbageCollectorExports
* ThreadExports
* ClassLoadingExports
* VersionInfoExports

Additionally the `active_sessions` metric is provided that has the number of open sessions in OpenUnison.  No security is propvided for this filter, instead use OpenUnison's built in security features to add authentication.


```yaml
- className: com.tremolosecurity.prometheus.filter.MetricsFilter
  params:
    # Optional - implementation of com.tremolosecurity.prometheus.sdk.LocalMetrics to add custom metrics
    localMetricsClassName: com.tremolosecurity.MyMetrics
  secretParams: []
```

## JMSPull

The JMSPull listener is designed to open access to Prometheus metrics that are not directly accessible from Prometheus.  Instead of a direct connection to metrics, this filter will proxy the request to a remove OpenUnison via a common message queue and wait for a response via a separate queue.  This filter should be paired with an OpenUnison instance that has the `com.tremolosecurity.prometheus.aggregate.PullListener` deployed.

```yaml
- className: com.tremolosecurity.prometheus.aggregate.JMSPull
  params:
    # Request queue
    requeustQueueName: prometheus-request.fifo
    # Response queue
    responseQueueName: prometheus-response.fifo
    # Job name
    job: remote
    # Instance name
    instance: prometheus
    # Time in milliseconds to wait for a response on the response queue
    maxWaitTime: "60000"
    # Implementation of com.tremolosecurity.prometheus.sdk.AdditionalMetrics for additional metrics
    additionalMetricsClassName: com.domain.idm.ExtraChecks
    # Additional options
    loginTestURL: "#[PROMETHEUS_LOGIN_URL]"
    loginTestUserName: "#[PROMETHEUS_LOGIN_USERNAME]"
    loginTestPassword: "#[PROMETHEUS_LOGIN_PASSWORD]"
    internal: "false"
  secretParams: []
```

## K8sInjectImpersonation

Injects impersonation headers into each request designed to be used by the Kubernetes API server.  In order for these headers to be accepted the must be accompanied by a service account that has impersonation rights.

```yaml
- className: com.tremolosecurity.proxy.filters.K8sInjectImpersonation
  params:
    # The name of the OpenShift (kubernetes) target
    targetName: k8s
    # The username attribute from the logged-in user's account
    userNameAttribute: sub
    # If true, groups are pulled directly from an LDAP search
    useLdapGroups: "false"
    # If useLdapGroups is false, the attribute to use to load groups
    groupAttribute: groups
  secretParams: []
```