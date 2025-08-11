# Authentication Mechanisms

Authentication Mechanisms define the ways in which a user can be authenticated.
Prior to being added to an authentication chain, a mechanism must
be defined in the section. Every
authentication method has its own configuration parameters. See the
Authentication Mechanisms section of the Configuration Reference for configuration options on
specific mechanisms.  Here's an example of the form login mechanism:

```yaml

---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: login-form
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.FormLoginAuthMech
  uri: "/auth/formlogin"
  init: {}
  secretParams: []
```

Once a mechanism is configured, its available to an authentication chain.

## Authentication Chains

Mechanisms are tied together in chains that allow for multiple mechanisms to be executed when a user attempts to authenticate.  This way, an application can require multiple mechanisms
for authentication.  For instance, the user may first login with a username and password, but then be required to accept terms and conditions.  A chain can also be used for multi-factor
authentication.  Once mechanisms are available, they can be added to a chain.

The below YAML shows an example chain:

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  # list of AuthenticationMechanism objects
  authMechs:
  # authentication with the OpenID Connect mechanism
  - name: oidc
    # each parameter can be a string or a list of strings
    params:
      bearerTokenName: oidcBearerToken
      defaultObjectClass: inetOrgPerson
      forceAuthentication: "true"
      hd: ""
      issuer: https://login.microsoftonline.com/xxxxxxxxxxxxxxx
      linkToDirectory: "false"
      lookupFilter: (uid=${sub})
      noMatchOU: oidc
      responseType: code
      scope: openid email profile
      uidAttr: sub
      userLookupClassName: com.tremolosecurity.unison.proxy.auth.openidconnect.loadUser.LoadAttributesFromWS
    # if a mechanism isn't required, this can be "optional"
    required: required
    # list of parameter from Kubernetes secrets
    secretParams:
      # the name of the parameter
    - name: secretid
      # the key in the secret's data section
      secretKey: OIDC_CLIENT_SECRET
      # the name of the secret
      secretName: orchestra-secrets-source#[openunison.static-secret.suffix]
    - name: clientid
      secretKey: OIDC_CLIENT_ID
      secretName: orchestra-secrets-source
  - name: map
    params:
      map:
      - uid|composite|${upn}
      - mail|composite|${email}
      - givenName|composite|${given_name}
      - sn|composite|${family_name}
      - displayName|composite|${name}
      - memberOf|user|roles
    required: required
  - name: include
    params:
      chainName: azuread-load-groups
    required: required
  - name: jit
    params:
      nameAttr: uid
      workflowName: jitdb
    required: required
  - name: genoidctoken
    params:
      idpName: k8sidp
      trustName: kubernetes
    required: required
  # the authentication level for this chain
  level: 20
  # the root of the internal LDAP virtual directory to load the user object from
  root: o=Data
```

### Compliance

In order to comply with most security requirements systems must be protected from brute force attacks where a user can repeatedly try
to authenticate with passwords until one is found.  OpenUnison can help protect against this by adding a compliance section to an
authentication chain.  This will trigger OpenUnison to track the last time a user failed to log in, succeeded and how many failures
have occurred.  In order to use this feature the following attributes must be available in the virtual directory:

| Description                     | Data Type                      | Description                                                                 |
|---------------------------------|---------------------------------|-----------------------------------------------------------------------------|
| Last Successful Authentication  | Directory String OR Long Integer | Stores the last successful authentication in the form of the number of milliseconds since epoch |
| Last Failed Authentication      | Directory String OR Long Integer | Stores the last failed authentication attempt in the form of the number of milliseconds since epoch |
| Number of failed authentications| Directory String or Integer     | The number of failed attempts since the last success                        |

In addition to these attributes, a workflow will need to be created that can update these attributes as needed.  An example:

```yaml
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: updateLockout
  namespace: openunison
spec:
  description: compliance
  inList: false
  label: compliance
  orgId: 687da09f-8ec1-48ac-b035-f2f182b9bd1e
  dynamicConfiguration:
    dynamic: false
    className: ""
    params: []
  tasks: |-
      # if there's no unisonFailedLogins supplied by OpenUnison to the workflow, load the current value
      - taskType: ifAttrExists
        name: unisonFailedLogins
        onFailure:
        - taskType: customTask
          className: com.tremolosecurity.provisioning.customTasks.LoadAttributes
          params:
            name: unisonFailedLogins
            nameAttr: uid

      # if there's no unisonLastFailedLogin supplied by OpenUnison to the workflow, load the current value
      - taskType: ifAttrExists
        name: unisonLastFailedLogin
        onFailure:
        - taskType: customTask
          className: com.tremolosecurity.provisioning.customTasks.LoadAttributes
          params:
            name: unisonLastFailedLogin
            nameAttr: uid

      # if there's no unisonLastSuccessLogin supplied by OpenUnison to the workflow, load the current value
      - taskType: ifAttrExists
        name: unisonLastSuccessLogin
        onFailure:
        - taskType: customTask
          className: com.tremolosecurity.provisioning.customTasks.LoadAttributes
          params:
            name: unisonLastSuccessLogin
            nameAttr: uid

      # provision the compliance attributes to the target
      - taskType: provision
        sync: true
        target: activedirectory
        setPassword: false
        onlyPassedInAttributes: true
        attributes:
        - unisonFailedLogins
        - unisonLastFailedLogin
        - unisonLastSuccessLogin
        - uid
