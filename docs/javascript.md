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

If you need to encoded or decode a string, these methods make it easier to do so:

*TODO: example*

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

## Finding Examples

Most of the various components that can be customized via JavaScript have existing examples to work off of.  The best places to look are:

* Tremolo Security's Blog, [The Whammy Bar](https://www.tremolo.io/blog/bloghome)
* The [OpenUnison GitHub Organization](https://github.com/openunison)
* The [Tremolo Security GitHub organization](https://github.com/TremoloSecurity)
* The [Kubernetes: An Enterprise Guide, 3rd Ed GitHub Repository](https://github.com/PacktPublishing/Kubernetes-An-Enterprise-Guide-Third-Edition)