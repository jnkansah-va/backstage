# Installation Guide

## Getting Started

### Configuring Chart.yaml

The latest stable version can be found [here](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/tree/main/charts/dx-universal). Feel free to update when your team is ready.

### Configuring values.yaml

#### Templatable Values

The ability to reference other Helm values in your `values.yaml` is possible with the following resources:

| resource | field |
| --- | ---|
| argoRollouts | `argoRollouts.blueGreen.analysis.prePromotion.templates` |
| argoRollouts | `argoRollouts.blueGreen.analysis.prePromotion.args` |
| argoRollouts | `argoRollouts.blueGreen.analysis.postPromotion.templates` |
| argoRollouts | `argoRollouts.blueGreen.analysis.postPromotion.args` |
| configMap | `configMaps.create.<element>` |
| cronjob | `cronJob.jobs[].env` |
| cronjob | `cronJob.jobs[].command` |
| datadog | `datadog.service` |
| datadog | `datadog.env` |
| datadog | `datadog.version` |
| deployment | `deployment.extraInitContainers` |
| deployment | `deployment.command` |
| deployment | `deployment.args` |
| deployment | `deployment.env` |
| deployment | `deployment.extraContainers` |
| deployment | `deployment.imagePullSecrets` |
| job | `job.extraInitContainers` |
| job | `job.command` |
| job | `job.args` |
| job | `job.env` |
| job | `job.extraContainers` |
| job | `job.imagePullSecrets` |
| vault | `vault.secrets[].path` |
| vault | `vault.secrets[].kvVersion` |
| vault | `vault.secrets[].data` |

**Note**: Ensure any referenced template values are surrounded in double quotes - ie.,

```yaml
dx-universal:
  environment: dev
  deployment:
    env:
      DEPLOY_ENVIRONMENT:
        value: "{{ .Values.environment }}"
  datadog:
    env: "{{ .Values.environment }}"
```

---

### Secret Creation