```

Once your workflow is defined, you will add the `compliance` section to your `AuthenticationChain` object:

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  compliance:
    # determines if compliance on this chain should be enabled
    enabled: true
    # the name of the attribute to store the timestamp as milliseconds from epoch since the last filed login
    lastFailedAttribute: unisonLastFailedLogin
    #the name of the attribute to store the timestamp as milliseconds from epoch since the last successful login
    lastSucceedAttribute: unisonLastSuccessLogin
    # the maximum number of failed attempts before the account is locked
    maxFailedAttempts: 5
    # The amount of time in seconds that an account should be locked after the maximum number of failed attempts occurs
    maxLockoutTime: 900
    #the name of the attribute to store the number of failed login attempts
    numFailedAttribute: unisonFailedLogins
    # the attribute that stores a user's unique id
    uidAttributeName: uid
    # the name of the workflow that is used to update compliance data
    updateAttributesWorkflow: updateLockout
  # list of AuthenticationMechanism objects
  authMechs:
  .
  .
  .
```

### Authentication Levels

Each chain has an authentication level.  This level is used to determine how "strong" the authentication is.  If a URL is configured with a chain of a certain level, OpenUnison will
respond in one of two ways:

1.  If the level of the chain on the URL is HIGHER then the level the user is currently authenticated to then the user will be forced to authenticate with the chain configured on the URL
2. If the level of the chain on the URL is EQUAL to or LESS then the level the user is currently authenticated to then the user will NOT be forced to re-authenticate

This allows you to mix and match authentication types depending on the user base.  For instance US Federal Government workers have PIV cards, but private industry might have a TOTP
credential.  By using authentication levels you can ensure they have access to the same resources.  A best practice is to assign levels in alignment with NIST 800-63 but in 10s instead
of 1s:

* 0 - Anonymous
* 10 - No level of assurance
* 20 - Some level of assurance
* 30 - Medium level of assurance
* 40 - High level of assurance

This way you can differentiate between different types of authentication in the same category providing some additional room for customization.

### Pre-Built Mechanisms

#### FormLoginAuthMech

##### Mechanism

An HTML login form. See auth/forms/default-form.jsp.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: login-form
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.FormLoginAuthMech
  uri: "/auth/formlogin"
  init: {}
  secretParams: []
```

##### Chain

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      # URL path to the login form, a JSP file
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  level: 20
  root: o=Tremolo
```

#### SAML2Auth

This mechanism is used to authenticate the user using a SAML2 assertion. The HTTP-POST and HTTP-REDIRECT profiles are supported.

##### Mechanism 

Some identity providers, such as Active Directory Federation Services, do not have a way of providing a default RelayState for IdP Initiated SSO. In such cases, a mapping from the Referer HTTP header to a default relay state may be configured on the mechanism.  See the command line reference for how to import and export metadata easily.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: saml2
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.SAML2Auth
  uri: "/auth/saml2"
  init: {}
  secretParams: []
```

##### Chain

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  authMechs:
  - name: saml2
    required: required
    params:
      # provide either a URL to a published saml2 metadata document or a path to a metadata document accessible to OpenUnison
      metadataUrl: https://myid.domain/path/to/metadata.xml
      # list of keys to validate signatures, these keys will be populated automatically from the metadata.  If multiple keys will be provided, add a number after a dash.  for instance if your identity provider provides two keys, add a -1 at the end of the second one. 
      idpSigKeyName: 
      - idp-saml2-sig
      - idp-saml2-sig-1
      # The RDN for an account that can not be linked to a user in the virtual directory
      ldapAttribute: uid
      # The OU of an account that can not be linked to a user in the virtual directory 
      dnOU: saml
      # Object Class for an account that can not be linked to a user in the virtual directory
      defaultOC: inetOrgPerson
      # The required authentication mechanism to be used by the identity provider
      authCtxRef: "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport"
      # The certificate used to sign all outbound requests (AuthnRequest and LogoutRequest)
      spSigKey: unison-saml2-rp-sig
      # The algorithm used to sign AuthnRequests and LogoutRequests
      sigAlg: "RSA-SHA256"
      # true if outbound requests should be signed
      signAuthnReq: "true"
      # Determine if inbound assertions must be signed
      assertionsSigned: "true"
      # Determine if inbound SAMLResponse must be signed
      responsesSigned: "false"
      # If set to true, requires inbound assertions to be encrypted
      assertionEncrypted: "false"
      # If assertionEncrypted is set to true, names the key used to decrypt the assertion
      spEncKey: ""
      # If set to true, will force outbound requests to secure connections
      forceToSSL: "true"
      # If true, OpenUnison will not attempt to find the user in the virtual directory, usually used with just-in-time provisioning
      dontLinkToLDAP: "true"
      # The final URL users are sent to after logout is complete
      logoutURL: "https://#[K8S_DASHBOARD_HOST]/logout"
      #  The SingleLogout URL of the Identity Provider
      idpRedirLogoutURL: "https://myid.domain/path/to/logout"
  level: 20
  root: o=Tremolo
```

#### AnonAuth

Anonymous authentication is used for scenarios when user authentication is not needed.  ***NOTE***: this mechanism is prebuilt into the OpenUnison container and does not need to be created.  The documentation is here for consistency.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: anon
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.AnonAuth
  uri: "/auth/anon"
  init:
    userName: "uid=Anonymous"
  secretParams: []
```

##### Chain

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: anon
  namespace: openunison
spec:
  authMechs:
  - name: anon
    required: required
    params: {}
  level: 0
  root: o=Tremolo
```

#### BasicAuth

HTTP Basic Authentication.


##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: basic
  namespace: {{ .Release.Namespace }}
spec:
  className: com.tremolosecurity.proxy.auth.BasicAuth
  uri: "/auth/basic"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: api-idp
  namespace: openunison
spec:
  level: 20
  root: o=Tremolo
  authMechs:
  - name: basic
    required: required
    params:
      # the name of the realm to respond to the client with on a failed authentication (or anonymous)
      realmName: k8s-api
      # the attribute to look the user up by
      uidAttr: uid
    secretParams: []
