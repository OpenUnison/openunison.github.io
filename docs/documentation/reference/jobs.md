# Scheduled Jobs

## ClearLogouts

Deletes Kubernetes `EndSession` objects that are more then a minute old.

```yaml
---
apiVersion: openunison.tremolo.io/v1
kind: OUJob
metadata:
  name: clear-logouts
  namespace: openunison
spec:
  className: com.tremolosecurity.jobs.ClearLogouts
  cronSchedule:
    dayOfMonth: '*'
    dayOfWeek: '?'
    hours: '*'
    minutes: '*'
    month: '*'
    seconds: '*'
    year: '*'
  group: admin-local-sessions
  params:
    # name of the kubernetes target
    - name: targetName
      value: k8s
    # name of the namespace holding the end
    - name: namespace
      value: openunison
```