The universal helm chart can create K8s secrets with values from [Vault](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/main/charts/dx-universal/values.yaml#L81-L85) for you

eg.,

```yaml
dx-universal:
  vault:
    secrets:
      elastic-cache-auth:
        path: "lhdi-e2e-tests-dev/data/redis"
        data:
          ec-auth: <password>
      postgres-auth:
        path: "lhdi-e2e-tests-dev/data/postgres"
        data:
          password: <mypassword>
```

will create two K8s secrets:

1. elastic-cache-auth
2. postgres-auth

with the following configuration:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elastic-cache-auth
  annotations:
    avp.kubernetes.io/kv-version: "2"
    avp.kubernetes.io/path: lhdi-e2e-tests-dev/data/redis
data:
  ec-auth: redacted # injected value of <password>
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-auth
  annotations:
    avp.kubernetes.io/kv-version: "2"
    avp.kubernetes.io/path: lhdi-e2e-tests-dev/data/postgres
data:
  password: redacted # injected value of <password>
```

Vault secret creation can also use templated values, ie.,

```yaml
dx-universal:
  vault:
    secrets:
      frontend-{{ .Values.environment }}-secret:
        path: lighthouse-di-enablement/data/{{ .Values.environment }}
        data:
          someValue: <someSecret>
```

#### Creating Image Pull Secrets

An image pull secret will be **REQUIRED** if your team is hosting container images on `ghcr.io`. The following example is how an image pull secret could be created based off of a value in Vault:

```yaml
dx-universal:
  vault:
    secrets:
      github-secret-frontend:
        type: kubernetes.io/dockerconfigjson
        path: "lighthouse-di-enablement/data/github"
        data:
          .dockerconfigjson: <pat>
```

---

### Enabling AWS resources

By default, creation of LHDI AWS supported resources are *disabled* by default.

#### Enabling RDS

RDS can be enabled by defining an `rds` object under your `dx-universal` values.yaml.

The following values are **REQUIRED**:

```yaml
dx-universal:
  rds:
    # allocatedStorage - base storage allocated to your RDS instance.
    allocatedStorage: 25
    # autoScaleSize - max size your RDS instance will scale to.
    autoScaleSize: 100
    # dbInstanceIdentifier - the name of the RDS instance as it will appear in AWS (not Kubernetes!).
    dbInstanceIdentifier: ""
    # engine - can be postgres, mssql, etc.
    engine: ""
    # secretPasswordKey - the key in the K8s secret in which your database password is defined.
    secretPasswordKey: foo
    # secretPasswordName - the name of the K8s secret in which your database password exists.
    secretPasswordName: bar
    # masterUserName - user-provided name of admin user.
    masterUsername: dbAdmin
```

The following values are **OPTIONAL**, and will use AWS defaults:

```yaml
dx-universal:
  rds:
    autoMinorVersionUpgrade: true
    dbName: ""
    deletionProtection: true
    engineVersion: ""
    monitoringInterval: 0
    performanceInsightsEnabled: false
    preferredBackupWindow: ""
    preferredMaintenanceWindow: ""
    retentionPeriod: ""
    port: 0
```

##### Other AWS values

- [sns](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/v0.1.27/charts/dx-universal/values.yaml#L306-L346)
  - *important* only `topic` is currently supported
- [elasticache](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/v0.1.27/charts/dx-universal/values.yaml#L226-L252)

### Enabled field exports

[Field exports](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/main/charts/dx-universal/values.yaml#L87-L93) are a way to programmatically gather metadata of various AWS-CRD's without manually inspecting the resource.

By enabling field exports, the following fields will be automatically exposed as `environment variables` to the `deployment.yaml` for you (depending on which AWS-resources you have enabled):

```yaml
REDIS_HOST: master.redis-identifier.koxxmv.usgw1.cache.amazonaws.com
REDIS_PORT: "6379"
S3_BUCKET_LOCATION: http://<bucket>.s3-us-gov-west-1.amazonaws.com/
S3_BUCKET_NAME: <bucket name as it exists in aws>
```

*important* - RDS host & port are also exported, though there is a bug in the field export where the key that gets populated in the configmap/secret is incorrect.

The [name](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/main/charts/dx-universal/values.yaml#L91) of the fieldexport `configmap` or `secret` is configurable, it will default to the release name.

### Enabling virtual services

The creation and configuration of [Virtual services](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/main/charts/dx-universal/values.yaml#L140-L151) are supported by the universal helm chart.

ex., virtual service configuration

```yaml
  istio:
    virtualService:
      enabled: true
      gateways:
        - istio-system/ldx-nonprod-1-dev-gateway
      hosts:
        - dev.lighthouse.va.gov
      uri: /playground/fastapi/
      rewrite: "/"
```

---

## Enabling argo rollouts

Argo rollouts are enabled by default, though the creation and definition of `analysisTemplates` and `prePromotion` / `postPromotion` analysis are also supported.

ex., analysis template configuration

```yaml
  argoRollouts:
    blueGreen:
      analysisTemplates:
        httpresp:
          args:
            - name: host
            - name: iaas-route
            - name: redis-route
            - name: s3-route
            - name: rds-route
            - name: release-name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          metrics:
            - name: iaas-health
              successCondition: 'true'
              provider:
                web:
                  insecure: true
                  url: "https://{{ args.host }}{{ args.iaas-route }}"
            - name: redis-health
              successCondition: 'true'
              provider:
                web:
                  insecure: true
                  url: "https://{{ args.host }}{{ args.redis-route }}"
            - name: s3-health
              successCondition: 'true'
              provider:
                web:
                  insecure: true
                  url: "https://{{ args.host }}{{ args.s3-route }}{{ args.release-name }}-bucket"
            - name: rds-health
              successCondition: 'true'
              provider:
                web:
                  insecure: true
                  url: "https://{{ args.host }}{{ args.rds-route }}"
```

ex., analysis configuration

```yaml
      analysis:
        prePromotion:
          enabled: true
          templates:
            - templateName: httpresp
          args:
            - name: host
              value: dev.lighthouse.va.gov
            - name: iaas-route
              value: /preview/playground/fastapi/iaas/health
            - name: redis-route
              value: /preview/playground/fastapi/redis/health
            - name: s3-route
              value: /preview/playground/fastapi/s3/list/
            - name: rds-route
              value: /preview/playground/fastapi/rds/
            - name: release-name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
        postPromotion:
          enabled: true
          templates:
            - templateName: httpresp
          args:
            - name: host
              value: dev.lighthouse.va.gov
            - name: iaas-route
              value: /playground/fastapi/iaas/health
            - name: redis-route
              value: /playground/fastapi/redis/health
            - name: s3-route
              value: /playground/fastapi/s3/list/
            - name: rds-route
              value: /playground/fastapi/rds/
            - name: release-name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
```

full argo rollouts values available [here](https://github.com/department-of-veterans-affairs/lighthouse-di-registry/blob/main/charts/dx-universal/values.yaml#L153-L203)

### Enabling ConfigMaps

ConfigMap creation is disabled by default. To create configMaps via the universal helm chart, do the following:

```yaml
# values.yaml
universal:
  ...
  configMaps:
    enabled: true
```

```yaml
# values.yaml
universal:
  ...
  configMaps:
    enabled: true
    # if required, define extraLabels (not needed)
    extraLabels: {}
    # if required, define extraAnnotations (not needed)
    extraAnnotations: {}
    create:
      test-a:                   # creates configmap with name "test-a"
        script.sh: |
          #!/bin/bash
          echo "hello world"
        some-var: foo
      test-b:                   # creates configmap with name "test-b"
        some-var: foo
        some-var-two: bar
```

### Enabling PersistentVolumesClaim

PersistentVolumeClaims are disabled by default. To create a PVC via the universal helm chart, do the following:

```yaml
# values.yaml - use default settings (5Gi PVC, ReadWriteOnce, default storageClass)
universal:
  ...
  persistence:
    enabled: true
```

```yaml
# values.yaml - override defaults
universal:
  ...
  persistence:
    enabled: true
    name: "my-persistent-volume-claim"      # name override - will use default release name if undefined.
    accessMode: "ReadWriteMany"             # accessMode override - will default to "ReadWriteOnce" if undefined.
    storageSize: 100Gi                      # size of volume - will default to 5Gi if undefined.
    storageClass: ebs-sc-retain             # storage class to use - will default to default LHDI cluster storageClass if undefined.
```

### Adding extraInitContainers & extraContainers

The universal helm chart allows developers to add `extraContainers` and `extraInitContainers` to:

- deployments
- job

#### To deployment

All values definable in a container can be added to the `extraContainers` key, though a base example is as follows:

```yaml
universal:
  ...
  deployment:
    image:
      repository: alpine
      tag: latest
    command: ['sh', '-c', 'sleep 1d']
    extraContainers:
      b-container:
        image: nginx
        imagePullPolicy: IfNotPresent
```

Adding extraInitContainers to a deployment follows the same pattern as adding `extraContainers`, ie.,

```yaml
universal:
  ...
  deployment:
    image:
      repository: alpine
      tag: latest
    extraInitContainers:
      a-container:
        image: nginx
        imagePullPolicy: IfNotPresent
```

##### To job

Adding extraContaineres/extraInitContainers to a job follows the same pattern as noted above, ie.,

```yaml
universal:
  ...
  job:
    enabled: true
    image:
      repository: alpine
      tag: latest
    command: ['sh', '-c', 'sleep 1d']
    extraInitContainers:
      a-container:
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ['/bin/sh', '-c', 'exit 0']
    extraContainers:
      b-container:
        image: nginx
        imagePullPolicy: IfNotPresent
```

### Enabling cronJobs

Cronjobs can be created by the universal helm chart by doing the following:

```yaml
# values.yaml
universal:
  ...
  cronJob:
    enabled: true
    jobs:
      test-cronjob-a:                             # name of cronjob resources
        schedule: "* * * 8 *"
        image:
          repository: docker.io/nginx
          tag: v1.0.0
        command: ["/bin/bash"]
        args: ["-c", "sleep 5000"]
        resources:
          requests:
            memory: 128Mi
            cpu: 1
      test-cronjob-b:                             # name of cronjob resources
        schedule: "* * * 8 *"
        env:
          foo:
            value: VALUE
        image:
          repository: alpine
          tag: latest
        command: ["/bin/bash"]
        args: ["-c", "sleep 5000"]
```

### Adding env and envFrom to deployment

```yaml
# values.yaml
universal:
  ...
  deployment:
    env:
      foo:
        value: bar
    envFrom:
      - configMapRef:
          name: test
```

### Adding env and envFrom To cronJob

```yaml
# values.yaml
universal:
  ...
  cronJob:
    enabled: true
    test-cronjob:
      env:
        KEY:
          value: VALUE
      envFrom:
      - configMapRef:
          name: test
```

### Adding env and envFrom  To job

```yaml
# values.yaml
universal:
  ...
  job:
    enabled: true
    env:
      - name: foo
        value: bar
```

## Job Template Configuration

The universal Helm chart supports defining one or more Kubernetes Jobs using the `jobs:` key in your `values.yaml`. Whether you need a single Job or multiple Jobs, the same configuration pattern applies—simply list your Jobs under `jobs:`.

### Defining Jobs in `values.yaml`

Below is an example of two Jobs. If you only need one, include a single item in the list.

```yaml
  jobs:
    - name: data-processing-job-example
      image:
        repository: alpine
        tag: latest
        pullPolicy: IfNotPresent
      command: ["sh", "-c", "echo 'Processing data' && sleep 10"]
      backoffLimit: 3
      completions: 1
      parallelism: 1
      ttlSecondsAfterFinished: 30
      env:
        - name: JOB_TYPE
          value: "Data Processing"

    - name: report-generation-job-example
      image:
        repository: busybox
        tag: latest
        pullPolicy: IfNotPresent
      command: ["sh", "-c", "echo 'Generating reports' && sleep 5"]
      backoffLimit: 5
      completions: 2
      parallelism: 1
      ttlSecondsAfterFinished: 30
      env:
        - name: JOB_TYPE
          value: "Report Generation"
```

Manifest Reference: values.yaml

This template iterates over each entry in *.Values.jobs* and creates a corresponding Kubernetes Job.

#### Enabling the jobs

- To enable a Job, set ***jobs.enabled: true*** in your values.yaml

#### Resource limits

- To ensure proper resource allocation, specify resource requests and limits:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

#### Using Secrets & ConfigMaps

- You can reference Secrets or ConfigMaps to provide additional environment variables:

```yaml
envFrom:
  - secretRef:
      name: my-secret
  - configMapRef:
      name: my-configmap
```

## Adding volumes to deployment

Adding volumes can be done through the universal helm chart for any applicable K8s resources (ie., deployments, cronjobs, jobs)

```yaml
# values.yaml
universal:
  deployment:
    ...
    volumes:
    - name: configmap
      configMap:
        name: test-a
```

## Adding volumes to cronJob

```yaml
# values.yaml
universal:
  ...
  cronJob:
    enabled: true
    volumes:
    - name: configmap
      configMap:
        name: test-a
```

## Adding volumes to job

```yaml
# values.yaml
universal:
  ...
  job:
    enabled: true
    volumes:
    - name: configmap
      configMap:
        name: test-a
```

## Adding volumeMounts to deployments

Adding volumemounts can be done through the universal helm chart for any applicable K8s resources (ie., deployments, cronjobs, jobs) and their applicable counterparts (init containers, extra containers, etc.)

```yaml
# values.yaml
universal:
  ...
  deployment:
    volumeMounts:
    - name: configmap
      mountPath: /tmp
    extraInitContainers:
      a-container:
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ['/bin/sh', '-c', 'exit 0']
        volumeMounts:
        - name: configmap
          mountPath: /tmp
    extraContainers:
      b-container:
        image: nginx
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: configmap
          mountPath: /tmp
```

## Adding volumeMounts to cronjob

```yaml
# values.yaml
universal:
  ...
  cronJob:
    enabled: true
    jobs:
      db-migration:
        ...
        volumes:
        - name: configmap
          configMap:
            name: test-a
        volumeMounts:
        - name: configmap
          mountPath: /tmp
```

## Adding volumeMounts to jobs

```yaml
# values.yaml
universal:
  job:
    enabled: true
    ...
    volumes:
    - name: configmap
      configMap:
        name: test-a
    volumeMounts:
    - name: configmap
      mountPath: /tmp
    extraInitContainers:
      a-container:
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ['/bin/sh', '-c', 'exit 0']
        volumeMounts:
        - name: configmap
          mountPath: /tmp
```