```

#### IWAAuth

IWA, or Integrated Windows Authentication, allows a user to authenticate using their current windows Kerberos token. For IWA to work the user MUST be logged into a desktop that is a member of one of the domains configured on this mechanism. NTLM is NOT supported.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: iwa
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.IWAAuth
  uri: "/auth/iwa"
  init:
    # The fully qualified domain name of the domain
    domain: 
    - ENTERPRISE.DOMAIN.COM
    # The hostname or IP of the KDC, repeat for each domain
    ENTERPRISE.DOMAIN.COM.kdc: kdc.tremolo.lan
    # The service account to use. repeat for each domain
    ENTERPRISE.DOMAIN.COM.userName: tremoloSvc@ENTERPRISE.DOMAIN.COM
  secretParams:
  # The service account password, should be repeated for each domain
  - name: ENTERPRISE.DOMAIN.COM.password
    secretName: orchestra-secrets-source
    secretKey: enterprise-domain-com-password
```

##### Chain

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  authMechs:
  - name: iwa
    required: sufficient
    params: {}
    secretParams: []
  # Fallback to form if IWA isn't available 
  - name: login-form
    required: sufficient
    params:
      # URL path to the login form, a JSP file
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  level: 20
  root: o=Tremolo
```

#### CertAuth

This mechanism supports authentication using TLS certificates. If the certificate can be associated with a user in the directory it will be, otherwise a user object is created. Note that in order for sslCert mechanisms to work certificate authentication must either be optional or required on the container.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: tlsauth
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.CertAuth
  uri: "/auth/tls"
  init:
    # Comma separated list of CRLs to check
    crl.names: "openssl,ldapcrl,ocsp"

    # CRLs are loaded once a minute
    crl.openssl.type: "com.tremolosecurity.proxy.auth.ssl.FileCRL"
    # Path to the CRL file
    crl.openssl.path: "crl.pem"

    # For checking a CRL stored in an LDAP object
    crl.ldapcrl.type: "com.tremolosecurity.proxy.auth.ssl.LdapCRL"
    crl.ldapcrl.path: "ldap://127.0.0.1:10983/cn=signing,ou=internal,dc=domain,dc=com"

    # For checking against OCSP
    crl.ocsp.type: "com.tremolosecurity.proxy.auth.ssl.OCSP"
    crl.ocsp.path: "http://127.0.0.1:2389"

    # Extracts can be used to get attributes out of a certificate besides the subject dn and subject alternative names
    # Must implement com.tremolosecurity.proxy.auth.CertificateExtractSubjectAttribute, may be listed multiple times
    extracts: ""
  secretParams: []
```

##### Chain

When authenticating a user using certificates the chain configuration specifies how to identify a user and link them to a user in the directory. If a user can?t be linked in the directory then a user object is created based on the components of the DN.

When a certificate has subject alternative names they are added as potential components or attributes. These attribute names are:

* otherName
* email
* dNSName
* x400Address
* directoryName
* ediPartyName
* uniformResourceIdentifier
* iPAddress
* registeredID

Any of these attributes are available to the matching filter for directory lookups or in the DN of an unmatched entry.

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  authMechs:
  - name: tlsauth
    required: required
    params:
        # Either an attribute name OR an ldap filter mapping the certificate DN components.
        # If this is an ldap filter, DN components are identified by ${component}
        uidAttr: "(uid=${CN})"

        # If the UID Attribute is a filter or just an attribute name
        uidIsFilter: "true"

        # A list of attributes in the certificate subject, or subject alternative names, that will be the RDN of an unmatched entry.
        rdnAttribute: "CN"

        # The object class to use for objects created because the user doesn’t exist in the directory
        defaultOC: "inetOrgPerson"

        # The ou component of the DN to use for users not matched.
        # For instance, if SSL is specified, the user’s DN would be uid=user,ou=SSL,o=Tremolo
        dnLabel: "ou=CertAuth"

        # List of DNs of trusted certificates that the chain will accept
        issuer: "OU=Dev, O=Tremolo Security Inc., C=US, ST=Virginia, CN=Root CA"

    secretParams: []
  level: 20
  root: o=Tremolo
```

#### UserOnlyAuthMech

An HTML login form that ONLY collects a username. This mechanism is convinient when using a custom authentication scheme or authentication system that doesn't have a password (like SMS). All login forms must be stored in the auth/forms directory. Forms can be static HTML or JSP pages. See auth/forms/userOnlyLogin.jsp.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: userOnlyForm
  namespace: openunison
spec:
  className: om.tremolosecurity.proxy.auth.UserOnlyAuthMech</className>
  uri: "/auth/userOnlyLogin"
  init: {}
  secretParams: []
```

##### Chain

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  authMechs:
  - name: userOnlyLoginChain
    required: required
    params:
        # The URI for the JSP page used to log the user in
        loginJSP: "/auth/forms/userOnlyLogin.jsp"

        # Either an attribute name OR an LDAP filter mapping the form parameters.
        # If this is an LDAP filter, form parameters are identified by ${parameter}
        uidAttr: "uid"

        # If true, the user is determined based on an LDAP filter rather than a simple user lookup
        uidIsFilter: "false"

        # The URI for the JSP page used when a user can't be found
        noUserJSP: "/auth/forms/noUser.jsp"

    secretParams: []
  level: 20
  root: o=Tremolo
