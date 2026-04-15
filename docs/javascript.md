# Customizing OpenUnison with JavaScript

Nearly every component of OpenUnison can be customized with JavaScript to create a highly customized approach to integrating identity into your environment. You can use JavaScript to write custom:

* [Application Filters](#application-filters)
* [Authentication Mechanisms](#authentication-mechanisms)
* [Authorization Rules](#authorization-rules)
* [Attribute Mappers](#authorization-rules)
* [Workflow Tasks](#workflow-tasks)
* [Attribute Lookups & Validators](#attribute-lookups-validators)
* [Jobs](#jobs)
* [Events](#events)

While each customization point has its own specific implementation details, all of them share common libraries and constraints. 

## Reusing JavaScript Code

***Available in 1.0.47***

To make it easier to have shared JavaScript functions that can be re-used across multiple components, create a `JavaScript` object that can be re-used from any JavaScript based customization:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: JavaScript
metadata:
  namespace: openunison
  name: shared-functions
spec:
  javascript: |-
    // this function can be called from any JavaScript customization that references this JavaScript object
    function getdata() {
      return 1;
    }
```

## Object Implementations

JavaScript customizations for each of OpenUnison's object types is implemented as a specialized customization of that object.  This section will provide a skeleton for each kind of object that can be customized to get you started.

### Application Filters

The `Application` object contains `urls` that each can have a `filter`.  Each of these filters gives OpenUnison the chance to manipulate headers and cookies.  You can also use a filter to provide an API for your environment.  This API can work with any of the resources OpenUnison is integrated with.  

```yaml
---
apiVersion: openunison.tremolo.io/v2
kind: Application
metadata:
  name: my-app
  namespace: openunison
spec:
  azTimeoutMillis: 3000
  cookieConfig:
    cookiesEnabled: false
    domain: 'localhost.localdomain'
    httpOnly: true
    keyAlias: session-tremolosession
    logoutURI: /logout
    scope: -1
    secure: true
    sessionCookieName: tremolosession
    timeout: 900
  isApp: true
  urls:
  - azRules:
    - constraint: (objectClass=*)
      scope: filter
    filterChain:
    // specify a JavaScript filter
    - className: com.tremolosecurity.proxy.filters.JavaScriptFilter
      params:
        # list of JavaScript objects that should be accessible
        includeJs:
        - shared-functions
        # the JavaScript
        javaScript: |-
          // useful for initializing the filter
          function initFilter(config) {

          }

          
          // do the work of the filter
          function doFilter(request,response,chain) {
            // create a response
            resp = {};
            // call the getdata() function from the JavaScript included from shared-functions
            resp["data"] = getdata();
            // write the data back to the client
            response.getWriter().println(JSON.stringify(resp));
          }

    authChain: anon
    hosts:
    - 'localhost.localdomain'
    
    results:
      auFail: default-login-failure
      azFail: default-login-failure
    uri: /api/jstest
```

The above JavaScript has comments specific for creating an API.  The [Applications](/documentation/configuring-openunison/#applications) section for configuring OpenUnison provides details for all of the various options.

### Authentication Mechanisms

Customize Authentication Mechanisms are often a good place to build custom mappings that can't be handled by one of OpenUnison's built in mechanisms.  For instance, you can do a lookup of a remote API to add groups.  

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: AuthenticationChain
metadata:
  name: jsauth
  namespace: openunison
spec:
  authMechs:
  - name: js
    params:
      # list of JavaScript objects to include for shared functions
      includeJs:
      - shared-functions
      js: |-
        // setup classes that we can use from Java
            Attribute = Java.type("com.tremolosecurity.saml.Attribute");
            ProxyConstants = Java.type("com.tremolosecurity.proxy.util.ProxyConstants");
            GlobalEntries = Java.type("com.tremolosecurity.server.GlobalEntries");
            HashMap = Java.type("java.util.HashMap");
            ArrayList = Java.type("java.util.ArrayList");
            System = Java.type("java.lang.System");
            AuthUtil = Java.type("com.tremolosecurity.proxy.auth.util.AuthUtil");
            AuthInfo = Java.type("com.tremolosecurity.proxy.auth.AuthInfo");

        function doAuth(request,response,as) {
          // call a shared function
          getvalue();

          
          
          // get the session data needed
          var session = request.getSession();
          var holder = request.getAttribute(ProxyConstants.AUTOIDM_CFG);

          var ac = request.getSession().getAttribute(ProxyConstants.AUTH_CTL);
          var act = holder.getConfig().getAuthChains().get(ac.getHolder().getAuthChainName());

          // lokup a user
          myvd = GlobalEntries.getGlobalEntries().getConfigManager().getMyVD();
          res = myvd.search("o=Tremolo",2,"(uid=testsaml2)",new ArrayList());
          res.hasMore();
          entry = res.next();
          while (res.hasMore()) res.next();

          // create a new AuthInfo object to represent the user
          authInfo = new AuthInfo(entry.getDN(),session.getAttribute(ProxyConstants.AUTH_MECH_NAME),act.getName(),act.getLevel(),null);
          session.getAttribute(ProxyConstants.AUTH_CTL).setAuthInfo(authInfo);
        
          // you could also get the current authInfo and make updates
          // authInfo = session.getAttribute(ProxyConstants.AUTH_CTL).getAuthInfo();

          it = entry.getAttributeSet().iterator();

          while (it.hasNext()) {
            attrib = it.next();
            attr = new Attribute(attrib.getName());
            vals = attrib.getStringValueArray();
            for (var i=0;i<vals.length;i++) {
              attr.getValues().add(vals[i]);
            }
            authInfo.getAttribs().put(attr.getName(), attr);
          }
          

          // mark that the process is complete and can move to the next mechanism in the chain
          as.setExecuted(true);
          as.setSuccess(true);
          holder.getConfig().getAuthManager().nextAuth(request, response,session,false);

        }
    required: required
  level: 1
  root: o=Tremolo
```

This custom mechanism hard codes our authentication to the user `testsaml2`.  First it gets the session information, then does a user lookup, next creates a new `AuthInfo` object that represents the user, and finally tells OpenUnison the authentication is complete.

### Authorization Rules

A custom authorization can be used from either an `Application`'s `azRules` or as an approval step in a workflow's list of approvers.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: CustomAuthorization
metadata:
  name: cluster-admin
  namespace: openunison
  annotations:
    argocd.argoproj.io/sync-wave: "30"
spec:
  className: com.tremolosecurity.customaz.JavaScriptAz
  params:
    # list of JavaScript objects to include for shared functions
    includeJs:
    - shared-functions
    javaScript: |-
        GlobalEntries = Java.type("com.tremolosecurity.server.GlobalEntries");

        // potentially initialize the rule
        function init(config) {
            // No initialization needed
        }

        // the subject is the user
        // params is the list of values in the `azRule` after the name of the rule
        function isAuthorized(subject,params) {
            var groups = subject.getAttribs().get('groups');
            if (groups == null || groups.getValues().size() == 0) {
                return false;
            }
            for (var i = 0; i < groups.getValues().size(); i++) {
                if ((groups.getValues().get(i).startsWith('k8s-cluster-k8s') && ((groups.getValues().get(i).endsWith('-administrators-internal') || (groups.getValues().get(i).endsWith('-administrators-external') ) )))) {
                    return true;
                }
            }
            return false;
        }

        // if this rule is being used for approvers in a workflow
        // return a list of distinguished names
        function listPossibleApprovers(params) {
            ArrayList = Java.type("java.util.ArrayList");
            var approvers = new ArrayList();
            
            return approvers;
        }

```

The primary function for most use cases will be `isAuthorized` and must return a true or false value.  If using a custom rule in a workflow approval, the `listPossibleapprovers` must return a `List` of distinguished names..

### Attribute Mappers

Heres an example from the [Custom SSO](/documentation/custom-sso/#creating-an-openid-connect-identity-provider) section of OpenUnison's documentation:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: JavaScriptMapping
metadata:
  namespace: openunison
  name: argocd-groups
spec:
  # list of JavaScript objects to include for shared functions
  includeJs:
  - shared-functions
  javascript: |-
    // user is an instance of com.tremolosecurity.provisioning.core.User
    function doMapping(user,name) {
      Attribute = Java.type("com.tremolosecurity.saml.Attribute");

      // get the current groups from the user
      var groups = user.getAttribs().get("groups").getValues()

      // list of simple group names
      simpleGroups = new Attribute(name);


      // convert groups from LDAP DNs from Active Directory to standard group names
      for (var i=0;i<groups.length;i++) {
        var group = groups.get(i);

        // remove everything before the first '=' and after the first comma ','
        // ex cn=my-group,dc=domain,dc=com --> my-group
        var simpleGroupName = group.substring(group.indexOf('=') + 1, group.indexOf(','));

        // we only want to send argocd groups
        if (simpleGroupName.startsWith('argocd-')) {
          simpleGroups.getValues().add(simpleGroupName);
        }
      }



      return simpleGroups;
    }
```

The user object (an instance of `com.tremolosecurity.provisioning.core.User`) is passed in, with all existing attributes along with the name of the attribute that is expected.  


### Workflow Tasks

You can use JavaScript in workflow tasks to customize onboarding.  For instance, from the Argo Workflows JIT integration:

```yaml
  # two tasks:
  # 1. map the user's sub to something that will work as a kubernetes object name
  # 2. map the user's groups to (Cluster)RoleBindings
  - taskType: customTask
    className: com.tremolosecurity.provisioning.customTasks.JavaScriptTask
    params:
      # list of JavaScript objects to include for shared functions
      includeJs:
      - shared-functions
      javaScript: |-
        HashMap = Java.type("java.util.HashMap");
        OpenShiftTarget = Java.type("com.tremolosecurity.unison.openshiftv3.OpenShiftTarget");
        Attribute = Java.type("com.tremolosecurity.saml.Attribute");
        K8sUtils = Java.type("com.tremolosecurity.k8s.util.K8sUtils");
        System = Java.type("java.lang.System");

        function init(task,params) {
        // nothing to do
        }

        function reInit(task) {
        // do nothing
        }

        function doTask(user,request) {
            // map the user to a dns compliant name
            var saname = OpenShiftTarget.sub2uid(user.getAttribs().get("sub").getValues().get(0));
            
            request.put("saname", saname);
            user.getAttribs().put("saname",new Attribute("saname",saname));
            request.put("sub",user.getAttribs().get("sub").getValues().get(0));

            // load the ConfigMap that stores mappings from groups to (Cluster)RoleBindings
            var group2bindings = JSON.parse(K8sUtils.loadConfigMap("k8s","openunison","argowf-groups2bindings").get("mappings"));
            var bindings = new Attribute("bindings");
            var memberOf = user.getAttribs().get("groups");
            for (var i = 0;i < memberOf.getValues().size();i++) {
              var group = memberOf.getValues().get(i);
              System.out.println("group:" + group);
              var binding = group2bindings[group];
              
              if (binding != null && binding != "") {
                // there's a binding, map to json
                
                
                if (binding["kind"] == "crb") {
                  // a ClusterRoleBinding doesn't have a namespace
                  bindings.getValues().add("crb:" +  binding["name"]);
                } else if (binding["kind"] == "rb") {
                  // RoleBindings require a namespace
                  bindings.getValues().add("rb:" +  binding["namespace"] + ":" + binding["name"]);
                } // else, ignore
              }
            }

            // add the attribute, we'll map into groups later
            user.getAttribs().put("bindings",bindings);

            return true;
        }

```

This task shows many of the utilities mentioned later in this page that make it easier to integrate with a cluster's API to manage information.

### Jobs

Scheduled tasks can be built in JavaScript.  The below example is from the privileged access charts.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: OUJob
metadata:
  name: clear-privileged-sessions
  namespace: openunison
spec:
  cronSchedule:
    seconds: "0"
    minutes: "*"
    hours: "*"
    dayOfMonth: "*"
    month: "*"
    dayOfWeek: "?"
    year: "*"
  className: com.tremolosecurity.provisioning.scheduler.jobs.JavaScriptJob
  group: sessions
  params:
  # list of JavaScript objects to include for shared functions, can be listed multiple times
  - name: includeJs
    value: shared-functions
  - name: javaScript
    value: |-
      OpenShiftTarget = Java.type("com.tremolosecurity.unison.openshiftv3.OpenShiftTarget");
      ProvisioningParams = Java.type("com.tremolosecurity.provisioning.core.ProvisioningParams");
      System = Java.type("java.lang.System");
      User = Java.type("com.tremolosecurity.provisioning.core.User");
      
      TremoloUser = Java.type("com.tremolosecurity.provisioning.service.util.TremoloUser");
      Attribute = Java.type("com.tremolosecurity.saml.Attribute");
      HashSet = Java.type("java.util.HashSet");
      HashMap = Java.type("java.util.HashMap");
      WFCall = Java.type("com.tremolosecurity.provisioning.service.util.WFCall");

      ServiceActions = Java.type("com.tremolosecurity.provisioning.service.util.ServiceActions");
      GlobalEntries = Java.type("com.tremolosecurity.server.GlobalEntries");
      DateTime = Java.type("org.joda.time.DateTime");
      ArrayList = Java.type("java.util.ArrayList");

      PreparedStatement = Java.type("java.sql.PreparedStatement");
      ResultSet = Java.type("java.sql.ResultSet");

      FilterBuilder = Java.type("org.apache.directory.ldap.client.api.search.FilterBuilder");

      Gson = Java.type("com.google.gson.Gson");
      JSUtils = Java.type("com.tremolosecurity.util.JSUtils");
      Base64 = Java.type("java.util.Base64");
      EncryptedMessage = Java.type("com.tremolosecurity.provisioning.util.EncryptedMessage");

      DateUtils = Java.type("org.apache.directory.api.util.DateUtils");
      DateTime = Java.type("org.joda.time.DateTime");
      DateTimeZone = Java.type("org.joda.time.DateTimeZone");

      Long = Java.type("java.lang.Long");
      Logger = Java.type("org.apache.log4j.Logger");
      String = Java.type("java.lang.String");

      // run on the schedule
      function execute(configManager,context) {
        System = Java.type("java.lang.System");
        System.out.println("here in clear-privileged-sessions");

        

        db = GlobalEntries.getGlobalEntries().getConfigManager().getProvisioningEngine().getTarget("jitdb").getProvider().getDS().getConnection();
        k8s = GlobalEntries.getGlobalEntries().getConfigManager().getProvisioningEngine().getTarget("k8s").getProvider();
        con = k8s.createClient();
        try {

          var privGroups = db.createStatement().executeQuery("select name from localGroups where name like '%-privileged-%'");

        
          while (privGroups.next()) {
            groupName = privGroups.getString("name");
            System.out.println("Group name: '" + groupName + "'" );
            checkForGroup(groupName,k8s,con);
          }

        } catch (ex) {
          System.out.println(ex);
        } finally {
          if (db != null) {
            db.close();
          }

          if (con != null) {
              con.getHttp().close();
              con.getBcm().close();
          }
        }
        
        

       

        
        
      }
```

### Events

JavaScript based event listeners can be created to react to asynchronous events in OpenUnison:

```yaml
apiVersion: openunison.tremolo.io/v1
kind: MessageListener
metadata:
    name: jslistener
    namespace: openunison
spec:
    className: com.tremolosecurity.provisioning.listeners.JSListener
    params:
      # list of JavaScript objects to include for shared functions, can be listed multiple times
      - name: includeJs
        value: shared-functions
      - name: javaScript
        value: |-
            function onMessage(cfg, payload, msg) {
                JSUtils = Java.type("com.tremolosecurity.util.JSUtils");
                GlobalEntries = Java.type("com.tremolosecurity.server.GlobalEntries");
                System = Java.type("java.lang.System");
                HashMap = Java.type("java.util.HashMap");
                Gson = Java.type("com.google.gson.Gson");

                System.out.println("in message listener!!!!");
                var syncToDR = "";

                try {
                    syncToDR = JSON.parse(payload);
                } catch (error) {
                    payload = payload.replace(/\\"/g, '"').replace(/\\\\/g, '\\');
                    System.out.println("cleaned payload:" + payload);
                    syncToDR = JSON.parse(payload);
                }

                

            }



## Accessing OpenUnison Classes

OpenUnison is built on Java, so every class provided by OpenUnison is available to your JavaScript.  All of the [JavaDocs are available](/documentation/javadocs/) for each version of OpenUnison.  To use a Java class from within your JavaScript, use the `Java.type()` function.  For instance, to use the class `java.lang.System`:

```java
System = Java.type("java.lang.System");
System.out.println("Hello world!!!");
```

To get more details of using Java within your JavaScript, see [GraalJS' Java Documentation](https://www.graalvm.org/latest/reference-manual/js/JavaInteroperability/#access-java-from-javascript).

In addition to GraalJS' access to Java, OpenUnison includes several utility classes to make life easier when developing customizations.

## String Utilities

Working with Kubernetes often involves Base64 encoding and decoding strings.  This can get frustrating with JavaScript because GraalJS uses JavaScript native strings and makes it very hard to use the Java based Base64 utilities to get raw UTF-8 encodings.  To make it easier to work with we provided the `com.tremolosecurity.util.JSUtils` class to make it easier to work with string conversions.  Below are examples:

### Base64 Decode

It's common to need to base64 decoded data, especially certificates and `Secrets`:

```javascript
// Create a reference to the JSUtils type for access
JSUtils = Java.type("com.tremolosecurity.util.JSUtils");
// Retrieve a Secret from the Kubernetes API server
var jsonData = k8s.callWS(k8s.getAuthToken(), con, secretUrl);
// Parse the result from a string to json
var secret = JSON.parse(jsonData);
// decoded a key from the Secret from a base64 string to a decoded string
var decoded = JSUtils.base64Decode(secret.data.key);
```

### String <-> UTF-8 Bytes

If you need to encoded or decode a string, these methods make it easier to do so.

### string2bytes

Generates an array of UTF-8 bytes useful with Java Base64 classes:

```javascript
// generate a string from json
strIncidentInfo = JSON.stringify(incidentInfo);
// encrypt the string
var ecnryptedIncidentInfo = GlobalEntries.getGlobalEntries().getConfigManager().getProvisioningEngine().encryptObject(strIncidentInfo);
gson = new Gson();
// generate a string from the JSON encrypted object
var encIncInfoStr = gson.toJson(ecnryptedIncidentInfo);
// generate an array of UTF-8 encoded bytes
var incBytes = JSUtils.string2bytes(encIncInfoStr);
// Base64 encode the array
var b64Inc = Base64.getEncoder().encodeToString(incBytes);
```

### bytes2string

Similar to `string2bytes`, but in reverse.  This method will generate a `String` from an array of UTF-8 bytes:

```javascript
// load data from a request
userData = (request.getSession().getAttribute(ProxyConstants.AUTH_CTL)).getAuthInfo();
// get raw binary array, decode to a string
var payload = JSUtils.bytes2string(request.getAttribute(ProxySys.MSG_BODY));
// parse JSON
var payloadJson = JSON.parse(payload);
var incident = payloadJson.attributes.incident;
```

## Working with Kubernetes

It's common to need to interact with one or more Kubernetes API servers in your customizations.  The common steps to working with Kubernetes are to:

1. Retrieve the provisioning `Target` that holds the configuration and security information for your cluster
2. Create an HTTP connection
3. Call APIs
4. Close the connection

Everything is done use raw HTTP calls.  You don't need to worry about credentials, tokens, or certificates.  Everything is handled for you by OpenUnison.  For instance, to retrieve a `Secret` using raw HTTPS calls:

```javascript
// retrieve the target for the local cluster.  The cluster you're running on is always called k8s
var k8s = GlobalEntries.getGlobalEntries().getConfigManager().getProvisioningEngine().getTarget("k8s").getProvider();
// create a connection, this manages all of your credentials and certificates
var con = k8s.createClient();
// encompass your work in a try to gracefully shutdown
try {
    // construct the URL to retrieve our Secret
    var secretUrl = "/api/v1/namespaces/openunison/secrets/googlews";
    // Retrieve a Secret from the Kubernetes API server
    var jsonData = k8s.callWS(k8s.getAuthToken(), con, secretUrl);
    // Parse the result from a string to json
    var secret = JSON.parse(jsonData);
    
} finally {
    // gracefully shutdown the connection
    if (con != null) {
        con.getHttp().close();
        con.getBcm().close()
    }
}
```

In addition to calling individual APIs using the HTTPS `GET` method, you can also perform `POST`, `PUT`, and `PATCH` methods using the `com.tremolosecurity.unison.openshiftv3.OpenShiftTarget`.  Take a look at the JavaDocs for your version of OpenUnison for the latest available methods.

### Kubernetes Utilities

If all you need is to load `Secrets` and `ConfigMaps` the `com.tremolosecurity.k8s.util.K8sUtils` is a simpler way to do that.  For example, to load a `ConfigMap`:

```javascript
// load a reference to the K8sUtils class
K8sUtils = Java.type("com.tremolosecurity.k8s.util.K8sUtils");
// Load a HashMap of the data elements from the resource-default ConfigMap in the openunison Namespace in the local cluster
defaultsConfigMap = K8sUtils.loadConfigMap("k8s","openunison","resource-defaults");
// retrieve the data element istio.annotation from the retrieved ConfigMap
currentIstio = defaultsConfigMap.get("istio.annotation");
```

If you want to load a `Secret`:

```javascript
// load a reference to the K8sUtils class
K8sUtils = Java.type("com.tremolosecurity.k8s.util.K8sUtils");
// Load a HashMap of the base64 decoded data elements from the googlews Secret in the openunison Namespace in the local cluster
googlewsSecret = K8sUtils.loadSecret("k8s","openunison","googlews");
// there's no need to base64 decode the data:
pem = googlewsSecret.get("pem");
```

## Calling External Services

One of OpenUnison's strengths is that it's capable of working with services outside of Kubernetes!  For example, you could use a [GitHub App](/applications/github/#provisioning-access-and-resources-to-github) to create resources in GitHub, provision projects to [GitLab](/applications/gitlab), or work with [AzureAD / EntraID](/applications/entraid) to lookup data or provision access.

In addition to the applications specifically called out in the **Applications** section of the documentation, you can also look at the [Targets](/documentation/reference/targets/) provided by OpenUnison.  `Targets` provide a framework for interacting with remote services so your code doesn't need details.

## Calling HTTP Services

If you're interacting with a Kubernetes API server and need to interact with other remote services, you can re-use the http connection used with Kubernetes. The user's token isn't injected into the request until you call one of the `OpenShiftv3` target's methods, so you won't leak a bearer token.  It uses the [Apache HTTP Client 4.x](https://hc.apache.org/httpcomponents-client-4.5.x/index.html) libraries.  As an example, to call a remote api using an HTTP `Post`:

```javascript
//assuming you've already retrieved a connection from the local Kubernetes cluster and are inside of a try/finally block
//create a POST object
var httpPost = new HttpPost("https://oauth2.googleapis.com/token");
// generate POST data
params = new ArrayList();
params.add(new BasicNameValuePair("grant_type", "urn:ietf:params:oauth:grant-type:jwt-bearer"));
params.add(new BasicNameValuePair("assertion", googleJwt));
// set the post's entity data to what we want
httpPost.setEntity(new UrlEncodedFormEntity(params));
// run the post
var resp = con.getHttp().execute(httpPost);
```

If you're not already using a Kubernetes API integration in whatever code you're writing, you can use the [Apache HTTP Client 4.x](https://hc.apache.org/httpcomponents-client-4.5.x/index.html) libraries directly or you can use [Java's built in HTTP client](https://openjdk.org/groups/net/httpclient/intro.html).

Finally, if you're accessing a remote service that uses a certificate that is self signed or signed by a self-signed CA, you can add the CA or certificate to the `trusted_certs` section of your helm values and it will be trusted by OpenUnison.  There's no need to manually trust certificates in your code.

## Finding Examples

Most of the various components that can be customized via JavaScript have existing examples to work off of.  The best places to look are:

* Tremolo Security's Blog, [The Whammy Bar](https://www.tremolo.io/blog/bloghome)
* The [OpenUnison GitHub Organization](https://github.com/openunison)
* The [Tremolo Security GitHub organization](https://github.com/TremoloSecurity)
* The [Kubernetes: An Enterprise Guide, 3rd Ed GitHub Repository](https://github.com/PacktPublishing/Kubernetes-An-Enterprise-Guide-Third-Edition)

