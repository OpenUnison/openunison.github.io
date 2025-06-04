# Privileged Access Management

Privileged Access Management, or Zero Standing Privileges, exposes administrator access to a cluster only during approved windows.  The goal of privileged access management is to limit the scope of a breach by ensuring that lost credentials can only be used to compromise a system during limited times.

OpenUnison's built in provisioning, deprovisioning, and scheduling capabilities make it an ideal system for managing privileged access in your Kubernetes clusters.  Before diving into the implementation specifics, let's cover an overview of how privileged access typically works.

## Managing Privileged Access in the Enterprise

The below diagram shows a typical enterprise privileged access system.  Access is granted during an "incident", which is logged in an incident management system.  An "incident" doesn't necessarily mean something bad, it can be scheduled maintenance or a rollout of a new system.  The point of an incident management system is to track that the incident has gone through the enterprise's compliance process for approval. 

Once an incident is created, the next step is to unlock a credential for the user that's going to do the work.  This is often a separate account that is bound to the user, but they only know the password during approved windows.  The job of unlocking this account and providing the credentials to the user securely is often done by a Privileged Access Manager, or PAM.  At this point the user is able to login to their cluster, in this case with OpenUnison, and do their work.  After logging in to OpenUnison they get a token that maps a group to a Kubernetes `ClusterRoleBinding` with `cluster-admin` access.  Once the incident's window is completed, the PAM locks the account.

![Generic privileged access scenario for Kubernetes](/assets/images/privileged-access/privileged-access-generic.png "Generic Privileged Access Management with Kubernetes")

This approach works well when you have a single cluster to manage.  What if you have multiple clusters that you could potentially have access to with the same privileged account?  

![Privileged access across multiple Kubernetes clusters](/assets/images/privileged-access/privilege-blast-radius.png "Privileged access across multiple Kubernetes clusters")

Once our privileged account is unlocked, it has access to not only the cluster listed in our incident, but also the other clusters I'm responsible for managing.  This leads to both technical and non-technical risks.  The largest non-technical risk is that you've broken your compliance framework because you're allowing privileged access to clusters that aren't part of an incident.  As for technical risks:

1. Lost credentials can be used against against clusters outside of the clusters named in your incident.  While time boxing the incident and credentials helps mitigate this risk, an attacker who has compromised the administrator's workstation could still compromise clusters outside of the incident.
2. An administrator could accidentally act on a cluster outside the incident.  If their credentials are unlocked across all possible clusters, a mistake could cause impacts to production systems.

Managing these risks can be difficult in Kubernetes given the disconnected nature of authorization between a `ClusterRoleBinding` be stored in-cluster and the group being managed in an identity provider.  We'll next walk through how OpenUnison makes this easier.

## Privileged Access with OpenUnison

OpenUnison simplifies privileged access management in Kubernetes by using it's built in identity management capabilities to:

1. Create a group separate from the identity provider's group for `cluster-admin` access.
2. When an incident is approved, a workflow adds the appropriate user to the correct group in a way that can be tracked both via SQL queries and inside of the SIEM tracking OpenUnison's logs, allowing the user to access the appropriate cluster.
3. When the incident is over, the user's access is removed and their sessions are destroyed, forcing them to re-login.
4. If it's found a user has access, but doesn't have a corresponding incident, then their access is removed and their sessions destroyed, requiring them to re-login.

The difficult part in automating this process is how to determine if there is an incident, which clusters the incident applies to, and how long the incident should last.  This is tough because incident management is generally a business process that is managed outside of standard Kubernetes workflows.  It will be unique to each enterprise. 

To make it easier to integrate OpenUnison with your incident management system, we've given you multiple options for triggering an incident:

* ***Manual Incident Approval*** - Instead of hooking your OpenUnison into an incident management system, you can specify the information manually and provide a list of approvers.  This provides the least amount of automation and integration, but allows you to get started quickly.

* ***Integrated*** - Almost anything in OpenUnison can be customized using JavaScript.  OpenUnison will provide you the base line and the form, as well as the area to create the customization needed to connect to your incident management system.  All of the boilerplate and identity work is done for you, you just need to provide the incident information.

Next, we'll walk through deploying both options.

### Deploying the Manual Incident Approval

If a user is an authorized cluster administrator, the manual incident approval mode deploys a form that asks for:

1. ***Cluster*** - One of any of the clusters the user is already authorized to be a `cluster-admin` in.
2. ***Window expiration for the incident*** - When the user's privileged access is set to expire.
3. ***An incident name or number*** - The name or the label of your incident that an approver can track back to the incident management system.
4. ***A task name or number***  - A task that can be tracked back to the incident management system.