```

#### AcknowledgeAuthMech

The Banner Acknowledge mechanism provides a way to make a user acknowledge a set of policies prior to logging in. Adding this mechanism to a chain to record the acknowledgement in the authorization logs. The stock acknowledgement form is in /auth/forms/acknowledge.jsp.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: acknowledge
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.AcknowledgeAuthMech
  uri: "/auth/acknowledge"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: login
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  - name: acknowledge
    required: required
    params:
      # The URI for the JSP page to host the banner and request acknowledgement
      loginJSP: "/auth/forms/acknowledge.jsp"
      # The text of the banner, may be HTML
      banner: "I acknowledge that I shall do no harm"
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### TwilioSMS

This mechanism allows for single use password to be used and sent over SMS to a user?s mobile phone via Twilio. Note, a Twilio account is required to use this mechanism.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: smsauth
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.otpsms.TwilioSMS
  uri: "/auth/smsauth"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: 2fasms
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/default-form.jsp"
    secretParams: []
  - name: smsauth
    required: required
    params:
      # Twilio Source Phone Number
      fromNumber: "1234567890"
      # The attribute that stores the user's phone number
      toAttrName: "mobile"
      # URI for the form to collect the login key
      redirectForm: "/auth/forms/smsKey.jsp"
      # The message to be sent to the user. '${key}' is used to represent the single-use password.
      message: "${key}"
      # The length of the single-use password
      keyLength: "10"
      # Checked if the
    secretParams:
    # Twilio Account SID
    - name: accountSID
      secretName: orchestra-secrets-source
      secretKey: accountSID
    # Twilio Account token
    - name: authToken
      secretName: orchestra-secrets-source
      secretKey: authToken
```

#### SecretQuestionAuth

This mechanism allows for secret or ?golden? to be used as a password. The answers are stored in JSON as an attribute on the user?s object and are hashed. All questions and answers are encrypted.  NOTE: this mechanism is designed to be used with the com.tremolosecurity.proxy.auth.secret.CreateSecretQuestionsTask provisioning task.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: SecretQuestions
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.secret.SecretQuestionAuth
  uri: "/auth/secretQuestions"
  init:
    # List of questions
    questionList: 
      - "What is your favorite team?"
      - "What is your favorite color?"
      - "What is your favorite pet?"
      - "What is your favorite town?"
      - "What is your favorite candy?"
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: login-form
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  - name: SecretQuestions
    required: required
    params:
      # The attribute that stores the base64 encoded JSON
      questionAttr: "jpegPhoto"
      # The URI of the secret question answer form
      login-form: "/auth/forms/secretQuestions.jsp"
      # One way hash to use
      alg: "SHA-512"
    secretParams:
    # Used to randomize the hash
    - name: salt
      secretName: orchestra-secrets-source
      secretKey: salt
  root: o=Tremolo
```

#### LoginService

The Login Service mechanism provides a way to give users a choice in how they login. This is useful in situations where the user could have multiple tokens and different levels of authentication. For instance, in a scenario where a user might be able to use 2-factor authentication when they have a token or a single factor when they don't.

The way the login service works is it redirects the user to another Application URL for authentication. Once the chain for that application URL is completed the user is re-directed back to the original request (with post preservation). This provides the advantage of providing the authentication level of the desired chain. Each application URL should be configured with the com.tremolosecurity.prelude.filters.CompleteLogin filter. This will complete the login process.

NOTE: If only one option is configured in the service it will NOT prompt the user to choose.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: loginservice
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.LoginService
  uri: "/auth/loginservice"
  init: {}
  secretParams: []
```

##### Chain

This mechanism should be the ONLY mechanism on a chain. In addition, the chain should be set at the lowest level of the other authentication chains involved. For instance if a chain on /login/ssl is set at 40 and another chain on /login/form is set at 20 then this chain should be 20.

For each login choice, the steps are

* Create an authentication chain
* Create an Application URL
* Associate the URL with the chain
* On the Application URL, add the com.tremolosecurity.prelude.filters.CompleteLogin filter
* Add the URI for this URL to the chain configuration for the login service

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: loginservice
  namespace: openunison
spec:
  authMechs:
  - name: loginservice
    required: required
    params:
      # Chains parameters point to the login endpoint for a choice and the label for the user's form
      chains:
        - "SAML2 Login=/login/saml2"
        - "Username and Password=/login/form"
      # The URI of the login method select page
      serviceUrl: "/auth/forms/chooseLogin.jsp"
      # The name of the cookie to store the user's decision to keep the user's decision to save the choice
      cookieName: "tremoloLoginChoice"
      # The number of days the cookie that determines if the user wants to remember their login choice will be valid
      cookieDays: "30"
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### OAuth2BearerLastMile

This mechanism allows for the use of a Unison Last Mile token to be used as a bearer token for OAuth2. The token must have one attribute named dn that maps to the user's DN in Unison's virtual directory.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: oauth2-lm
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.oauth2.OAuth2BearerLastMile
  uri: "/auth/oath2-lm"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: oauth2-lm
  namespace: openunison
spec:
  authMechs:
  - name: oauth2-lm
    required: required
    params:
      # The realm for the bearer token
      realm: "MyEntWS"
      # The scope of the token
      scope: "Mobile"
      # The encryption key used to decrypt the token
      encKeyAlias: "sessionkey"
      # Determines if the user is found based on a user attribute or the user's DN
      lookupByAttribute: "true"
      # If looking up by attribute, which attribute to search?
      lookupAttributeName: "uid"
      # If true, Unison will look at the URI instead of the scope to verify the LastMile
      useURIForLastMile: "true"
    secretParams: []
  level: 5
  root: o=Tremolo
