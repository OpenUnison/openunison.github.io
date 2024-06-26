# Customizing the NaaS Onboarding Workflow

The out-of-the-box Namespace as a Service workflow will generate a `Namespace` and `RoleBindings`, along with the groups needed to manage access.  This is rarely enough to satisfy multitenancy requirements.  You'll generally want to create at least a `ResourceQuota`, maybe `NetworkPolicy`s, etc.  There are several customization points where you can add your own tasks.

## Workflow Hooks

You can run multiple custom workflows at different points through the `Namepsace` creation process.  A custom workflow is where you can create custom objects in your cluster, your new `Namespace`, or a remote system like a GitHub or Gitlab repository.  All custom workflows should be associated with a non existent `Organization` and not be "listed".  For instance, the below workflow creates a `ResourceQuota` after the `Namespace` is created:

```
---
apiVersion: openunison.tremolo.io/v1
kind: Workflow
metadata:
  name: create-resource-quota
  namespace: openunison
spec:
  description: creates a ResourceQuota object
  inList: false
  label: Creates a ResourceQuota
  orgId: internal-does-not-exist
  tasks: |-
      - taskType: customTask
        className: com.tremolosecurity.provisioning.tasks.CreateK8sObject
        params:
            targetName: $cluster$
            template: |-
                kind: ResourceQuota
                apiVersion: v1
                metadata:
                  name: resource-quota
                  namespace: $nameSpace$
                annotations:
                    tremolo.io/managedByGit: "$useGit$"
                spec:
                    hard:
                        cpu: "5"
                        memory: 10Gi
                        pods: 10
            srcType: yaml
            writeToRequest: "$useGit$"
            requestAttribute: git-secret-cluster-k8s-$nameSpace$
            path: /yaml/ns/$nameSpace$/resourcequotas/
```

There are some variables available to your workflow:

| Request Attribute | Name |
| ----------------- | ---- |
| $nameSpace$       | The `Namespace` to be created |
| $cluster$         | The target cluster where to create the tenant |

Once you've deployed your `Workflow`, set its name to the correct setting (see below) in your values.yaml and redeploy the `tremolo/orchestra-k8s-cluster-management` chart.  See the [Kubernetes Workflow Tasks](../../applications/kubernetes#workflow-tasks) for how to interact with your clusters from an OpenUnison `Workflow`.

### Pre-Run Workflow

This hook is where you can run any processes you want to start before the workflow begins building out the tenant.  This is before groups are created.  This might be a good time to create a git repository as an example.  

**Helm Value:** `openunison.naas.workflows.new_namespace.pre_run_workflow`

### Pre-Provision Workflow

You can define a workflow that will run after the groups are all created, but before the `Namespace` is created.  This may be a good time to create cluster level objects.  Once your workflow is created.  

**Helm Value:** `openunison.naas.workflows.new_namespace.pre_provision_workflow`

### Post Namespace Create Workflow

Once the namespace is created, you can run a workflow to create additional objects in the `Namespace`.  

**Helm Value:** `openunison.naas.workflows.new_namespace.post_namespace_create_workflow`

## Custom Attributes

The form for onboarding a new namespace can have custom attributes added.  These attributes can be as simple as a text box, or as complex as a drop down with values loaded from an external API.  Adding an attribute requires an update to the `openunison.naas.forms.new_namespace.additional_attributes` section.  For instance, if you wanted to as users if they want to enable istio side car injection in your namespace, you could add:

```yaml
openunison:
  naas:
    forms:
      new_namespace:
        additional_attributes:
        - name: istio_enabled
          # what will be displayed on the form
          displayName: Enable Istio Sidecars
          # validating regular expression
          regEx: ".*"
          # An error to provide the user if the regex fails
          regExFailedMsg: "Invalid istio configuration"
          # minimum number of characters
          minChars: 0
          # maximum number of characters
          maxChars: 0
          # should the value of this attribute be unique in OpenUnison
          unique: false
          # can by one of list, text, dynamicList
          type: list
          # if list, the values and labels in the form of DisplayLabel=value
          values:
          - Enabled=enabled
          - Disabled=disabled
```

Once the attribute is added to a form, it's available to workflows as both a request object with the name of the attribute or a user attribute.  Attribute values are also available to approvers to provide additional context.

## Namespace Labels and Annotations

When the namespace is provisioned, it can have custom labels and annotations.  These annotations and labels can be generated from custom attributes, or created in a (workflow hook)[#workflow_hooks].  The value for an annotation or label must be preset in the request object.  You add custom labels through name/value pairs in `openunison.naas.workflows.new_namespace.namespace_request_labels` and annotations through `openunison.naas.workflows.new_namespace.namespace_request_annotations`.  For instance, to add the `istio-injection` label to a new namespace, you would add:

```yaml
openunison:
  naas:
    workflows:
      new_namespace:
          namespace_request_labels:
            istio-injection: istio_enabled
```