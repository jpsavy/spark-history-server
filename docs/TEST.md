## Test

Two tests are proposed: the first uses a PVC for event-log storage, the second uses SeaweedFS (S3) storage.

Add spark-rbac and spark-operator
## Prerequisites

Install the spark-rbac to get a serviceAccount

```bash
export NAMESPACE=spark
export SPARK_SERVICE_ACCOUNT=spark
helm upgrade --install spark-rbac  oci://quay.io/okdp/charts/spark-rbac \
 --version 1.0.0 \
 --namespace $NAMESPACE --create-namespace \
 --set serviceAccount.name=$SPARK_SERVICE_ACCOUNT
```

Verification:

(the spark serviceAccount exist)
```bash
kubectl get sa -n $NAMESPACE
# NAME                        AGE
# default                     3m
# spark                       3m
```
---

Install the spark-history-server with the serviceAccount

```bash
helm upgrade --install spark-history-server oci://quay.io/okdp/charts/spark-history-server \
  --version 1.0.0 \
  --namespace $NAMESPACE --create-namespace \
  --set serviceAccount.name=$SPARK_SERVICE_ACCOUNT \
  --wait \
  --timeout 10m
```

Expected result:

```
NAME: spark-history-server
LAST DEPLOYED: <timestamp>
NAMESPACE: spark
STATUS: deployed
REVISION: 1
```

