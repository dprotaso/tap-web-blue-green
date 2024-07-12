## TAP Web Workloads Blue/Green Deploy - POC

#### Installation of custom templates

Note: this is following the steps outlined in the TAP Documents
https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.11/tap/scc-authoring-supply-chains.html#modifying-an-out-of-the-box-supply-template-4

1. Update your TAP values to exclude some default templates

```yaml
ootb_templates:
  excluded_templates:
  - app-deploy
  - config-template
```

Update TAP and ensure the old templates are gone

```sh
$ kubectl get clusterdeploymenttemplates.carto.run app-deploy
$ kubectl get clusterconfigtemplates.carto.run config-template
```

2. Apply the custom templates that have the correct rebase rules

```sh
kubectl apply -f app-deploy.yaml -f config-template.yaml
```

The custom configs have an update kapp config for Knative workloads. We include an additional rebaseRule so that kapp doesn't touch traffic block unless it's set explicitly.

```yaml
rebaseRules:
- path:
  - spec
  - traffic
  type: copy
  sources:
  - new
  - existing
  resourceMatchers:
  - apiVersionKindMatcher:
      apiVersion: serving.knative.dev/v1
      kind: Service
``

#### Create the first workload wait for it to succeed

```sh
$ tanzu apps workload apply --file workload-blue.yaml --yes
...
$ tanzu apps workload get go-app
...
$ kubectl get kservice 
```

#### Pin traffic to the first Revision

Now manipulating the `spec.traffic` of the Knative Service directly.
See the Knative documentation to understand the details https://knative.dev/docs/getting-started/first-traffic-split/

```sh
$ kubectl patch kservice go-app --type "merge" -p '{
  "spec":{
    "traffic":[
        {"percent":100, "revisionName":"go-app-00001"}
    ]
  }
}'
```

#### Deploy the second workload version and wait for the second revision to appear

```sh
$ tanzu apps workload apply --file workload-green.yaml --yes
...
$ kubectl get revisions
```

#### Shift traffic pecentage wise

```sh
$ kubectl patch kservice go-app --type "merge" -p '{
  "spec":{
    "traffic":[
        {"percent":50, "revisionName":"go-app-00001"},
        {"percent":50, "revisionName":"go-app-00002"}
    ]
  }
}'
```
## Notes

Backup and restore is broken with web workloads due to odd velero assumptions. 
It is recommended prior to backup you point the traffic back to the latest revision.