![Manual authorization privileged access form](/assets/images/privileged-access/manual-az-1.png)

Once submitted, a list of approvers will get the change to review and approve the access request.  While this won't integrate with your existing workflow and relies on the approvers responding in a timely manner, it can get you running quickly.

The first step is to run [OpenUnison NaaS deployment](/namespace_as_a_service/).  You don't need to run a full OpenUnison NaaS per cluster, you can integrate multiple clusters into a control plane to simplify rollout.

Once the NaaS is deployed, the next step is to make some updates to OpenUnison's values.yaml.  The first is to set `openunison.naas.groups.privilegedAccessGroup` to `k8s-cluster-k8s-privileged-internal`.  This tells OpenUnison not to create a `ClusterRoleBinding` for the cluster's administrator groups (internal or external).  

The next setting is to tell OpenUnison how to populate the form.  Add the following under `openunison.naas.forms`:

```yaml
openunison:
  naas:
    forms:
      privileged_access:
        # needed for additional javascdript in the OpenUnison frontend
        includeExtraJs: true
        # Should be false unless testing an integration
        test_mode: false
        # The name of the attribute that identifies users, should only be changed if your LDAP directory is configured for a different lookup attribute
        uid_attribute: uid
        # configuration specific to the manual approval process
        manual_approval:
          # true if you want to use manual approvals
          enabled: true
          # a list of OpenUnison groups (not from your identity provider)
          groups:
          - priv-approval-internal
          - priv-approval-external
        # minimal js needed to tell OpenUnison about the incident
        submission_javascript: |-
          function validate_submission(newUser, errors, userData) {
            // create an object that represents an incident
            var incident = {
              "id": newUser.getAttributes().get("incident"),
              "task": newUser.getAttributes().get("taskid"),
              "expires": newUser.getAttributes().get("expiry")
            };

            // serialize the data to a string, store in the globals map
            globals.put("incident_info",JSON.stringify(incident));

            // no errors, return null
            return null;
          }
        # list of attributes to display on the form
        additional_attributes:
        - name: expiry
          type: text
          displayName: Window Expiration (YYYY-MM-DDTHH:MMZ)
          regEx: "^\\\\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\\\\d|3[01])T([01]\\\\d|2[0-3]):[0-5]\\\\d(Z|[+-]([01]\\\\d|2[0-3]):[0-5]\\\\d)$"
          regExFailedMsg: "Must be in the correct format"
          minChars: 0
          maxChars: 0
          unique: false
        - name: incident
          type: text
          displayName: Incident
          regEx: ".*"
          minChars: 0
          maxChars: 0
        - name: taskid
          type: text
          displayName: Task
          regEx: ".*"
          minChars: 0
          maxChars: 0
```

For manual authorization, these fields shouldn't be changed much.  `displayName` for each attribute.  Beyond that, OpenUnison won't have the information it will need in order to properly authorize the user and add them to the correct group.

Once your values.yaml is updated, the last step is to redeploy OpenUnison, with the additional `openunison-privileged-access` chart.  If you're deploying manually or with Argo CD, you'll need to force a full-rollout, not just installing the additional chart to ensure that the `ClusterRoleBinding` is correctly set to only allow access to users who have been authorized.  If you're using the `ouctl` utility, you can use `-r` to add additional charts:

```sh
ouctl install-auth-portal -r 'openunison-privileged-access=tremolo/openunison-privileged-access' /path/to/values.yaml
```

With manual authorization deployed, you can get a feel for how privileged access can be automated.  Once you're ready, the next step will be to integrate privileged access with your incident management system.


### Integrating with your Incident Management Service

Having looked at the manual approval process, next we can integrate OpenUnison with our incident management service.  This part is going to be different for every each enterprise and organization.  It will probably always rhyme, but differences in process, products, etc will mean that this is going to change based on your organization.  

In order to setup our privileged session we'll need to collect the incident, task, and expiration from our incident management service.  As an example, we're going to create a simple service that takes the cluster, incident, and user to return a list of possible tasks and an expiration.  We're going to use a [Python based service we published](https://github.com/OpenUnison/privileged-access-service) as an easy starting point.  Once you create the container, you can deploy it using our pre-built yaml.  Update the image to wherever you pushed your image to.  Once it's running, you can exec into a `Pod` in the openunison namespace and get a task with the url `http://pamapi.openunison.svc/verify-access?user_id=user&cluster_id=k8s&ticket_number=inc1234567` and get the response:

