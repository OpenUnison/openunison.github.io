# OpenUnison Kubernetes CRD Documentation

## Trust

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | trust |
| **Plural** | trusts |

### Version: v1

Represents a dynamic trust for an identity provider

| Name | Type | Description |
|------|------|-------------|
| label | string | A descriptive label for the trust |
| clientId | string | A unique id for the trust shared with the trusted client |
| clientSecret | object | A reference to the `Secret` used to store the secret shared between the identity provider and the client [Details](#root-clientsecret) |
| publicEndpoint | boolean | If true, `clientSecret` is ignored |
| redirectURI | array | List of allowed URLs that should be trusted to redirect to after successful authentication |
| codeLastMileKeyName | string | The name of the key in the OpenUnison keystore that is used to encryp the code portion of a transaction |
| authChainName | string | The name of an `AuthenticationChain` that is used to authenticate this trust |
| codeTokenSkewMilis | integer | The time, in milliseconds, to allow for clock skew when checking if a token has expired |
| accessTokenTimeToLive | integer | The time, in miliseconds, that a trust's token should be valid |
| accessTokenSkewMillis | integer | The time, in milliseconds, to allow for clock skew when checking if a token has expired |
| signedUserInfo | boolean | If true, the `user_info` endpoint's response is a JWT, not simple JSON |
| verifyRedirect | boolean | If true, verifies if the redirect is accurate.  Should only be false in development environments. |


###<a name="root-clientsecret"></a> clientSecret

A reference to the `Secret` used to store the secret shared between the identity provider and the client

| Name | Type | Description |
|------|------|-------------|
| secretName | string | The name of the `Secret` |
| keyName | string | The key storing the data in the `Secret` |


## PortalUrl

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | portalurl |
| **Plural** | portalurls |

### Version: v1

A PortalURL represents a "badge" in the OpenUnison portal

| Name | Type | Description |
|------|------|-------------|
| label | string | A descriptive label for the badge |
| url | string | The URL for the application a user will be sent to when clicking on the badge |
| org | string | The `Orginaization.uuid` the url is a part of |
| icon | string | A base64 encoded PNG that is 240px high x 210px wide |
| azRules | array | List of rules used to authorize access.  Authorization succeeds if **ANY** of these rules pass [Details](#root-azrules) |


###<a name="root-azrules"></a> azRules

List of rules used to authorize access.  Authorization succeeds if **ANY** of these rules pass

| Name | Type | Description |
|------|------|-------------|
| scope | string | One of:<br>- `group` - The DN of a group the suer must be a member of in OpenUnison's virtual directory<br>- `dynamicGroup` - The DN of an LDAP dynamic group the suer must be a member of in OpenUnison's virtual directory<br>- `dn` - The DN of an area of the virtual directory where all users under that DN are authorized<br>- `custom` - The name of a `CustomAuthorization`.  Can be followed by a `!` with parameters.  For instance `SomeCustomAZ!param1!param2` will call the custom authorization `SomeCustomAZ` with the parameters `param1` and `param2`<br>- `filter` - An LDAP filter that will be applied to a logged in user |
| constraint | string | How to enforce this rule based on the `scope` |


## Org

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | org |
| **Plural** | orgs |

### Version: v1

{}

| Name | Type | Description |
|------|------|-------------|
| label | string | What people will see in the OpenUnison portal |
| description | string | Descriptive text |
| uuid | string | A unique identifier used to associate resources with this `Organization` |
| parent | string | The parent `Organization`'s uuid in the tree |
| showInPortal | boolean | If true, shows this orgnaization on the portal screen in OpenUnison |
| showInRequestAccess | boolean | If true, shows the `Orgianization` in the `Request Access` screen of OpenUnison |
| showInReports | boolean | If true, show in the  `Reports` section in OpenUnison. |
| azRules | array | List of rules used to authorize access.  Authorization succeeds if **ANY** of these rules pass [Details](#root-azrules) |


###<a name="root-azrules"></a> azRules

List of rules used to authorize access.  Authorization succeeds if **ANY** of these rules pass

| Name | Type | Description |
|------|------|-------------|
| scope | string | One of:<br>- `group` - The DN of a group the suer must be a member of in OpenUnison's virtual directory<br>- `dynamicGroup` - The DN of an LDAP dynamic group the suer must be a member of in OpenUnison's virtual directory<br>- `dn` - The DN of an area of the virtual directory where all users under that DN are authorized<br>- `custom` - The name of a `CustomAuthorization`.  Can be followed by a `!` with parameters.  For instance `SomeCustomAZ!param1!param2` will call the custom authorization `SomeCustomAZ` with the parameters `param1` and `param2`<br>- `filter` - An LDAP filter that will be applied to a logged in user |
| constraint | string | How to enforce this rule based on the `scope` |


## Target

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | target |
| **Plural** | targets |

### Version: v1

{}

| Name | Type | Description |
|------|------|-------------|
| className | string | Implementation of `com.tremolosecurity.provisioning.core.UserStoreProvider` |
| params | array | Set of configuration parameters [Details](#root-params) |
| secretParams | array |  [Details](#root-secretparams) |
| targetAttributes | array | Attributes that are managed by this target [Details](#root-targetattributes) |


###<a name="root-params"></a> params

Set of configuration parameters

| Name | Type | Description |
|------|------|-------------|
| name | string | Configuration parameter name, may be listed multiple times |
| value | string | Value |


###<a name="root-secretparams"></a> secretParams



| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the configuration parameter |
| secretName | string | The name of the Kubernetes `Secret` |
| secretKey | string | The name of the key in the `Secret` to pull the data from |


###<a name="root-targetattributes"></a> targetAttributes

Attributes that are managed by this target

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the attribute in the target |
| source | string | Depends on `sourceType`:<br> - `static` - The specific value of the attribute<br> - `user` - The name of the attribute from the user to use<br> - `custom` - Implementation of `com.tremolosecurity.provisioning.mapping.CustomMapping`, parameters can be supplied after a `|` that's comma delimited.  For instance, `com.tremolosecurity.mapping.JavaScriptMapping|k8s,openunison,argocd-groups` Uses the `JavaScriptMapping`, passing the parameters `k8s`, `openunison`, and `argocd-groups` as parameters |
| sourceType | string | Where the attribute data will come from |
| targetType | string | The data type for the target attribute |


## Workflow

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | workflow |
| **Plural** | workflows |

### Version: v2

Holder for a workflow object

| Name | Type | Description |
|------|------|-------------|
| label | string | Descriptive name |
| description | string | Descriptive text |
| inList | boolean | If true, includes this workflow as requestable on the `Request Access` screen |
| orgId | string | The value of an `Organization.uuid` that contains this workflow |
| dynamicConfiguration | object | Used to apply a workflow to several potential objects.  Will instanciate one instance of this workflow for each object.  For instance creating one instance of a workflow for each namespace in a cluster. [Details](#root-dynamicconfiguration) |
| tasks | string | See https://openunison.github.io/UPDATEME |


###<a name="root-dynamicconfiguration"></a> dynamicConfiguration

Used to apply a workflow to several potential objects.  Will instanciate one instance of this workflow for each object.  For instance creating one instance of a workflow for each namespace in a cluster.

| Name | Type | Description |
|------|------|-------------|
| dynamic | boolean | Determines if the workflow should be loaded based on a dynamic configuration |
| className | string | Implementation of `com.tremolosecurity.provisioning.util.DynamicWorkflow` |
| params | array | List of parameters to pass to the `DynamicWorkflow` implementation. [Details](#root-dynamicconfiguration--params) |
| filterAnnotations | array | List of parameters to pass to the `DynamicWorkflow` implementation. [Details](#root-dynamicconfiguration--filterannotations) |


####<a name="root-dynamicconfiguration--params"></a> params

List of parameters to pass to the `DynamicWorkflow` implementation.

| Name | Type | Description |
|------|------|-------------|
| name | string | Can be repeated |
| value | string |  |


####<a name="root-dynamicconfiguration--filterannotations"></a> filterAnnotations

List of parameters to pass to the `DynamicWorkflow` implementation.

| Name | Type | Description |
|------|------|-------------|
| name | string | name of the annotation filter |
| requestObjectName | string | name of the request object to get the annotation value from |


## Report

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | report |
| **Plural** | reports |

### Version: v1

{}

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the report |
| description | string | Descriptive text for the report |
| groupBy | string | Report field that data should be grouped by |
| groupings | boolean | If true, the report is broken into sections based on the field identified by `groupBy` |
| orgId | string | The `Orgnanization.id` that this report belongs in |
| parameters | object | Determines which parameters should be included [Details](#root-parameters) |
| sql | string | The SQL used to generate the report. |
| headerFields | array | Data to be displayed as a header |
| dataFields | array | List of fields to show in the report |


###<a name="root-parameters"></a> parameters

Determines which parameters should be included

| Name | Type | Description |
|------|------|-------------|
| beginDate | boolean |  |
| endDate | boolean |  |
| userKey | boolean |  |
| currentUser | boolean |  |


## OUJob

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | oujob |
| **Plural** | oujobs |

### Version: v1

{}

| Name | Type | Description |
|------|------|-------------|
| cronSchedule | object | The cron configuration, using standard cron rules. [Details](#root-cronschedule) |
| className | string | Implementaiton of `com.tremolosecurity.provisioning.scheduler.UnisonJob` |
| group | string | A key to group jobs by, arbitrary |
| params | array | List of configuration options.  individual options may be listed multiple times [Details](#root-params) |
| secretParams | array | Secret data that should be loaded directly from Kubernetes Secrets [Details](#root-secretparams) |


###<a name="root-cronschedule"></a> cronSchedule

The cron configuration, using standard cron rules.

| Name | Type | Description |
|------|------|-------------|
| seconds | string |  |
| minutes | string |  |
| hours | string |  |
| dayOfMonth | string |  |
| month | string |  |
| dayOfWeek | string |  |
| year | string |  |


###<a name="root-params"></a> params

List of configuration options.  individual options may be listed multiple times

| Name | Type | Description |
|------|------|-------------|
| name | string |  |
| value | string |  |


###<a name="root-secretparams"></a> secretParams

Secret data that should be loaded directly from Kubernetes Secrets

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the configuration option for the job |
| secretName | string | The name of the `Secret` to get the value from |
| secretKey | string | The key in the `data` section of the `Secret` to get the data from |


## MessageListener

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | messagelistener |
| **Plural** | messagelisteners |

### Version: v1

Reveives inbound messages for asynchronous processing

| Name | Type | Description |
|------|------|-------------|
| className | string | Implementation of `com.tremolosecurity.provisioning.core.UnisonMessageListener` |
| params | array | List of configuration options, each option can be listed multiple times [Details](#root-params) |
| secretParams | array | Secret data to be loaded directly from Kubernetes `Secret` objects [Details](#root-secretparams) |


###<a name="root-params"></a> params

List of configuration options, each option can be listed multiple times

| Name | Type | Description |
|------|------|-------------|
| name | string |  |
| value | string |  |


###<a name="root-secretparams"></a> secretParams

Secret data to be loaded directly from Kubernetes `Secret` objects

| Name | Type | Description |
|------|------|-------------|
| name | string | The configuration option to set |
| secretName | string | The `Secret` to pull data from |
| secretKey | string | The key in the `data` section of the `Secret` to pull the value from |


## ResultGroup

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | resultgroup |
| **Plural** | resultgroups |

### Version: v1

A group of actions to take as a result of an authentication or authorization event

***The spec is an array with each item below being a property of each array item***

| Name | Type | Description |
|------|------|-------------|
| resultType | string | A result of an authenticaiton or authorization event:<br>- `header` - A request header injected into an HTTP request, result of an authoriztion event<br>- `cookie` - A Cookie being added to the response, result of an authentication event<br>- `redirect` - A 302 sent back to the client, can be the result of either |
| source | string | The source of the value of a result |
| value | string | Depends on `source`:<br>- `static` - The value to set<br>- `user` - The name of the attribute from the logged in user to user<br>- `custom` - Implementation of `com.tremolosecurity.proxy.results.CustomResult` |


## CustomAuthorization

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | customaz |
| **Plural** | customazs |

### Version: v1

A custom authorization is an implementation of `com.tremolosecurity.proxy.az.CustomAuthorization`

| Name | Type | Description |
|------|------|-------------|
| className | string | Implementation of `com.tremolosecurity.proxy.az.CustomAuthorization` |
| params | object | List of configuration options.  Properties can have the value of a `string` or array of `string` |


## AuthenticationMechanism

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | authmech |
| **Plural** | authmechs |

### Version: v1

Defines a way to authenticate users

| Name | Type | Description |
|------|------|-------------|
| uri | string | The path of a URL that triggers this method, **MUST** start with `/auth/` |
| className | string | Implementation of `com.tremolosecurity.proxy.auth.AuthMechanism` |
| init | object | Initialization parameters for when this mechanism is created.  These parameters are global for all chains.  Each parameter can have a value of a single `string` or an array of `string` |
| secretParams | array | Pull secret data directly from a Kubernetes `Secret` [Details](#root-secretparams) |


###<a name="root-secretparams"></a> secretParams

Pull secret data directly from a Kubernetes `Secret`

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the configuration option |
| secretName | string | The name of the Kubernetes `Secret` |
| secretKey | string | The name of the key in the `data` section of the Kubernetes `Secret` to pull the value from |


## AuthenticationChain

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | authchain |
| **Plural** | authchains |

### Version: v1

An authentication chain links together multiple mechanisms to authenticate a user

| Name | Type | Description |
|------|------|-------------|
| level | integer | The strength of the chain.  If a user is already authenticated using a chain of equal or higher strength then authenticaiton is not re-run. |
| finishOnRequiredSucess | boolean | If `true`, fill short-circuit a chain that has mechanisms which aren't required once the required ones finish.  Generally left to `false` |
| root | string | Where in the OpenUnison virtual directory to pull users from.  Usually `o=Tremolo` |
| compliance | object | Configures protection against brute-force attacks.  See https://openunison.github.io/UPDATE for detailed configuration instructions [Details](#root-compliance) |
| authMechs | array | List of mechanisms in this chain [Details](#root-authmechs) |


###<a name="root-compliance"></a> compliance

Configures protection against brute-force attacks.  See https://openunison.github.io/UPDATE for detailed configuration instructions

| Name | Type | Description |
|------|------|-------------|
| enabled | boolean |  |
| maxFailedAttempts | integer |  |
| maxLockoutTime | integer |  |
| numFailedAttribute | string |  |
| lastFailedAttribute | string |  |
| lastSucceedAttribute | string |  |
| updateAttributesWorkflow | string |  |
| uidAttributeName | string |  |


###<a name="root-authmechs"></a> authMechs

List of mechanisms in this chain

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of an `AuthenticationMechanism` |
| required | string | Determines if a mechanism satisfies a chain.  Mostly should be `required` |
| params | object | Configuration parameters.  Values can be a `string` or array of `string` |
| secretParams | array | Configuraiton options that come directly from Kubernetes `Secret` objects [Details](#root-authmechs--secretparams) |


####<a name="root-authmechs--secretparams"></a> secretParams

Configuraiton options that come directly from Kubernetes `Secret` objects

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the configuration option |
| secretName | string | The Kubernetes `Secret` to pull the value from |
| secretKey | string | The name of the key in the `data` section of the source `Secret` |


## Application

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | application |
| **Plural** | applications |

### Version: v2

Contains the configuration for a web application, identity provider, or API

| Name | Type | Description |
|------|------|-------------|
| azTimeoutMillis | integer | Number of milliseconds authorization decisions should be cached for |
| isApp | boolean | If true, the application is assumed to be a reverse proxy or local API.  If false, the application is an identity provider |
| urls | array | List of URLs that make up an application [Details](#root-urls) |
| cookieConfig | object | Determines how an `Application` manages its session via cookies [Details](#root-cookieconfig) |


###<a name="root-urls"></a> urls

List of URLs that make up an application

| Name | Type | Description |
|------|------|-------------|
| hosts | array | List of host names that this URL is valid for.  May be `*` for all hosts |
| filterChain | array | List of filters that can manipulate request and response headers [Details](#root-urls--filterchain) |
| proxyTo | string | **Optional** If isApp is true, determines the URL to forward the request to.  typically uses the form `https://host:port${fullURI}` where `${fullURI}` is replaced with the path section of the requested URL |
| proxyConfiguration | object | Provides timeout configurations for proxying to the `proxyTo` URL.  If omited, there are no timeouts [Details](#root-urls--proxyconfiguration) |
| uri | string | The path of the URL, the portian after the host that this URL will match on |
| regex | boolean | if true, the `uri` is interpreted as a regular expression |
| authChain | string | The name of an `AuthenticationChain` that will be used to authenticate access to this URL |
| overrideHost | boolean | If true, the the proxied request will include the original request's `HOST` header |
| overrideReferer | boolean | If true, and referals' hosts are overriden with the orginal request's `HOST` header |
| results | object | Actions that will triger as the result to an authentication or authorization event [Details](#root-urls--results) |
| azRules | array | List of rules used to authorize access.  Authorization succeeds if **ANY** of these rules pass [Details](#root-urls--azrules) |
| idp | object | Configuration for an `Application` that acts as an identity provider [Details](#root-urls--idp) |


####<a name="root-urls--filterchain"></a> filterChain

List of filters that can manipulate request and response headers

| Name | Type | Description |
|------|------|-------------|
| className | string | Implementation of `com.tremolosecurity.proxy.filter.HttpFilter` |
| params | object | List of configuration parameters.  Parameters may be a `string` or an `array` of `string` |
| secretParams | array | Load parameters directly from Kubernetes `Secret` objects [Details](#root-urls--filterchain--secretparams) |


#####<a name="root-urls--filterchain--secretparams"></a> secretParams

Load parameters directly from Kubernetes `Secret` objects

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the filter parameter |
| secretName | string | The name of the Kubernetes `Secret` |
| secretKey | string | The name of the key in the `data` section of the `Secret` |


####<a name="root-urls--proxyconfiguration"></a> proxyConfiguration

Provides timeout configurations for proxying to the `proxyTo` URL.  If omited, there are no timeouts

| Name | Type | Description |
|------|------|-------------|
| connectionTimeoutMillis | integer | The number of milliseconds an attempted connection will wait until timing out.  Defaults to `0`, which is no timeout |
| requestTimeoutMillis | integer | The number of milliseconds in an individual request will wait until timing out.  Defaults to `0`, which is no timeout |
| socketTimeoutMillis | integer | The number of milliseconds a socket can remain open without receiving any data.  Defaults to `0`, which is no timeout |


####<a name="root-urls--results"></a> results

Actions that will triger as the result to an authentication or authorization event

| Name | Type | Description |
|------|------|-------------|
| auSuccess | string | The name of a `ResultGroup` that will be executed when a user successfully authenticates to this URL |
| auFail | string | The name of a `ResultGroup` that will be executed when a user fails to authenticate to this URL |
| azSuccess | string | The name of a `ResultGroup` that will be executed when a user is successfully authorized for this URL |
| azFail | string | The name of a `ResultGroup` that will be executed when a user is failed to be authorized for this URL |


####<a name="root-urls--azrules"></a> azRules

List of rules used to authorize access.  Authorization succeeds if **ANY** of these rules pass

| Name | Type | Description |
|------|------|-------------|
| scope | string | One of:<br>- `group` - The DN of a group the suer must be a member of in OpenUnison's virtual directory<br>- `dynamicGroup` - The DN of an LDAP dynamic group the suer must be a member of in OpenUnison's virtual directory<br>- `dn` - The DN of an area of the virtual directory where all users under that DN are authorized<br>- `custom` - The name of a `CustomAuthorization`.  Can be followed by a `!` with parameters.  For instance `SomeCustomAZ!param1!param2` will call the custom authorization `SomeCustomAZ` with the parameters `param1` and `param2`<br>- `filter` - An LDAP filter that will be applied to a logged in user |
| constraint | string | How to enforce this rule based on the `scope` |


####<a name="root-urls--idp"></a> idp

Configuration for an `Application` that acts as an identity provider

| Name | Type | Description |
|------|------|-------------|
| className | string | Implementation of `com.tremolosecurity.idp.server.IdentityProvider` |
| params | object | properties to be passed to the identity provider.  Values can be either a `string` or an array of `string` |
| secretParams | array | Load parameters directly from Kubernetes `Secret` objects [Details](#root-urls--idp--secretparams) |
| mappings | object | Provides mappings from a user's attributes into an identity provider's assertions [Details](#root-urls--idp--mappings) |
| trusts | array | List of trusted clients [Details](#root-urls--idp--trusts) |


#####<a name="root-urls--idp--secretparams"></a> secretParams

Load parameters directly from Kubernetes `Secret` objects

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the identity provider parameter |
| secretName | string | The name of the Kubernetes `Secret` |
| secretKey | string | The name of the key in the `data` section of the `Secret` |


#####<a name="root-urls--idp--mappings"></a> mappings

Provides mappings from a user's attributes into an identity provider's assertions

| Name | Type | Description |
|------|------|-------------|
| strict | boolean | If true, only mapped attributes will be included in the assertion.  If false, all attributes and mapped attributes are included. |
| map | array | list of mappings [Details](#root-urls--idp--mappings--map) |


######<a name="root-urls--idp--mappings--map"></a> map

list of mappings

| Name | Type | Description |
|------|------|-------------|
| targetAttributeName | string | The name of the attribute as it will appear in the assertion |
| targetAttributeSource | string | Depends on `sourceType`:<br>- `static` - The specific value of the attribute<br>- `user` - The name of the attribute from the user to use<br>- `composite` - Assemble an attribute from the first values of other attributes and static content.  For instance `${givenName} ${sn}` where `givenName` is `First` and `sn` is `Last` will create the value `First Last`<br>- `custom` - Implementation of `com.tremolosecurity.provisioning.mapping.CustomMapping`, parameters can be supplied after a `|` that's comma delimited.  For instance, `com.tremolosecurity.mapping.JavaScriptMapping|k8s,openunison,argocd-groups` Uses the `JavaScriptMapping`, passing the parameters `k8s`, `openunison`, and `argocd-groups` as parameters |
| sourceType | string |  |


#####<a name="root-urls--idp--trusts"></a> trusts

List of trusted clients

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the trust |
| params | object | The parameters to pass the to trust configuration.  Values may be a `string` or an array of `string` |
| secretParams | array | Load parameters directly from Kubernetes `Secret` objects [Details](#root-urls--idp--trusts--secretparams) |


######<a name="root-urls--idp--trusts--secretparams"></a> secretParams

Load parameters directly from Kubernetes `Secret` objects

| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the trust parameter |
| secretName | string | The name of the Kubernetes `Secret` |
| secretKey | string | The name of the key in the `data` section of the `Secret` |


###<a name="root-cookieconfig"></a> cookieConfig

Determines how an `Application` manages its session via cookies

| Name | Type | Description |
|------|------|-------------|
| sessionCookieName | string | Name of the cookie |
| domain | string | The domain the cookie applies to.  Can be `*` to apply to any domain the URLs of the application accept |
| scope | integer | ignore |
| logoutURI | string | The URI, or path in the URL, that triggers the ending of the session |
| keyAlias | string | The name of the key in OpenUnison's keystore used to encrypt the session cookie |
| timeout | integer | The time, in seconds, that an idle session should be discarded |
| secure | boolean | If true, cookies require an HTTPS connection |
| httpOnly | boolean | If true, cookies are not accessible from javascript |
| sameSite | string | Used to set the `SameSite` option in a cookie.  One of `None`, `Lax`, `Strict`, `Ignore` |
| cookiesEnabled | boolean | If true, sessions are enabled.  If false (such as for an API), no session is generated or tracked |


## GroupMetaData

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | groupmetadata |
| **Plural** | groupmetadatas |

### Version: v1

Stores mapping from an external group to an internal group.  Can also be used to create a group in the database without executing SQL

| Name | Type | Description |
|------|------|-------------|
| groupName | string | The name of the group to create |
| externalName | string | **Optional** - the name of the external group to map to |


## JavaScriptMapping

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | javascriptmapping |
| **Plural** | javascriptmappings |

### Version: v1

Stores the javascript for a custom mapping

| Name | Type | Description |
|------|------|-------------|
| javascript | string | JavaScript function for the mapping.  Must contain at least one funciton named `doMapping` with two arguments: `user` and `attributeName` |


## WaitForState

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | waitforstate |
| **Plural** | waitforstates |

### Version: v1

Stores the state of a workflow waiting for an object to reach a certain status

| Name | Type | Description |
|------|------|-------------|
| state | string | The encrypted state of the workflow waiting for the expected status before continuing |


## Notifier

| Attribute | Value |
| --------- | ----- |
| **Group** | openunison.tremolo.io |
| **Scope** | Namespaced |
| **Singular** | notifier |
| **Plural** | notifiers |

### Version: v1

Configures a custom notification mechanism for OpenUnison

| Name | Type | Description |
|------|------|-------------|
| className | string | Implementation of `com.tremolosecurity.openunison.notifications.NotificationSystem` |
| params | array | Set of configuration parameters [Details](#root-params) |
| secretParams | array |  [Details](#root-secretparams) |


###<a name="root-params"></a> params

Set of configuration parameters

| Name | Type | Description |
|------|------|-------------|
| name | string | Configuration parameter name, may be listed multiple times |
| value | string | Value |


###<a name="root-secretparams"></a> secretParams



| Name | Type | Description |
|------|------|-------------|
| name | string | The name of the configuration parameter |
| secretName | string | The name of the Kubernetes `Secret` |
| secretKey | string | The name of the key in the `Secret` to pull the data from |