```

#### OAuth2JWT

This mechanism will allow a JWT to be used as an OAuth2 Bearer Token.  The token's signature is validated along with the issuer and timestamps.  The mechanism can look the user up or create an object based on the claims in the token.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: oauth2jwt
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.oauth2.OAuth2JWT
  uri: "/auth/oauth2jwt"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: oauth2-linked
  namespace: openunison
spec:
  authMechs:
  - name: oauth2jwt
    required: required
    params:
      # The required Issuer
      issuer: "https://k8sou.tremolo.lan/auth/idp/k8sIdp"
      # If true, OpenUnison will pull URLs and public keys from the well-known URL
      fromWellKnown: "true"
      # Certificate entry to use to validate the JWT's signature ONLY NEEDED if fromWellKnown is false
      validationKey: "jwt-sig"
      # If true, the user will be looked up in the virtual directory
      linkToDirectory: "true"
      # The OU in the DN to use if the user isn't found in the virtual directory
      noMatchOU: "oauth2"
      # If not linked to a user, the claim to use as the user identifier
      uidAttr: "uid"
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

#### OAuth2K8sServiceAccount

This mechanism will validate a Kubernetes service account token against the cluster it was issued from.  Once validated, the token is either mapped to a user or an ephemeral user is created for the session. 

##### Mechanism

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

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: oauth2-linked
  namespace: openunison
spec:
  authMechs:
  - name: oauth2k8s
    required: required
    params:
      # The name of the Kubernetes target
      k8sTarget: "k8s"
      # If true, the user will be looked up in the virtual directory
      linkToDirectory: "true"
      # The OU in the DN to use if the user isn't found in the virtual directory
      noMatchOU: "oauth2"
      # If not linked to a user, the claim to use as the user identifier
      uidAttr: "uid"
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

#### JITAuthMech

Runs a workflow to provision accounts "just-in-time" during authentication.  The workflow is run synchronously, so an error causes the authentication to fail.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: jit
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.auth.JITAuthMech
  uri: "/auth/jit"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: jit
  namespace: openunison
spec:
  authMechs:
  - name: jit
    required: required
    params:
      # The name of the attribute that will be used as the user identifier
      nameAttr: "uid"
      # The name of the workflow
      workflowName: "JIT-Saml2"
      # 1.0.43+
      # optional - enables JIT to only be run outside of timeframes.  useful if your using with an API that runs at a high rate
      # # number of seconds between JIT runs
      # gracePeriod: 60
      # # An attribute on the user's authentication object that represents the number of milliseconds since the last time the account was updated.
      # # this attribute should be updated within the JIT workflow
      # lastUpdatedAttributeName: lastUpdated
      # # The base DN to reload the object from if the JIT workflow isn't run
      # reloadBaseDN: o=Tremolo
    secretParams: []
  level: 5
  root: o=Tremolo
```

#### LoadLastUpdatedAuth

*1.0.43+ only*

Loads the last updated attribute from OpenUnison's Kubernetes Namespace as a Service (NaaS) database.

##### Mechanism

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: load-last-updated
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.LoadLastUpdatedAuth
  init: {}
  secretParams: []
  uri: /auth/loadlastupdated
```

##### Chain

```yaml
  - name: load-last-updated
    params:
      # the attribute to be loaded from the db, also the attribute to be added the user's authentication object.  Should be the same of the JITAuthMech's lastUpdatedAttributeName chain configuration.
      lastUpdatedForUser: lastUpdated
      # The SQL to load the value from the database
      sql: SELECT lastUpdated FROM localUsers WHERE sub=?
      # the target to load the attribute from
      target: jitdb
      # The user's authentication object's unique id attribute
      uidAttributeName: uid
    required: required
