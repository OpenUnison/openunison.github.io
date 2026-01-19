# Custom Provisioning Targets

One of OpenUnison's most powerful features is its ability to simplify the onboarding, update, and deletion of users in applications.  While OpenUnison already has an [extensive list of supported applications](/documentation/reference/targets/), it's impossible for us to handle all applications.  The good news is that we've made it easier to create a custom target the same way you can create other customized OpenUnison configurations, through JavaScript!

When creating a custom target the development process can be on of the most difficult and time consuming processes.  To keep development time down, you want to be able to automate testing locally without having to deploy your target.  Since OpenUnison is built on Java, we use Java tooling for automating the testing of targets.  In our example, we're going to use [Apache Maven](https://maven.apache.org/) to automate the testing of our target.  Maven has extensive documentation and integrates well with the [Apache JUnit](https://junit.org/) project.  If you've never worked with these tools before, don't worry!  we're going to walk through step-by-step.

## Create a Maven Project

All you need to get started is Java 21 and Maven.  Then you can [fork the openunison-test-harness](https://github.com/OpenUnison/openunison-test-harness) project on GitHub.  This project is self contained,  once downloaded you can make sure everything is working:

```bash
openunison-test-harness git:(main) mvn clean test
[INFO] Scanning for projects...
[INFO] 
[INFO] ---------------< openunison-provisioning:custom-target >----------------
[INFO] Building custom-target 1.0.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ custom-target ---
[INFO] Deleting /path/to/openunison-test-harness/target
.
.
.
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 7.866 s -- in io.openunison.provisioning.customtarget.CustomTargetTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 9, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  10.252 s
[INFO] Finished at: 2026-01-13T18:07:02-05:00
[INFO] ------------------------------------------------------------------------
```

You're now ready to being developing your target.

## Develop Your Target

With your test harness ready to begin development, you can start working on your target by duplicating `src/test/yaml/custom-target.yaml`.  This target is a simple implementation built around an in-memory JSON database of users.  It's obviously not a production viable user management system, but it should make for a good example of how to implement a target.  THe majority of the work is marshalling between your downstream system's structure and the `com.tremolosecurity.provisioning.core.User` and `com.tremolosecurity.provisioning.core.Group` classes. We'll walk through the main functions for pushing identity data into your downstream system.

### Generic Documentation

OpenUnison custom targets are built off of Java, but this section details implementing them in JavaScript.  JavaScript implementations don't require a custom built OpenUnison container because the code is loaded from a CRD directly.  Our [JavaScript customization guide](/javascript/) details where to look for additional examples and helper functions.

### Logging Changes

Whenever making changes to an object in a downstream system, it should be logged.  OpenUnison provides a change log function to make this easier.  If your OpenUnison is deployed with audit database support, like when using our [Namespace as a Service Portal](/namespace_as_a_service/), these functions will log both to standard logging and to the audit database.  Even if not using the audit database, you should be piping the logs from OpenUnison into a SIEM such as Splunk or similar.  These logs can make be very valuable in trying to determine why a user's object in a downstream system looks the way it does. 

In the example project, our target's logging will look something like:


```javascript
// log changes
// first load metadata about the workflow from the request
// this is provided by OpenUnison
approvalID = 0;
if (request.containsKey("APPROVAL_ID")) {
approvalID = request.get("APPROVAL_ID");
}

workflow = request.get("WORKFLOW");

cfgMgr = state.get("cfgMgr");
name = state.get("name");

// log the action
cfgMgr.getProvisioningEngine().logAction(name,false, ProvisioningUtil.ActionType.Add,  approvalID, workflow, key, val);
```

The above function will generate log data that looks like:

```
[2026-01-18 10:15:48,317][main] INFO  ProvisioningEngineImpl - target=custom-target entry=false Add user=test workflow=null approval=0 group='group4'
```

The parameters for the `logAction` function are:

| Parameter | Description |
| --------- | ----------- |
| **name** | The name of your target, configured in `init` |
| **isEntry** | if `true`, this action applies to the object, not a specific attribute.  When creating or deleting an object, this should be `true`.  When changing individual attributes, should be `false`. |
| **actionType** | One of `ProvisioningUtil.ActionType.Add`, `ProvisioningUtil.ActionType.Replace`, `ProvisioningUtil.ActionType.Delete` depending on the action |
| **approvalID** | If part of an approval workflow, the approval id.  Provided by OpenUnison |
| **workflow** | The workflow object, provided by OpenUnison | 
| **attributeName** | The attribute name being changed.  If `isEntry` is `true`, this should be the user's login or id |
| **attributeValue** | The value of the change |



### Required Functions

#### `init(cfg, cfgMgr, name)`

Called when the target is initially setup.  Any configuration information or pools that need to be accessed by other functions can be stored in the state `Map`.

#### `findUser(userID, attributes, request)`

Look for the user based on their user name or id.  This ID must be unique.  It is generally a user's login.  If your downstream supports an immutable id, this is often stored as an attribute.  If the user doesn't exist, return null.

####  `createUser(user,attributes,request)`

Creates the user in your downstream system, based on the passed in user object.  Only the attributes included in the `attributes` set should included.

#### `deleteUser(user,request)`

Deletes the passed in user.

#### `syncUser(user,addOnly,attributes,request)`

Synchronizes the user from the `user` object into the downstream system.  This function should be written in an idempotent way, such that if called multiple times with the same data the user object should be the same.

#### `setUserPassword(user,request)`

If your target supports passwords, implement this function to set the user's password.

#### `shutdown()`

Called when a target is being decommissioned so that shutdown logic such as clearing pools and other resources can be performed.

### Lookup Functionality

These functions support more lookup capabilities then the `findUser` function.  They must be implemented if your target is being built to support a SCIM 2.0 gateway.

#### `lookupUserByLogin(login)`

Similar to findUser, but expected to return all possible attributes.  It also assumes that your input is a user login name, instead of an immutable unique id.

#### `lookupUserById(id)`

Looks up a user by an immutable id.  This is generally not the same as the user's login name.  Not all downstream targets will support an immutable id outside of their login name.  If not supported, just return `lookupUserByLogin(id)`.

#### `searchUsers(ldapFilter)`

Search for a user with more then just their id or login.  The LDAP Filter can be converted into whatever search filter your downstream supports.  If your downstream doesn't support a complex filter, you can do lookups and then use the utility methods to verify the search.

#### `lookupGroupByName(name)`

Search for a group based on it's name.  If your downstream system supports unique user ids, then members should be the ids.  If it doesn't support unique ids, then the members should be user names.

#### `lookupGroupById(id)`

Search for a user based on it's unique id.  If your store doesn't support unique ids for groups, return `lookupGroupByName(id)`.  If your downstream system supports unique user ids, then members should be the ids.  If it doesn't support unique ids, then the members should be user names.

#### `searchGroups(ldapFilter)`

Search for a group based on a complex filter, often looking for a specific member.  You can convert the LDAP filter into what your downstream target supports, or just do a simple lookup and compare the results.  If your downstream system supports unique user ids, then members should be the ids.  If it doesn't support unique ids, then the members should be user names.

#### `isGroupMembersUniqueIds()`

Return `true` if the `searchGroups`, `lookupGroupById`, and `lookupGroupByName` functions return members as unique ids.  `false` if members are user login names.

#### `isUniqueIdTremoloId`

Return `true` if the user's login/user name is different from the unique id and a unique id is supported.  If a unique id is not supported, return `false`.

#### `isGroupIdUniqueId`

Return `true` if group objects have a unique id instead of just a name.


## Test Your Target

While we separated documentation between development and testing, the fastest approach is to develop and test is to use an iterative process.  Generally start with `findUser`, as this is where the most marshalling occurs.  Once `findUser` is built and tested, the other functions tend to come together quickly.  Running `mvn clean test` will run the test case that will run through all of your JUnit test cases.  You can pass configuration data into your target by simulating how OpenUnison parametrizes configurations and from secrets.  From our example test case:

```java
// Create system/environment variables used by your target
// accessed by using #[name] instead of the variable
// DO NOT INCLUDE SECRET DATA!!!!
HashMap<String,String> env = new HashMap<>();
env.put("name","value");


// Simulates pulling data from Kubernetes Secret objects
// translates to the secretParams section of your target configuration
Map<String,Map<String,String>> secrets = new HashMap<>();
Map<String,String> orchestrSecretsSource = new HashMap<>();
orchestrSecretsSource.put("somepasswordname","value");
secrets.put("orchestra-secrets-source",orchestrSecretsSource);
```
This way your target has no environmental data hard coded.  You can populate this data from environment variables available to the test case, making it easier to automate continuous testing.


## Deploy Your Target

Deploying a custom target can be as simple as `kubectl create -f /path/to/target.yaml`, however this is not a great way to do this in production.  Assuming you haven't hard coded anything that's environment specific, you can copy the target yaml into a git repository or other project that gets synced into your cluster.

If using `#[]` syntax for configuration, these only get updated when the operator redeploys OpenUnison.  For instance, if you have a database url in the option `#[databaseurl]`, to configure it would would add `databaseurl` to the `openunison.non_secret_data` section of your values.yaml such as:

```yaml
openunison:
  non_secret_data:
    databaseurl: jdbc:mysql://....
```

If you want your yaml to update without a restart, you can translate `#[database.url]` to `{{ .Values.openunison.non_secret_data.databaseurl }}` and your target will hot load the change without a restart.