> Replace `1.0.0` with the latest chart version from [Releases](https://github.com/OKDP/spark-history-server/releases).

---
Install the spark-operator. It must watch the same namespace where the Spark jobs run (`spark`), otherwise the `SparkApplication` is never reconciled:

```bash
curl -Lo spark-operator.tgz https://github.com/kubeflow/spark-operator/releases/download/v2.5.0/spark-operator-2.5.0.tgz
helm upgrade --install spark-operator ./spark-operator.tgz \
  --set "spark.jobNamespaces={spark}" \
  --set serviceAccount.name=$SPARK_SERVICE_ACCOUNT \
  --namespace spark --create-namespace
```

---

### Test with PVC


#### 1. Create namespace if needed
```bash
kubectl create namespace spark --dry-run=client -o yaml | kubectl apply -f -
```

---

#### 2. Install spark-rbac. Create service account

Create the spark service account:
```bash
export SPARK_SERVICE_ACCOUNT=spark
export NAMESPACE=spark
helm upgrade --install spark-rbac  oci://quay.io/okdp/charts/spark-rbac:1.0.0 \
  --set serviceAccount.name=$SPARK_SERVICE_ACCOUNT \
  -n $NAMESPACE
```

---

#### 3. Create a PVC

<details>
<summary><strong><large>CLICK HERE to retrieve the pvc_spark.yaml manifest CREATE COMMAND</large></strong></summary>
<br>

```sh
cat > ./pvc_spark.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-event-logs
  namespace: spark
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 2Gi
EOF
```

</details>

Click above to retrieve the manifest creation command before applying it.

Create the PVC for the spark-history-server:
```bash
kubectl apply -f ./pvc_spark.yaml
```


---

#### 4. Download and Install spark operator

Install the spark operator so it watches the `spark` namespace (where the `SparkApplication` is deployed):
```bash
curl -Lo spark-operator.tgz https://github.com/kubeflow/spark-operator/releases/download/v2.5.0/spark-operator-2.5.0.tgz

export SPARK_SERVICE_ACCOUNT=spark
export NAMESPACE=spark
helm upgrade --install spark-operator ./spark-operator.tgz \
  --set "spark.jobNamespaces={spark}" \
  --set serviceAccount.name=$SPARK_SERVICE_ACCOUNT \
  --namespace spark --create-namespace
```

---

#### 5. Set the PVC spark history server


<details>
<summary><strong><large>CLICK HERE to retrieve the pvc-values-spark-hs.yaml manifest CREATE COMMAND</large></strong></summary>
<br>

```sh
cat > ./pvc-values-spark-hs.yaml <<'EOF'
extraVolumes:
  - name: event-logs
    persistentVolumeClaim:
      claimName: spark-event-logs

extraVolumeMounts:
  - name: event-logs
    mountPath: /tmp/spark-events-operator

config:
  spark.history.fs.logDirectory: file:/tmp/spark-events-operator
EOF
```

</details>

Click above to retrieve the manifest creation command before applying it.

Set the volumes to PVC for the spark-history-server:
```bash
helm upgrade --install spark-history-server oci://quay.io/okdp/charts/spark-history-server \
  -f ./pvc-values-spark-hs.yaml \
  --set serviceAccount.name=$SPARK_SERVICE_ACCOUNT \
  --version 1.0.0 \
  --namespace $NAMESPACE
```

---

#### 6. Deploy the Spark Pi spark operator


<details>
<summary><strong><large>CLICK TO GET THE spark-okdp-pi-for-spark-hs.yaml manifest CREATE COMMAND</large></strong></summary>
<br>

```sh
cat > ./spark-okdp-pi-for-spark-hs.yaml <<'EOF'
    apiVersion: sparkoperator.k8s.io/v1beta2
    kind: SparkApplication
    metadata:
      name: spark-pi
      namespace: spark
    spec:
      type: Scala
      mode: cluster
      image: "quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.12-java-17"
      imagePullPolicy: Always
      mainClass: org.apache.spark.examples.SparkPi
      mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.5.6.jar"
      sparkVersion: "3.5.6"
      restartPolicy:
        type: Never
      sparkConf:
        spark.kubernetes.authenticate.driver.serviceAccountName: spark
        spark.eventLog.enabled: "true"
        spark.eventLog.dir: "file:///tmp/spark-events"
        spark.history.fs.logDirectory: file:///tmp/spark-events
      volumes:
        - name: event-logs
          persistentVolumeClaim:
            claimName: spark-event-logs
      driver:
        cores: 1
        coreLimit: "1200m"
        memory: "512m"
        serviceAccount: spark
        volumeMounts:
          - name: event-logs
            mountPath: /tmp/spark-events
      executor:
        cores: 1
        instances: 2
        memory: "512m"
        volumeMounts:
          - name: event-logs
            mountPath: /tmp/spark-events
EOF
```

</details>

Click above to retrieve the manifest creation command before applying it.

```bash
kubectl apply -f spark-okdp-pi-for-spark-hs.yaml
```

Expected result:
```log
<expected result>
```

---

#### 7. Consult Jobs list in Spark History Server.

Forward the 18080 port:
```bash
kubectl port-forward svc/spark-history-server 18080:18080 -n spark &
```

Consult the Job list
```bash
curl -i http://localhost:18080/api/v1/applications
```

Expected result
```log
[ {
  "id" : "<spark-job-id-1>",
  "name" : "Spark Pi",
  "attempts" : [ {
    "startTime" : "<old-start-timestamp>",
    "endTime" : "<end-timestamp>",
    "lastUpdated" : "<update-start-timestamp>",
    "duration" : 7415,
    "sparkUser" : "spark",
    "completed" : true,
    "appSparkVersion" : "3.5.6",
    "startTimeEpoch" : <start-epochtimestamp>,
    "endTimeEpoch" : <end-epochtimestamp>,
    "lastUpdatedEpoch" : <update-epochtimestamp>
  } ]
}, {
  "id" : "<spark-id-2>",
  "name" : "Spark Pi",
  "attempts" : [ {
    "startTime" : "<start-timestamp>",
    "endTime" : "<end-timestamp>",
    "lastUpdated" : "<update-start-timestamp>",
    "duration" : 7415,
    "sparkUser" : "spark",
    "completed" : true,
    "appSparkVersion" : "3.5.6",
    "startTimeEpoch" : <start-epochtimestamp>,
    "endTimeEpoch" : <end-epochtimestamp>,
    "lastUpdatedEpoch" : <update-epochtimestamp>
  } ]
} ]

```

To activate the Job again
```sh
kubectl delete -f ./spark-okdp-pi-for-spark-hs.yaml
kubectl apply -f ./spark-okdp-pi-for-spark-hs.yaml
```

---

### Test with SeaweedFS

Deploy a Spark Pi Application.

Create the following files. Click on each file to get the file creation command:

<details>
<summary><strong><large>01-creds-seaweedfs-s3.secret.yaml</small></strong></summary>
<br>

```bash
apiVersion: v1
kind: Secret
metadata:
  name: creds-seaweedfs-s3
  namespace: spark
type: Opaque
stringData:
  accessKey: <seaweedfs-access-key>
  secretKey: <seaweedfs-secret-key>
```

</details>

<details>
<summary><strong><large>02-creds-spark-history-s3.secret.yaml</small></strong></summary>
<br>

```bash
apiVersion: v1
kind: Secret
metadata:
  name: creds-spark-history-s3
  namespace: spark
type: Opaque
stringData:
  accessKey: <spark-history-access-key>
  secretKey: <spark-history-secret-key>
```

</details>

<details>
<summary><strong><large>03-seaweedfs-auth-config-values.yaml</small></strong></summary>
<br>

```bash
filer:
  auth:
    method: basic
    basic:
      create_auth_secret: creds-seaweedfs-filer-basic
      username: admin
      password: admin123

iamConfig:
  policies:
    - name: S3WritePolicy
      document:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - s3:List*
              - s3:Get*
              - s3:Put*
              - s3:DeleteObject
            Resource:
              - "*"
  roles:
    - roleName: S3WriteRole
      roleArn: arn:aws:iam::000000000000:role/S3WriteRole
      attachedPolicies:
        - S3WritePolicy
      trustPolicy:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: "*"
            Action:
              - sts:AssumeRole
  sts:
    issuer: seaweedfs-sts
    tokenDuration: 1h
    maxSessionLength: 12h
    signingKey: ""
  providers:
    oidc:
      enabled: false

s3Config:
  enabled: true
  secretName: creds-seaweedfs-identities
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/resource-policy: keep
  identities:
    - name: seaweedfs
      actions:
        - Admin
      credentialsSecret:
        name: creds-seaweedfs-s3
        accessKeyKey: accessKey
        secretKeyKey: secretKey
    - name: sparkHistory
      actions:
        - Read
        - Write
        - List
      credentialsSecret:
        name: creds-spark-history-s3
        accessKeyKey: accessKey
        secretKeyKey: secretKey
```

</details>

<details>
<summary><strong><large>04-seaweedfs-values.yaml</small></strong></summary>
<br>

```bash
fullnameOverride: seaweedfs
nameOverride: seaweedfs

global:
  imageName: chrislusf/seaweedfs

image:
  tag: "4.17"

master:
  enabled: true
  replicas: 1
  defaultReplication: "000"
  volumeSizeLimitMB: 1000
  data:
    type: persistentVolumeClaim
    storageClass: standard
    size: 1Gi
  logs:
    type: persistentVolumeClaim
    storageClass: standard
    size: 0.5Gi

volume:
  enabled: true
  replicas: 1
  dataDirs:
    - name: data
      type: persistentVolumeClaim
      storageClass: standard
      size: 2Gi
      maxVolumes: 0

filer:
  enabled: true
  replicas: 1
  defaultReplicaPlacement: "000"
  enablePVC: true
  data:
    type: persistentVolumeClaim
    storageClass: standard
    size: 0.5Gi
  logs:
    type: persistentVolumeClaim
    storageClass: standard
    size: 0.5Gi
  extraEnvironmentVars:
    WEED_FILER_BUCKETS_FOLDER: /buckets
    WEED_FILER_OPTIONS_RECURSIVE_DELETE: "false"
    WEED_LEVELDB2_ENABLED: "true"
  ingress:
    enabled: true
    className: nginx
    host: seaweedfs-console.okdp.sandbox
    path: /
    pathType: Prefix
    annotations:
      nginx.ingress.kubernetes.io/auth-realm: Authentication Required
      nginx.ingress.kubernetes.io/auth-secret: creds-seaweedfs-filer-basic
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/proxy-body-size: 130m
    tls: []

s3:
  enabled: true
  enableAuth: true
  existingConfigSecret: creds-seaweedfs-identities
  port: 8333
  httpsPort: 0
  createBuckets:
    - name: spark-events
  extraArgs:
    - -iam.config=/etc/seaweed/iam/iam.json
  extraVolumeMounts: |
    - name: auth-config
      mountPath: /etc/seaweed/iam
      readOnly: true
  extraVolumes: |
    - name: auth-config
      configMap:
        name: seaweedfs-auth-config
  ingress:
    enabled: true
    className: nginx
    host: seaweedfs.okdp.sandbox
    path: /
    pathType: Prefix
    annotations:
      nginx.ingress.kubernetes.io/proxy-body-size: 130m
    tls: []

worker:
  enabled: false

admin:
  enabled: false

sftp:
  enabled: false

allInOne:
  enabled: false
```

</details>

<details>
<summary><strong><large>05-create-spark-event-log-dir.job.yaml</small></strong></summary>
<br>

```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: create-spark-event-log-dir
  namespace: spark
spec:
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: create-dir
          image: chrislusf/seaweedfs:4.17
          imagePullPolicy: IfNotPresent
          env:
            - name: WEED_CLUSTER_DEFAULT
              value: sw
            - name: WEED_CLUSTER_SW_MASTER
              value: seaweedfs-master.spark:9333
            - name: WEED_CLUSTER_SW_FILER
              value: seaweedfs-filer-client.spark:8888
          command:
            - /bin/sh
            - -ec
            - |
              echo "Waiting for SeaweedFS filer..."
              until wget -q --spider http://seaweedfs-filer-client.spark:8888/; do
                sleep 5
              done

              echo "Creating Spark event log directory..."
              echo 'fs.mkdir /buckets/spark-events/event-logs' | /usr/bin/weed shell
              curl -fsS -X PUT --data-binary '' \
                http://seaweedfs-filer-client.spark:8888/buckets/spark-events/event-logs/.keep
              echo 'fs.ls /buckets/spark-events' | /usr/bin/weed shell
```

</details>

<details>
<summary><strong><large>06-spark-history-server-values.yaml</small></strong></summary>
<br>

```bash
image:
  repository: quay.io/okdp/spark
  pullPolicy: IfNotPresent
  tag: spark-3.5.6-scala-2.12-java-17

serviceAccount:
  create: false
  name: default

ingress:
  enabled: true
  ingressClassName: nginx
  hosts:
    - host: spark-history.okdp.sandbox
      paths:
        - path: /
          pathType: Prefix
  annotations: {}
  labels: {}
  tls: []

config:
  spark.history.provider: org.apache.spark.deploy.history.FsHistoryProvider
  spark.history.fs.logDirectory: s3a://spark-events/event-logs/
  spark.history.fs.update.interval: 60s
  spark.history.retainedApplications: 50
  spark.history.ui.maxApplications: 2147483647
  spark.history.ui.port: 18080
  spark.history.kerberos.enabled: false
  spark.history.kerberos.principal: ""
  spark.history.kerberos.keytab: ""
  spark.history.fs.cleaner.enabled: true
  spark.history.fs.cleaner.interval: 1d
  spark.history.fs.cleaner.maxAge: 7d
  spark.history.fs.cleaner.maxNum: 2147483647
  spark.history.fs.endEventReparseChunkSize: 1m
  spark.history.fs.inProgressOptimization.enabled: true
  spark.history.fs.driverlog.cleaner.enabled: spark.history.fs.cleaner.enabled
  spark.history.fs.driverlog.cleaner.interval: spark.history.fs.cleaner.interval
  spark.history.fs.driverlog.cleaner.maxAge: spark.history.fs.cleaner.maxAge
  spark.history.store.maxDiskUsage: 10g
  spark.history.store.path: ""
  spark.history.store.serializer: JSON
  spark.history.custom.executor.log.url: ""
  spark.history.custom.executor.log.url.applyIncompleteApplication: true
  spark.history.fs.eventLog.rolling.maxFilesToRetain: 5
  spark.history.store.hybridStore.enabled: false
  spark.history.store.hybridStore.maxMemoryUsage: 2g
  spark.history.store.hybridStore.diskBackend: ROCKSDB
  spark.history.fs.update.batchSize: 2147483647
  spark.hadoop.fs.s3a.endpoint: http://seaweedfs-s3.spark.svc.cluster.local:8333
  spark.hadoop.fs.s3a.connection.ssl.enabled: false
  spark.hadoop.fs.s3a.path.style.access: true
  spark.hadoop.fs.s3a.impl: org.apache.hadoop.fs.s3a.S3AFileSystem
  spark.hadoop.fs.s3a.aws.credentials.provider: com.amazonaws.auth.EnvironmentVariableCredentialsProvider

extraEnvs:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: creds-spark-history-s3
        key: accessKey
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: creds-spark-history-s3
        key: secretKey

podSecurityContext:
  seccompProfile:
    type: RuntimeDefault

containerSecurityContext:
  runAsUser: 185
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  capabilities:
    drop:
      - ALL
```

</details>


#### 1. Create namespace if needed

```bash
kubectl create namespace spark --dry-run=client -o yaml | kubectl apply -f -
```

#### 2. Create necessary S3 secrets

```bash
kubectl apply -f 01-creds-seaweedfs-s3.secret.yaml
kubectl apply -f 02-creds-spark-history-s3.secret.yaml
```

#### 3. Install SeaweedFS authentication configuration

```bash
helm install seaweedfs-auth-config oci://quay.io/okdp/charts/seaweedfs-auth-config --version 1.0.0 \
  --namespace spark \
  --values 03-seaweedfs-auth-config-values.yaml
```

This step generates, in particular:

| Object | Role |
| --- | --- |
| `creds-seaweedfs-identities` | Secret read by the SeaweedFS S3 gateway |
| `seaweedfs-auth-config` | ConfigMap IAM mounted in SeaweedFS |
| `creds-seaweedfs-filer-basic` | Secret basic auth for the Filer console |

#### 4. Install SeaweedFS

```bash
curl -L -o ./seaweedfs-4.17.0.tgz https://seaweedfs.github.io/seaweedfs/helm/seaweedfs-4.17.0.tgz
```

```bash
helm upgrade --install seaweedfs ./seaweedfs-4.17.0.tgz \
  --namespace spark \
  --values 04-seaweedfs-values.yaml \
  --wait \
  --timeout 10m
```

#### 5. Create logs Spark Directory in SeaweedFS

```bash
kubectl apply -f 05-create-spark-event-log-dir.job.yaml
kubectl wait --for=condition=complete job/create-spark-event-log-dir -n spark --timeout=5m
```

This step is important: without an object in the S3 prefix `event-logs/`, Spark History Server may consider that the folder `s3a://spark-events/event-logs/` does not exist.


#### 6. Install Spark History Server with SeaweedFS values

```bash
helm upgrade --install spark-history-server oci://quay.io/okdp/charts/spark-history-server --version 1.0.0 \
  --namespace spark \
  --values 06-spark-history-server-values.yaml \
  --wait \
  --timeout 10m
```
If the installation fails with a message such as `host "...okdp.sandbox" and path "/" are already defined`, it means that another entry point is already using the same URL in the cluster. In this case, remove the old entry point or modify the host in the corresponding values file before rerunning the command.

#### 7. Start the Spark Pi Job

Set the job manifest
- spark configuration : 
  - logDir for the jobs to S3 bundle
  - serviceAccount for Spark application
  - S3 SeaweedFS endpoint
- env driver
  - serviceAccount for Spark application
  - AWS_ACCESS_KEY_ID from creds-spark-history-s3 accessKey
  - AWS_SECRET_ACCESS_KEY from creds-spark-history-s3 secretKey
- env executor
  - AWS_ACCESS_KEY_ID from creds-spark-history-s3 accessKey
  - AWS_SECRET_ACCESS_KEY from creds-spark-history-s3 secretKey


<details>
<summary><strong><large>CLICK TO GET THE spark-s3-okdp-pi-for-spark-hs.yaml CREATE COMMAND</large></strong></summary>
<br>

```sh
cat > ./spark-s3-okdp-pi-for-spark-hs.yaml <<'EOF'
    apiVersion: sparkoperator.k8s.io/v1beta2
    kind: SparkApplication
    metadata:
      name: spark-pi
      namespace: spark
    spec:
      type: Scala
      mode: cluster
      image: "quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.12-java-17"
      imagePullPolicy: Always
      mainClass: org.apache.spark.examples.SparkPi
      mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.5.6.jar"
      sparkVersion: "3.5.6"
      restartPolicy:
        type: Never
      sparkConf:
        spark.kubernetes.authenticate.driver.serviceAccountName: spark
        spark.eventLog.enabled: "true"
        spark.eventLog.dir: s3a://spark-events/event-logs/
        spark.hadoop.fs.s3a.endpoint: http://seaweedfs-s3.spark.svc.cluster.local:8333
        spark.hadoop.fs.s3a.path.style.access: "true"
        spark.hadoop.fs.s3a.connection.ssl.enabled: "false"
      driver:
        cores: 1
        coreLimit: "1200m"
        memory: "512m"
        serviceAccount: spark
        env:
          - name: AWS_ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: creds-spark-history-s3
                key: accessKey
          - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: creds-spark-history-s3
                key: secretKey
      executor:
        cores: 1
        instances: 2
        memory: "512m"
        env:
          - name: AWS_ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: creds-spark-history-s3
                key: accessKey
          - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: creds-spark-history-s3
                key: secretKey
EOF
```

</details>

Click above to retrieve the manifest creation command before applying it.
```sh
kubectl apply -f ./spark-s3-okdp-pi-for-spark-hs.yaml
```

After running job, check the finished jobs list in spark-history-server:

```sh
kubectl port-forward svc/spark-history-server 18080:18080 -n spark &
curl http://localhost:18080/api/v1/applications
```

Expected result:
jobs list
```log
[ {
  "id" : "<spark-job-id>",
  "name" : "Spark Pi",
  "attempts" : [ {
    "startTime" : "<start-timestamp>",
    "endTime" : "<end-timestamp>",
    "lastUpdated" : "<update-start-timestamp>",
    "duration" : 7415,
    "sparkUser" : "spark",
    "completed" : true,
    "appSparkVersion" : "3.5.6",
    "startTimeEpoch" : <start-epochtimestamp>,
    "endTimeEpoch" : <end-epochtimestamp>,
    "lastUpdatedEpoch" : <update-epochtimestamp>
  } ]
} ]
```

If you want to apply the Job again
```sh
kubectl delete -f ./spark-s3-okdp-pi-for-spark-hs.yaml
```