```

#### PersistentCookie

The persistent cookie mechanism is used in situations where a heavy gui client (such as Office or Windows Explorer) uses http calls to do work but is unable to handle redirects or form based authentication. For instance when integrating with a webdav system that is protected by Unison. Using this mechanism, a user can be authenticated in a web browser but use Explorer or Office using a persistent cookie. This cookie is encrypted and has a certain lifespan beyond the life of the user's Unison session.

To enhance security this mechanism uses three levels of security:

| Feature                     | Description                                                                                           |
|-----------------------------|-------------------------------------------------------------------------------------------------------|
| AES-256 Encryption          | The cookie is encrypted using industry standard AES with 256 bit keys                                |
| Client IP Address           | The user's IP address is stored in the cookie, if the IP of the source of a request doesn't match this value the cookie is rejected |
| Optional - SSL Session ID   | The session id for the current SSL session can be stored as an extra layer of validation             |

When using this mechanism, the com.tremolosecurity.proxy.auth.persistentCookie.PersistentCookieResult custom result must be a cookie result on a Result Group that is on the Authentication Success result of the application configured with this mechanism.

Finally, when using with internet explorer the site that will generate and use this cookie MUST be "Trusted".

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: persistentcookie
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.persistentCookie.PersistentCookie
  uri: "/auth/cookie"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: saml2test
  namespace: openunison
spec:
  authMechs:
  - name: persistentcookie
    required: sufficient
    params:
      # The name of the cookie to generate. Scoping information is taken from the application's cookie configuration.
      cookieName: "login"
      # The number of milliseconds before this cookie needs to be re-generated. Defaults to 4 hours.
      millisToLive: "10000"
      # The name of the Last Mile Key from the Certificate Management screen to use to encrypt the cookie
      keyAlias: "lastmile"
      # If true, the user's SSL session ID is included in the validation process.
      useSSLSessionID: "true"
    secretParams: []
  - name: saml2test
    required: sufficient
    params:
      idpURL: "http://shib2x.tremolo.lan/idp/profile/SAML2/POST/SSO"
      idpRedirURL: "http://shib2x.tremolo.lan/idp/profile/SAML2/Redirect/SSO"
      idpSigKeyName: "shib-idp-sig"
      ldapAttribute: "uid"
      dnOU: "SAML2Test"
      defaultOC: "inetOrgPerson"
      authCtxRef: "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport"
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### OTPAuth

This mechanism provides a one-time-password using the OATH time based protocol (sometimes referred to as Google Authenticator). This mechanism needs to be paired with a workflow that contains the com.tremolosecurity.provisioning.customTasks.CreateOTPKey custom task for setting the user's encrypted key.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: totp
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.otp.OTPAuth
  uri: "/auth/totp"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: login
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  - name: totp
    required: required
    params:
      # The path to the form for entering the one-time-password
      formURI: "/auth/forms/otp.jsp"
      # The last mile key used to encrypt the user's token
      keyName: "sessionkey"
      # The name of the attribute to retrieve the user's token from
      attributeName: "givenName"
      # Number of valid 30-second windows that a token should remain usable
      windowSize: "4"
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### LogUserAgentAuth

This mechanism will log the user's browser (UserAgent header).

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: logagent
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.LogUserAgentAuth
  uri: "/auth/logagent"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: login
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  - name: logagent
    required: required
    params: {}
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### MappingAuthmech

This mechanism will perform a simple mapping of the user's authenticated object.  Its useful when using with SAML/OIDC before calling Just-In-Time provisioning.  This way the provisioning
engine can properly log all the attributes.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: map
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.MappingAuthmech
  uri: "/auth/map"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: login
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
    secretParams: []
  - name: map
    required: required
    params:
      # The map parameter can be repeated multiple times with the formation "new_name=pre_mapped_name"
      map:
        - "new1=old1"
        - "new2=old2"
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### PasswordReset

The PasswordReset mechanism is meant to provide a short lived token that is sent over email.  The mechanism was designed to work with a password reset application, like ScaleJS Password.  This mechanism requires a database.  No schema needs to be loaded into the database since OpenUnison will create the schema on startup
if the table doesn't exist.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: passwordreset
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.PasswordReset
  uri: "/auth/passwdReset"
  init:
    # Database driver
    driver: "com.mysql.jdbc.Driver"
    # JDBC URL
    url: "jdbc:mysql://dbs.tremolo.lan/myvddata"
    # DB User
    user: "myvd"
    # Maximum number of connections
    maxCons: "5"
    # Maximum number of connections not actively working
    maxIdleCons: "5"
    # The URI to redirect users to after being authenticated if the user's session is over
    passwordResetURI: "/echo/echo?roles=admin&myparam=value"
    # The number of minutes a key is valid
    minValidKey: "1"
    # SMTP Host
    smtpHost: "smtp.gmail.com"
    # SMTP port
    smtpPort: "587"
    # SMTP user
    smtpUser: "user@gmail.com"
    # Email with key subject line
    smtpSubject: "Password Reset"
    # Message for the password reset, ${key} for the user's key
    smtpMsg: "Click to reset: http://localhost.localdomain:9090/auth/pwdRest?${key}"
    # The email address for the "From"
    smtpFrom: "user@gmail.com"
    # Set to true if using TLS
    smtpTLS: "true"
    # Set to true to enable this mechanism
    enabled: "true"
    # The HibernateSQL dialect
    dialect: "org.hibernate.dialect.MySQLDialect"
    # Validation query to make sure the connection is still available
    validationQuery: "SELECT 1"
    # Optional - The mapping file (.hbx.xml) to use if the default mapping doesn't work, must be in the classpath
    hibernateConfig: ""
    # Optional - If true, create tables. After initial configuration, you may want to disable this depending on the dialect.
    hibernateCreateSchema: "true"
    # The name of the attribute to look users up by, defaults to "mail"
    uidAttributeName: "mail"
  secretParams:
  # SMTP Password
  - name: smtpPassword
    secretName: orchestra-secrets-source
    secretKey: smtpPassword
  # DB Password
  - name: password
    secretName: orchestra-secrets-source
    secretKey: jdbcPassword
```

##### Chain 

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: resetpassword
  namespace: openunison
spec:
  authMechs:
  - name: passwordreset
    required: required
    params:
      # The page to collect the user's email address
      emailCollectionRedir: "/auth/forms/pwdResetEmail.jsp"
      # The splash screen telling the user a reset message has been sent
      splashRedirect: "/auth/forms/pwdResetSplash.jsp"
      # Page to show the user when an email address can't be found
      noUserSplash: "/auth/forms/pwdResetNoUser.jsp"
      # Maximum number of times a request can be made with a URL
      maxChecks: "1"
    secretParams: []
  level: 1
  root: o=Tremolo
```

#### OpenIDConnectionAuthMech

This authentication mechanism is designed to support an OpenID Connect Provider (OP) such as Google, LinkedIn and KeyCloack (as with others).  This implementation
does not implement the full standard, only enough to allow authentication against an identity provider

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: oidc
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.proxy.auth.openidconnect.OpenIDConnectAuthMech
  uri: "/auth/oidc"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: keycloak
  namespace: openunison
spec:
  authMechs:
  - name: oidc
    required: required
    params:
      # Name of the request object the bearer token should be stored in
      bearerTokenName: "kcBearerToken"
      # The client_id supplied by the OP
      clientid: "unison"

      # What to request from the OP
      responseType: "code"
      
      # The identity provider's issuer
      issuer: https://login.microsoftonline.com/xxxxxxxxxxxxxxx

      # Only set these attributes if the issuer does not provide an oidc discovery document

      # The authentication request url of the OP
      #idpURL: "http://keycloak.tremolo.lan:8080/auth/realms/unison/protocol/openid-connect/auth"
      # The URL to load the token from
      #loadTokenURL: "http://keycloak.tremolo.lan:8080/auth/realms/unison/protocol/openid-connect/token"
      # set to true if the identity provider supports PKCE
      #usePKCE: "true"

      # Claims to request from the OP
      scope: "openid name offline"
      # If true, the mechanism will attempt to find the user inside of OpenUnison's internal virtual directory
      linkToDirectory: "true"
      # If a user isn't matched in the virtual directory, ou to be assigned to
      noMatchOU: "keycloak"
      # How to lookup a user. Use ${claim_name} to include a claim from the user's id_token in the search
      lookupFilter: "(uid=${preferred_username})"
      # If the user's object isn't matched in the virtual directory, the objectClass to assign
      defaultObjectClass: "inetOrgPerson"
      # The name of the attribute in the id_token that identifies the user
      uidAttr: "name"
      # Optional - If you want to limit which user domains to authenticate by the OP, specify that domain here
      hd: ""
      # Implementation of com.tremolosecurity.unison.proxy.auth.openidconnect.sdk.LoadUserData
      userLookupClassName: "com.tremolosecurity.unison.proxy.auth.openidconnect.loadUser.LoadJWTFromAccessToken"
      jwtTokenAttributeName: "id_token"
    secretParams:
    - name: secretid
      # the key in the secret's data section
      secretKey: OIDC_CLIENT_SECRET
      # the name of the secret
      secretName: orchestra-secrets-source#[openunison.static-secret.suffix]
  level: 1
  root: o=Tremolo
```

