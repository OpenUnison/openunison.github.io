# Kubernetes Integration

This page has common resources that can be used for integrating and managing Kubernetes clusters.  In addtion to providing SSO access to your clusters, OpenUnison can provision objects into your clusters.

## Workflow Tasks

### Create Kubernetes Object

The most common Kubernetes `Workflow` task is `CreateK8sObject`.  This task will create a new object in your cluster if it doesn't exist.  If it does exist, it will overwrite it.  There are two ways to write your object, either directly to a cluster's API, or you can commit your object to a git repository allowing for the automation of GitOps.

```yaml

- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.CreateK8sObject
  params:
    # the name of the cluster to provision to, must by the name of a `Target` object
    targetName: $cluster$
    # The YAML or JSON to generate
    template: |-
        kind: ServiceAccount
        apiVersion: v1
        metadata:
          name: "gitops"
          namespace: $nameSpace$
    # can by yaml or json
    srcType: yaml
    # optional, if "true", then instead of writing the object to the API server
    # the object is written into the workflow request so it can be pushed into Git
    writeToRequest: "false"
    # optional, if writeToRequest is "true", the name of the request object to store the object in
    requestAttribute: git-secret-cluster-$cluster$-$nameSpace$
    # optional, if writeToRequest is "true", the path in the git repo to write the file to
    path: /yaml/ns/$nameSpace$/serviceaccounts/gitops.yaml
```

#### Working with GitOps

In order for OpenUnison to work with a remote repository, a `Secret` must exist that contains the private key used to talk to the remote git repository.

### Delete Kubernetes Object

This task will delete an object, either directly against the cluster's API or in a remote git repository.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.DeleteK8sObject
  params:
    # the name of the cluster to provision to, must by the name of a `Target` object
    targetName: $cluster$
    # The type of object being deleted
    kind:  RoleBinding
    # The URI for the object to be deleted
    url: /apis/rbac.authorization.k8s.io/v1/namespaces/$nameSpace$/rolebindings/{{ $bind.binding }}-binding{{ $root.Values.openunison.naas.groups.internal.suffix }}
    # optional, if "true", then instead of deleting the object to the API server
    # the object is deleted in the workflow request so it can be pushed into Git
    writeToRequest: "false"
    # optional, if writeToRequest is "true", the name of the request object to store the deleted object in
    requestAttribute: git-secret-namespace-$cluster$-$nameSpace$
    # optional, if writeToRequest is "true", the path in the git repo to delete
    path: /yaml/ns/$nameSpace$/rolebindings/{{ $bind.binding }}-binding{{ $root.Values.openunison.naas.groups.internal.suffix }}.yaml
```

### Patch Kubernetes Object

Patches an object with a given JSON either directly against the API or into a Git repository.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.PatchK8sObject
  params:
    # the name of the cluster to provision to, must by the name of a `Target` object
    targetName: $cluster$
    # The type of object being patched
    kind: Namespace
    # The URI for the object to be patched
    url: /api/v1/namespaces/$nameSpace$
    # what kind of patch, one of marge (default), strategic, or json
    patchType: merge
    # the patch template
    template: |-
      {
        "metadata": {
          "annotations": {
            "splunk_server": "$splunk_server$",
            "splunk_index": "$splunk_index$"
          }
        }
      }
    # optional, if "true", then instead of patching the object to the API server
    # the object is patched in the workflow request so it can be pushed into Git
    writeToRequest: "false"
    # optional, if writeToRequest is "true", the name of the request object to store the patch object in
    requestAttribute: git-secret-namespace-$cluster$-$nameSpace$
    # optional, if writeToRequest is "true", the path in the git repo to patch
    path: /yaml/ns/$nameSpace$/rolebindings/{{ $bind.binding }}-binding{{ $root.Values.openunison.naas.groups.internal.suffix }}.yaml
```

### Push To Git

When working with GitOps based clusters, this task will take all of the objects stored in the request object and push them into git.  

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.PushToGit
  params:
    # The name of the Kubernetes Secret that stores the private key for the remote git repository
    secretName: sshkey-cluster-$cluster$
    # the namespace where the Secret is stored
    nameSpace: openunison
    # The cluster where the Secret is stored
    target: k8s
    # The key name in the Secret that stores the private key used to establish an ssh connection to the remote repository
    keyName: id_rsa
    # the git ssh URL to the remote repository
    gitRepo: $clusterGitUrl$
    # the name of the request object storing changes to the remote repository
    requestObject: git-secret-cluster-$cluster$-$nameSpace$
    # The commit message
    commitMsg: For workflow $WORKFLOW.id$
```

Once the push happens, it's common to use the [Wait For Status](#wait-for-status) task to wait for your GitOps controller to synchronize the objects before moving on in the workflow.

### Wait For Status

When provisioning an object that can take some time and is asynchronous, it's helpful to be able to pause the workflow until a certain object is created and a status is set.  For instance, provisioning a vCluster via the ClusterAPI can take a few minutes.  Your workflow needs to wait until the vCluster is ready before provisioning OpenUnison into it.  This task pauses the workflow until an object has been created and a status has been met.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.WaitForStatus
  params:
    # control plane cluster
    holdingTarget: k8s
    # namespace that holds the target for the cluster to test
    namespace: openunison
    # target that points to the cluster you wish to test
    target: $cluster$
    # The URI of the object in the target cluster to test
    uri: /apis/apps/v1/namespaces/$nameSpace$/statefulsets/vcluster
    # Label for this test
    label: wait-for-vcluster
    # List of conditions that must be met in JSON Path notation
    conditions:
    - .status.readyReplicas=1
    - .status.replicas=1
```

In addition to adding this task, make sure the `oujob` **wait-for** has been created:

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: OUJob
metadata:
  name: wait-for
  namespace: openunison
spec:
  className: com.tremolosecurity.provisioning.jobs.WaitForJob
  cronSchedule:
    dayOfMonth: '*'
    dayOfWeek: '?'
    hours: '*'
    minutes: '*'
    month: '*'
    seconds: '*/10'
    year: '*'
  group: admin
  params:
    - name: target
      value: k8s
    - name: namespace
      value: openunison
```

### CleanLabels

Cleans request attributes that are meant to become labels on Kubernetes objects. Labels must be compliant with DNS host names. This task will replace invalid characters and inject an "x" before or after the label if the leading or trailing characters don't properly match.

```yaml
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.CleanLabels
  params:
    # Character that will replace invalid characters in the named attributes
    replacementCharacter: "-"
    # The suffix to use on the new request attribute. For instance, if cleaning "rawlabel," and setting this to "_clean," you would reference "rawlabel_clean" in the tasks for creating objects.
    newAttributeSuffix: "_clean"
    # Request attributes to be cleaned, may be listed multiple times
    attributes: "raw label"
  secretParams: []
```

### CheckApiExists

***1.0.43+***

When working in a multi-cluster environment, you can't always guarantee every cluster is at the same API level.  This task will let you check the remote cluster and add an attribute to the user object that can be tested with the `ifAttrExists` task:

```yaml
# check if the remote target has EndSessions enabled
- taskType: customTask
  className: com.tremolosecurity.provisioning.tasks.CheckApiExists
  params:
    # apiGroup to check
    apiGroup: openunison.tremolo.io/v1
    # kind to check
    apiKind: EndSession
    # cluster to check, must have a corresponding Target object
    cluster: k8s-$clusterName$
    # the user attribute to create
    attribute: api_enabled
```