```json
{
  "pamTasks": [
    {
      "expiryTime": "2025-06-03T19:06:02+00:00",
      "pamTaskID": "TASK-1234"
    }
  ]
}
```

This isn't very useful in real life, but we'll use it to illustrate how to integrate our enterprise's incident management system into OpenUnison.  We're going to pull this data into OpenUnison from the same privileged access form we did before, but instead of manually specifying the task and expiration, we're going to specify a cluster and incident number, and let our service provide the rest.  When we're done, our form will look like:

![Integrated authorization privileged access form](/assets/images/privileged-access/integrated-1.png)

When an authorized user is presented with the form, they enter their incident number and click "LOOKUP" to trigger the call to the incident lookup service.  Once tasks are returned the **Task** drop down box is populated with a list of available tasks and when they expire.  The privileged user is able to choose the task they want their sessions to be associated with and once they submit, they immediately are granted the administrator access without waiting for any outside approvals.

The first step to making this happen is to create a `ConfigMap` with some javascript that will be used by OpenUnison's front end to interact with our web service.  OpenUnison's frontend is built on react, so our additional function will also need to be built on react.  This is the function that will be called when the user hits the "LOOKUP" button, but since it runs on the client, we'll also need to implement an API in OpenUnison to call our service.  Both the [`ConfigMap` and the OpenUnison `Application`](https://raw.githubusercontent.com/OpenUnison/privileged-access-service/refs/heads/main/src/kubernetes/register-extra-functions.yaml) are in the same project as our service.  We won't walk through the configuration here, but we've commented the code for you.

With the code deployed for our front end integration, we can update our values.yaml.  First, we need to configure OpenUnison to load the `ConfigMap` with our front end code in it:

```yaml
openunison:
  html:
    registerFunctionsConfigMap: scalejs-functions
```

This tells OpenUnison to attach the `scalejs-functions` `ConfigMap` to the `ouhtml-orchestra-login-portal` `Deployment` so it can be served to the user's browser.  Next, we'll start configuring our privileged access request form.  Replace your `openunison.naas.forms.privileged_access`:

```yaml
openunison:
  naas:
    forms:
      privileged_access:
        includeExtraJs: true
        test_mode: true
        uid_attribute: uid
        manual_approval:
          enabled: false
        submission_javascript: |-
          function validate_submission(newUser, errors, userData) {
            // pull data out of the selected task
            task = newUser.getAttributes().get("task");
            taskJson = JSON.parse(task);

            taskid = taskJson.pamTaskID;
            expiry = taskJson.expiryTime;


            // create an object that represents an incident
            var incident = {
              "id": newUser.getAttributes().get("incident"),
              "task": taskid,
              "expires": expiry
            };

            // serialize the data to a string, store in the globals map
            globals.put("incident_info",JSON.stringify(incident));

            // no errors, return null
            return null;
          }
        additional_attributes:
        - name: incident
          type: text-button
          displayName: Indident / Change
          regEx: "^[iI][nN][cC]\\\\d{7,8}$|^[cC][hH][gG]\\\\d{7,8}$"
          regExFailedMsg: "Incidents must be of the form INC1111111 or CHG1111111"
          minChars: 0
          maxChars: 0
          unique: false
          required: true
          editJavaScriptFunction: "react_load_tasks(eventObj)"
        - name: task
          type: list
          displayName: Task
          regEx: ".*"
          minChars: 0
          maxChars: 0
          required: true
          dynamicValueSource:
            className: com.tremolosecurity.scalejs.register.dynamicSource.JavaScriptSource
            config:
              javaScript: "GlobalEntries = Java.type('com.tremolosecurity.server.GlobalEntries'); HashMap = Java.type('java.util.HashMap'); BasicHttpClientConnectionManager = Java.type('org.apache.http.impl.conn.BasicHttpClientConnectionManager'); RequestConfig = Java.type('org.apache.http.client.config.RequestConfig'); CookieSpecs = Java.type('org.apache.http.client.config.CookieSpecs'); HttpClients = Java.type('org.apache.http.impl.client.HttpClients'); HttpGet = Java.type('org.apache.http.client.methods.HttpGet'); EntityUtils = Java.type('org.apache.http.util.EntityUtils'); ProxyConstants = Java.type('com.tremolosecurity.proxy.util.ProxyConstants'); ProxySys = Java.type('com.tremolosecurity.proxy.ProxySys'); JSUtils = Java.type('com.tremolosecurity.util.JSUtils'); BasicHeader = Java.type('org.apache.http.message.BasicHeader'); UUID = Java.type('java.util.UUID'); function init(attribute,config) {} function getSourceList(request) { ArrayList = Java.type('java.util.ArrayList'); return new ArrayList(); } function validate(value,request){ System = Java.type('java.lang.System'); var sessionId = request.getSession().getAttribute('tremolo.io/priv-session-id'); if (sessionId == null) { sessionId = UUID.randomUUID().toString(); request.getSession().setAttribute('tremolo.io/priv-session-id',sessionId); } userData = (request.getSession().getAttribute(ProxyConstants.AUTH_CTL)).getAuthInfo(); var payload = JSUtils.bytes2string(request.getAttribute(ProxySys.MSG_BODY)); System.out.println('payload: ' + payload); var payloadJson = JSON.parse(payload); var incident = payloadJson.attributes.incident; var cluster = payloadJson.attributes.cluster; var bhcm = new BasicHttpClientConnectionManager(GlobalEntries.getGlobalEntries().getConfigManager().getHttpClientSocketRegistry()); var rc = RequestConfig.custom().setCookieSpec(CookieSpecs.STANDARD).setRedirectsEnabled(false).build(); var http = HttpClients.custom().setConnectionManager(bhcm).setDefaultRequestConfig(rc).build(); var taskData = JSON.parse(value); try { var get = new HttpGet('http://pamapi.openunison.svc/verify-access?user_id=' + userData.getAttribs().get('sub').getValues().get(0) + '&cluster_id=' + cluster + '&ticket_number=' + incident); get.addHeader(new BasicHeader('tremoloio-request-type','verify')); get.addHeader(new BasicHeader('tremoloio-priv-session-id',sessionId)); svcResp = http.execute(get); if (svcResp.getStatusLine().getStatusCode() == 200) { respPayload = EntityUtils.toString(svcResp.getEntity()); tasks = JSON.parse(respPayload); var found = false; for (var i=0;i<tasks.pamTasks.length;i++) { if (tasks.pamTasks[i].pamTaskID == taskData.pamTaskID) { found = true; break; } } if (! found) { return 'Incident and task are not associated'; } } else { return 'Unable to validate incident and task'; } } finally { if (http) { http.close(); } if (bhcm) { bhcm.close(); } } return null; }"
              
                # Pre-minified code
                # GlobalEntries = Java.type("com.tremolosecurity.server.GlobalEntries");
                # HashMap = Java.type("java.util.HashMap");
                # BasicHttpClientConnectionManager = Java.type("org.apache.http.impl.conn.BasicHttpClientConnectionManager");
                # RequestConfig = Java.type("org.apache.http.client.config.RequestConfig");
                # CookieSpecs = Java.type("org.apache.http.client.config.CookieSpecs");
                # HttpClients = Java.type("org.apache.http.impl.client.HttpClients");
                # HttpGet = Java.type("org.apache.http.client.methods.HttpGet");
                # EntityUtils = Java.type("org.apache.http.util.EntityUtils");
                # ProxyConstants = Java.type("com.tremolosecurity.proxy.util.ProxyConstants");
                # ProxySys = Java.type("com.tremolosecurity.proxy.ProxySys");
                # JSUtils = Java.type("com.tremolosecurity.util.JSUtils");
                # BasicHeader = Java.type("org.apache.http.message.BasicHeader");
                # UUID = Java.type("java.util.UUID");

                # function init(attribute,config) {

                # }

                # // we don't need to retrieve the list on its own because it will
                # // be loaded interactively from the form
                # function getSourceList(request) {
                #   ArrayList = Java.type('java.util.ArrayList');
                #   return new ArrayList();
                # }

                # // make sure that the window submitted matches for the requested incident
                # // this is duplicative, but could be avoided by signing the data on response to the frontend
                # function validate(value,request){
                #   System = Java.type("java.lang.System");
                  
                #   //useful for tracking requests
                #   var sessionId = request.getSession().getAttribute("tremolo.io/priv-session-id");
                #   if (sessionId == null) {
                #     sessionId = UUID.randomUUID().toString();
                #     request.getSession().setAttribute("tremolo.io/priv-session-id",sessionId);
                #   }
                  
                  


                #   //get the data from the payload to verify
                #   userData = (request.getSession().getAttribute(ProxyConstants.AUTH_CTL)).getAuthInfo();
                #   var payload = JSUtils.bytes2string(request.getAttribute(ProxySys.MSG_BODY));
                #   System.out.println("payload: " + payload);
                #   var payloadJson = JSON.parse(payload);
                #   var incident = payloadJson.attributes.incident;
                #   var cluster = payloadJson.attributes.cluster;

                #   // create an http connection to our service.  there's no authentication, but for a production system this
                #   // would be a good place to include an authentication token
                #   var bhcm = new BasicHttpClientConnectionManager(GlobalEntries.getGlobalEntries().getConfigManager().getHttpClientSocketRegistry());
                #   var rc = RequestConfig.custom().setCookieSpec(CookieSpecs.STANDARD).setRedirectsEnabled(false).build();
                #   var http = HttpClients.custom().setConnectionManager(bhcm).setDefaultRequestConfig(rc).build();

                #   var taskData = JSON.parse(value);
                  
                #   try {
                #     // call our service
                #     var get = new HttpGet("http://pamapi.openunison.svc/verify-access?user_id=" + userData.getAttribs().get("sub").getValues().get(0) + "&cluster_id=' + cluster + '&ticket_number=" + incident);
                    
                #     // useful for tracking requests
                #     get.addHeader(new BasicHeader("tremoloio-request-type","verify"));
                #     get.addHeader(new BasicHeader("tremoloio-priv-session-id",sessionId));

                #     svcResp = http.execute(get);
                #     if (svcResp.getStatusLine().getStatusCode() == 200) {
                #.      // everything is OK, make sure that the returned task matches what was submitted
                #       respPayload = EntityUtils.toString(svcResp.getEntity());
                #       tasks = JSON.parse(respPayload);
                #       var found = false;
                #       for (var i=0;i<tasks.pamTasks.length;i++) {
                #         if (tasks.pamTasks[i].pamTaskID == taskData.pamTaskID /*&& tasks.pamTasks[i].expiryTime == taskData.expiryTime*/) {
                #           found = true;
                #           break;
                #         }
                #       }

                #.      // there was a problem, return an error
                #       if (! found) {
                #         return "Incident and task are not associated";
                #       }

                #     } else {
                #       return "Unable to validate incident and task";
                #     }
                #   } finally {
                #     if (http) {
                #       http.close();
                #     }
                #     if (bhcm) {
                #       bhcm.close();
                #     }
                #   }

                #   return null;
                # }
```