###### Loading Claims

Once the OP confirms authentication, claims must be loaded.  Most OPs support the id_token attribute in the authentication response.  How those claims are loaded is
based on the `com.tremolosecurity.unison.proxy.auth.openidconnect.sdk.LoadUserData` interface.  There are two implementations built into OpenUnison:

**com.tremolosecurity.unison.proxy.auth.openidconnect.loadUser.LoadJWTFromAccessToken**

This will load the id_token from the authentication response with the following parameters:

| Attribute              | Description                                                            | Example Value |
|------------------------|------------------------------------------------------------------------|---------------|
| jwtTokenAttributeName  | The name of the attribute in the authentication response that stores the id_token | id_token       |

**com.tremolosecurity.unison.proxy.auth.openidconnect.loadUser.LoadAttributesFromWS**

Calls a web service to load the claims using the bearer token from the authenticaiton process with the following parameters:

| Attribute | Description                          |
|-----------|--------------------------------------|
| restURL   | The URL to call to get the JWT       |

#### GenerateOIDCTokens

This mechanism will generate an OpenID Connect session on behalf of an authenticated user.  This is useful when using OpenUnison with Kubernetes to
authenticate users or proxy authentication with other identity providers and using OpenUnison's session specific client secrets with kubectl.  Use
this mechanism by adding it to the end of the chain to generate the proper session.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: genoidctoken
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.GenerateOIDCTokens
  uri: "/auth/oidctoken"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: formloginfilter
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
      uidAttr: "uid"
      uidIsFilter: "false"
    secretParams: []
  - name: genoidctoken
    required: required
    params:
      # The name of the OpenID Connect identity provider to use
      idpName: "oidc"
      # The name of the trust to generate a session for
      trustName: "kubernetes"
      # set to true if you don't want to tie the generated oidc token to the web session, 1.0.43+
      doNotTieSession: "true"
    secretParams: []
  level: 1
  root: "dc=domain,dc=com"
```

#### WebAuthn

The WebAuthn standard supports hardware and platform keys for auhentication.  Often refered to as "Passkeys".  In order to support webauthn, you'll need to create a workflow that can store credential IDs and the encrypted blob of user's keys.  Prior to using a webauthn enabled key, the key needs to be provisioned.  See [the provisioning webauthn keys]() for details on how to setup webauthn provisioning.

Here is an example of a workflow for updating webauthn keys on authentication:

```yaml
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: update-webauthn
  namespace: openunison
spec:
  description: webauthn update
  inList: false
  label: webauthn update
  orgId: 687da09f-8ec1-48ac-b035-f2f182b9bd1e
  dynamicConfiguration:
    dynamic: false
    className: ""
    params: []
  tasks: |-
    # load the user's groups to make sure they don't get lost in the sync process
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.LoadGroups
      params:
        nameAttr: uid
        inverse: "false"
    # load the current credential id
    - taskType: customTask
      className: com.tremolosecurity.provisioning.customTasks.LoadAttributes
      params:
        name: departmentNumber
        nameAttr: uid
    # provision the updated keys into the target
    - taskType: provision
      sync: true
      attributes:
      - uid
      - carLicense
```

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: webauthn
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.WebAuthn
  uri: "/auth/webauthn"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: webuthn
  namespace: openunison
spec:
  authMechs:
  - name: webauthn
    required: required
    params:
      attribute: "carLicense"
      encryptionKeyName: "sessionKey"
      uidAttributeName: "uid"
      formURI: "/auth/forms/webauthnAuthSingleStep.jsp"
      workflowName: "update-webauthn"
      allowPassword: "true"
      userVerificationRequirement: "discouraged"
      singleStep: "true"
      credentialIdAttributeName: "departmentNumber"
    secretParams: []
  level: 2
  root: o=Tremolo
```

#### GithubAuthMech

Provides authentication using GitHub.  Prior to setup create a new OAuth2 Application in GitHub.  This mechanism should be paired with the `AuthorizationAuthMech` mechanism to limit the teams and organizations that can authenticate the user.  GitHub provides no way to limit who can authenticate to an OAuth2 Application.   The user's organizations are stored in the attribute `githubOrgs` and all teams are stored in the attribute `githubTeams` in the form org/team.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: github
  namespace: openunison
spec:
  className: com.tremolosecurity.unison.proxy.auth.github.GithubAuthMech
  uri: "/auth/github"
  init: {}
  secretParams: []
```

##### Chain


```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: github
  namespace: openunison
