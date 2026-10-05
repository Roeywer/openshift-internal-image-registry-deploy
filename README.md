# Internal OpenShift image registry on NetApp S3

OpenShift 4.20 + NetApp AWS-compatible S3. Get bucket, HTTPS endpoint, access key, and secret key from the storage team. Reuse the existing `custom-ca` ConfigMap in `openshift-config` for S3 TLS. Then configure `configs.imageregistry.operator.openshift.io/cluster` and pin pods to infra nodes.

| File | Purpose |
| --- | --- |
| [image-registry-s3-infra.yaml](image-registry-s3-infra.yaml) | Registry Config: S3 storage, infra placement, `custom-ca` |
| [image-pruner-infra.yaml](image-pruner-infra.yaml) | Image pruner CronJob on infra nodes |

Docs: [Registry 4.20](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/registry/index), [S3-compatible storage](https://docs.okd.io/4.20/registry/configuring_registry_storage/configuring-registry-storage-aws-user-infrastructure.html).

## 0. Prerequisites

Ask the other team for:

- S3 bucket name
- HTTPS `regionEndpoint` (scheme required, e.g. `https://s3.example.com`)
- Access key and secret key

Also:

- Existing ConfigMap `custom-ca` in `openshift-config` (cluster custom CA)
- `oc` logged in as `cluster-admin`
- Infra nodes labeled `node-role.kubernetes.io/infra=`
- Taints: `node-role.kubernetes.io/infra=reserved:NoSchedule` and `NoExecute`
- Infra nodes can reach the S3 HTTPS endpoint

On bare metal / vSphere the registry often ships as `Removed` until storage is set and `managementState: Managed`.

## 1. Confirm current registry state

```bash
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
oc get co image-registry
oc get pods -n openshift-image-registry
oc get nodes -l node-role.kubernetes.io/infra -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

## 2. Reuse `custom-ca` in `openshift-config`

Do not create a second CA ConfigMap. The Image Registry Operator only reads a ConfigMap from `openshift-config`; set `spec.storage.s3.trustedCA.name: custom-ca`. The data key **must** be `ca-bundle.crt`.

```bash
oc get cm custom-ca -n openshift-config
oc get cm custom-ca -n openshift-config -o jsonpath='{range $k,$v := .data}{$k}{"\n"}{end}'
```

If the key is not `ca-bundle.crt`, add or rename it on that same ConfigMap (do not copy to another name). The S3 CA must already be in that bundle; if NetApp S3 uses a CA that is missing from `custom-ca`, append it there.

## 3. Credentials secret (required name and keys)

```bash
oc create secret generic image-registry-private-configuration-user \
  --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY='REPLACE_ACCESS_KEY' \
  --from-literal=REGISTRY_STORAGE_S3_SECRETKEY='REPLACE_SECRET_KEY' \
  -n openshift-image-registry
```

Recreate if it already exists (`oc delete secret ...` then create again).

## 4. Enable the registry on NetApp S3 and infra nodes

Edit `bucket` and `regionEndpoint` in `image-registry-s3-infra.yaml`, then:

```bash
oc apply -f image-registry-s3-infra.yaml
```

If StorageGRID uses virtual-hosted URLs (`https://<bucket>.<s3-domain>/...`), set `virtualHostedStyle: true`.

Equivalent patch:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{
  "spec": {
    "managementState": "Managed",
    "replicas": 2,
    "disableRedirect": true,
    "nodeSelector": {"node-role.kubernetes.io/infra": ""},
    "tolerations": [
      {"effect":"NoSchedule","key":"node-role.kubernetes.io/infra","operator":"Equal","value":"reserved"},
      {"effect":"NoExecute","key":"node-role.kubernetes.io/infra","operator":"Equal","value":"reserved"}
    ],
    "storage": {
      "managementState": "Unmanaged",
      "s3": {
        "bucket": "REPLACE_BUCKET_NAME",
        "region": "us-east-1",
        "regionEndpoint": "https://s3.netapp.example.com",
        "virtualHostedStyle": false,
        "encrypt": false,
        "trustedCA": {"name": "custom-ca"}
      }
    }
  }
}'
```

## 5. Wait until healthy

```bash
oc get co image-registry
oc get deploy,pods -n openshift-image-registry -o wide
oc logs -n openshift-image-registry -l docker-registry=default --tail=80
```

`image-registry-*` must run on infra nodes. Operator: Available, not Degraded.

Common failures:

- `x509: certificate signed by unknown authority` → NetApp S3 CA missing from `custom-ca`, or the ConfigMap key is not `ca-bundle.crt`
- `NoSuchBucket` / `SignatureDoesNotMatch` → bucket name, keys, clock skew, path vs virtual-hosted style
- `Pending` pods → missing infra tolerations
- Redirect / timeout on pull → `disableRedirect: true`

## 6. Enable image pruning

`successfulJobsHistoryLimit` and `failedJobsHistoryLimit` are **not** on the registry Config. They belong on `imagepruners.imageregistry.operator.openshift.io/cluster` (must be `>= 1`; default is `3`). YAML needs a space after the colon: `failedJobsHistoryLimit: 3`.

Prune **options** (what gets deleted from ImageStreams and, when the registry is `Managed`, from NetApp S3):

| Field | Meaning | Default |
| --- | --- |
| `suspend` | `false` = CronJob runs | `false` |
| `schedule` | Cron | `0 0 * * *` (midnight) |
| `keepTagRevisions` | Revisions kept per ImageStream tag | `3` |
| `keepYoungerThanDuration` | Do not prune images younger than this | `60m` |
| `successfulJobsHistoryLimit` | Completed pruner Jobs to keep | `3` |
| `failedJobsHistoryLimit` | Failed pruner Jobs to keep | `3` |

With registry `managementState: Managed`, the job uses `--prune-registry=true` and removes unused blobs from S3. If the registry is `Removed`, it only prunes etcd image metadata.

```bash
oc apply -f image-pruner-infra.yaml
oc get imagepruner cluster -o yaml
oc get cronjob -n openshift-image-registry
oc get jobs -n openshift-image-registry
```

To pause pruning: set `spec.suspend: true`.

`keepYoungerThanDuration: 60m` is aggressive for production. Increase it (for example `96h`) if builds should stay longer.

## 7. Optional: default route

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"defaultRoute":true}}'
HOST=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login -u kubeadmin -p "$(oc whoami -t)" "$HOST"
```

## 8. Smoke test

Push via a BuildConfig to `image-registry.openshift-image-registry.svc:5000`. Confirm objects in the NetApp bucket (docker/registry layout).

## Rollback

Does not delete Unmanaged NetApp data:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge \
  -p '{"spec":{"managementState":"Removed"}}'
```
