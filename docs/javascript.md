# Customizing OpenUnison with JavaScript

Nearly every component of OpenUnison can be customized with JavaScript to create a highly customized approach to integrating identity into your environment. You can use JavaScript to write custom:

* Application Filters
* Authentication Mechanisms
* Authorization Rules
* Attribute Mappers
* Results
* Workflow Tasks
* Attribute Lookups & Validators
* Jobs
* Events

While each customization point has its own specific implementation details, all of them share common libraries and constraints. 

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