spec:
  authMechs:
  - name: github
    required: required
    params:
      # Name in the session to store the token
      bearerTokenName: "githubToken"
      # The client ID from GitHub
      clientid: "xxxxxxxx"
      # Scopes to request, shouldn't change
      scope: "user:read user:email read:org"
      # Should the account be linked to the directory?
      linkToDirectory: "true"
      # If no account exists, what OU to use for DNs
      noMatchOU: "github"
      # Lookup filter for finding a user
      lookupFilter: "(uid=${login})"
      # The attribute that is the unique identifier, should not change
      uidAttr: "login"
      # The objectClass to use if no account can be found
      defaultObjectClass: "inetOrgPerson"
    secretParams:
    # The secret from GitHub
    - name: secretid
      secretName: orchestra-secrets-source
      secretKey: github-secret
  - name: az
    required: required
    params:
      rules: "custom;github!testingopenunison/team1"
    secretParams: []
  level: 20
  root: o=Tremolo
```

#### AuthorizationAuthMech

This mechanism provides an authorization check prior to completing a chain.  This can be important when you want to limit who can authenticate when integrating an external provider without that remove provider providing a way to limit which users can be authenticated.

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: az
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.AuthorizationAuthMech
  uri: "/auth/az"
  init: {}
  secretParams: []
```

##### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: github
  namespace: openunison
spec:
  authMechs:
  - name: github
    required: required
    params:
      # Name in the session to store the token
      bearerTokenName: "githubToken"
      # The client ID from GitHub
      clientid: "xxxxxxxx"
      # Scopes to request, shouldn't change
      scope: "user:read user:email read:org"
      # Should the account be linked to the directory?
      linkToDirectory: "true"
      # If no account exists, what OU to use for DNs
      noMatchOU: "github"
      # Lookup filter for finding a user
      lookupFilter: "(uid=${login})"
      # The attribute that is the unique identifier, should not change
      uidAttr: "login"
      # The objectClass to use if no account can be found
      defaultObjectClass: "inetOrgPerson"
    secretParams:
    # The secret from GitHub
    - name: secretid
      secretName: orchestra-secrets-source
      secretKey: github-secret
  - name: az
    required: required
    params:
      # Multiple rules can be listed, any need to be successful to be satisfied. Each value is first the constraint type, then a semi-colon, then the constraint
      rules: "custom;github!testingopenunison/team1"
    secretParams: []
  level: 20
  root: o=Tremolo
```

#### DuoSecLogin

Add DUO authentication.  This mechanism must be *AFTER* a mechanism that will load the user's identifier.  

##### Mechanism

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: duo
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.DuoSecLogin
  uri: "/auth/duo"
  init: {}
  secretParams: []
```

#### Chain

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: duoChain
  namespace: openunison
spec:
  authMechs:
  - name: login-form
    required: required
    params:
      FORMLOGIN_JSP: "/auth/forms/defaultForm.jsp"
      uidAttr: "uid"
    secretParams: []
  - name: duo
    required: required
    params:
      # DUO API Host
      duoApiHostName: "api-bb81f354.duosecurity.com"
      # user id attribute
      userNameAttribute: "uid"
    secretParams:
    # duo integration key
    - name: duoIntegrationKey
      secretName: orchestra-secrets-source
      secretKey: duoIntegrationKey
    # duo secret key
    - name: duoSecretKey
      secretName: orchestra-secrets-source
      secretKey: duoSecretKey
    # duo authentication key
    - name: duoAKey
      secretName: orchestra-secrets-source
      secretKey: duoAKey
  level: 2
  root: o=Tremolo
```

#### FullMappingAuthMech

Map attributes on the authentication process' attributes.

| Source Type | Description                                                                                          | Source                                | Example                                |
|-------------|------------------------------------------------------------------------------------------------------|---------------------------------------|----------------------------------------|
| user        | Map an attribute from the user’s directory object                                                   | Name of an attribute                  | givenName                              |
| static      | A static value that doesn’t change                                                                  | The static value                      | Myvalue                                |
| custom      | A class that is used to determine the mapping                                                       | Class name, see the SDK for details on how to implement | com.mycompany.mapper.Mapper |
| composite   | A composite of attributes and static values. Attributes are defined with `${attributename}`. Only attributes that exist before the mappings are run are available | Static and attribute data             | `${givenName}.${sn}@mydomain.com`      |

##### Mechanism 

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationMechanism
metadata:
  name: map
  namespace: openunison
spec:
  className: com.tremolosecurity.proxy.auth.FullMappingAuthMech
  uri: "/auth/map"
  init: {}
  secretParams: []
```

##### Chain

```yaml
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: enterprise-idp
  namespace: openunison
spec:
  authMechs:
  # initial authentication chain
  - name: oidc
    params:
      bearerTokenName: oidcBearerToken
      defaultObjectClass: inetOrgPerson
      forceAuthentication: "true"
      hd: ""
      issuer: https://login.microsoftonline.com/61cbe426-d3ca-4ebd-8ca4-e47e354a85bb
      linkToDirectory: "false"
      lookupFilter: (uid=${sub})
      noMatchOU: oidc
      responseType: code
      scope: openid email profile
      uidAttr: sub
      userLookupClassName: com.tremolosecurity.unison.proxy.auth.openidconnect.loadUser.LoadAttributesFromWS
    required: required
    secretParams:
    - name: secretid
      secretKey: OIDC_CLIENT_SECRET
      secretName: orchestra-secrets-source#[openunison.static-secret.suffix]
    - name: clientid
      secretKey: OIDC_CLIENT_ID
      secretName: orchestra-secrets-source
  # FullMapping
  - name: map
    params:
      map:
      # target attribute|type|source
      - uid|composite|${upn}
      - mail|composite|${email}
      - givenName|composite|${given_name}
      - sn|composite|${family_name}
      - displayName|composite|${name}
      - memberOf|user|roles
    required: required
```