There are some important differences from the manual authorization configuration:

* ***Disable Manual Authorizations*** - `openunison.naas.forms.privileged_access.manual_approval.enabled=false` to disable the manual workflow
* ***Test Mode Enabled*** - `openunison.naas.forms.privileged_access.test_mode=true` allows us to test the integration with our service without triggering a privileged access request.
* ***Updates to the submission javascript*** - We're pulling our expiration data from the loaded task information, instead of directly from fields.
* ***Changes to attributes*** - Instead of asking for expiration data, we configure a lookup for our task and a drop down list.  The validation javascript for the list of tasks needs to be minified (removing `\n` and changing `"` --> `'`).  We included the de-minified version.

With our updates made, now we just need to re-run ouctl to deploy our changes:

```sh
ouctl install-auth-portal -r 'openunison-privileged-access=tremolo/openunison-privileged-access' /path/to/values.yaml
```

Now you can login and test your privileged access form.  Once you're satisfied with it and want to enable it for production use, change `openunison.naas.forms.privileged_access.test_mode=false`.

Now that your service is deployed and integrated with the incident service, what happens if your incident management service isn't available?  We'll cover that scenario next.

#### Break Glass Administrator Access

What happens if your incident management system isn't available during an outage?  How do you ensure your cluster administrators have access?  OpenUnison includes a workflow that allows for authorized users to create a `ClusterRoleBinding` in all of the managed clusters to make sure cluster administrators aren't locked out.  This list of users are generally members of the security team and are authorized.  When privileged access is enabled, a `ClusterRoleBinding` is created in each manage cluster to bind anyone that could be a cluster administrator.  Once the incident management service is back online, break glass can be disabled as well by the same users.  To configure your break glass:

```yaml
openunison:
  naas:
    forms:
      privileged_access:
        # this is a group from your enterprise identity provider
        external_group: CN=kube-break-glass,CN=Users,DC=ent2k22,DC=tremolo,DC=dev
```

Redeploy your `openunison-privileged-access` helm deployment, and now any member of the group configured above can enable break glass mode.  When they login to OpenUnison and click on *Request Access*, they'll see an option for *Break Glass* where they can run the enable or disable workflows:

![Break glass workflow](/assets/images/privileged-access/break-glass.png)

#### What's Next?

Nearly all of OpenUnison can be customized via CRDs and javascript.  Check out our applications section to see examples of how to manage more of your organizations management systems into OpenUnison.