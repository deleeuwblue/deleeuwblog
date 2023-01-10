---
title: Deployments to Kubernetes using KServe Modelmesh
date: 2023-01-06 09:00:00 +/-0000
categories: [IBM Watson for Embed, NLP]
tags: [ai, nlp, kubernetes, kserve]     # TAG names should always be lowercase
---

In this blog, I will demonstrate how to deploy the Watson for NLP Library to OpenShift using  KServe Modelmesh.

For initial context, read my blog [introducing IBM Watson for Embed]({{ site.baseurl }}/Introducing-IBM-Watson-for-Embed)

## Introducing KServe

[KServe](https://github.com/kserve/kserve) is a standard model inference platform on k8s.  It is built for highly scalable use cases and supports existing thrid party model servers and standard ML/DL model formats, or it can be extended to support additional runtimes like the Watson NLP runtime.

[Modelmesh Serving](https://github.com/kserve/modelmesh-serving) is intended to further increase KServe's scalability, especially when there are a large number of models which change frequently.  It intelligently loads and unloads models into memory from from cloud object storage (COS), to strike a trade-off between responsiveness to users and computational footprint.

## Install Kserve Modelmesh on a Kubernetes or OpenShift cluster

KServe Modelmesh requires etcd, S3 storage and optionally Knative and Istio.

Two approaches are available for installation:

* A [quick start approach](https://github.com/kserve/modelmesh-serving/blob/release-0.9/docs/quickstart.md) which includes all the pre-reqs, i.e. Etcd and even local COS with minIO.
* A [customizable approach](https://github.com/kserve/modelmesh-serving/blob/release-0.9/docs/install/install-script.md) which requires Etcd to be already installed.

I took the quick start approach and installed to an OpenShift cluster with the following commands:

```sh
RELEASE=release-0.9
git clone -b $RELEASE --depth 1 --single-branch https://github.com/kserve/modelmesh-serving.git
cd modelmesh-serving
oc new-project modelmesh-serving
```

The quickstart at `release-0.9` currently has limitations meaning the Etcd and Minio pods will crash at startup on OpenShift (although it works fine on Kubernetes).  This can be resolved by editing the `etcd` and `minio` Deployments in `/config/dependencies/quickstart.yaml`

For `etcd`, specify an alternative `--data-dir` shown in the last two lines below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: etcd
  name: etcd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
        - command:
            - etcd
            - --listen-client-urls
            - http://0.0.0.0:2379
            - --advertise-client-urls
            - http://0.0.0.0:2379
            - --data-dir
            - /tmp/etcd.data
```

For `minio`, chat the `-data1` to `-/tmp/data1` as shown in the last line below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: minio
  name: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - args:
            - server
            - /tmp/data1
```

Now run the quickstart:

```
./scripts/install.sh --namespace modelmesh-serving --quickstart
```

After the script completes, you will find the et components running:

```sh
oc get pods

NAME                                    READY   STATUS    RESTARTS   AGE
etcd                                    1/1     Running   0          76m
minio                                   1/1     Running   0          76m
modelmesh-controller-77b8bf999c-2knhf   1/1     Running   0          75m
```

By default, there are some model inference services defined:

```sh
oc get servingruntimes

NAME           DISABLED   MODELTYPE     CONTAINERS   AGE
mlserver-0.x              sklearn       mlserver     4m11s
ovms-1.x                  openvino_ir   ovms         4m11s
triton-2.x                keras         triton       4m11s
```

The quick start installation has also created a secret with credentials for the local minIO object storage.

```sh
oc get secret/storage-config -n modelmesh-serving       

NAME             TYPE     DATA   AGE
storage-config   Opaque   1      117m
```

The secret could contain multiple connection details, but this secret has only a section for "localMinIO".  This secret becomes important later when defining the models which can be served.

curl https://github.com/minio/operator/releases/download/v4.5.6/kubectl-minio_4.5.6_linux_arm64 -o kubectl-minio
chmod +x kubectl-minio
sudo mv kubectl-minio /usr/local/bin/

Use finder to permit iOS to open the file: https://support.apple.com/en-gb/guide/mac-help/mchleab3a043/mac

create route

oc project openshift-operators
oc get secret $(oc get serviceaccount console-sa -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode

note that zsh adds a termination character '%' so discard that.

use JWT to login



## Configure a ServingRuntime for Watson NLP

An example [Serving Runtime resource](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/servingruntime.yaml) is provided.  Note the serving runtime defines the `cp.icr.io/cp/ai/watson-nlp-runtime` container image should be used to serve models that specify `watson-nlp` as their model format.

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: watson-nlp-runtime
spec:
  containers:
  - env:
      - name: ACCEPT_LICENSE
        value: "true"
      - name: LOG_LEVEL
        value: info
      - name: CAPACITY
        value: "6000000000"
      - name: DEFAULT_MODEL_SIZE
        value: "500000000"
      - name: METRICS_PORT
        value: "2113"
    args:
      - --  
      - python3
      - -m
      - watson_runtime.grpc_server
    image: cp.icr.io/cp/ai/watson-nlp-runtime:1.0.20
    imagePullPolicy: IfNotPresent
    name: watson-nlp-runtime
    resources:
      limits:
        cpu: 2
        memory: 8Gi
      requests:
        cpu: 1
        memory: 8Gi
  grpcDataEndpoint: port:8085
  grpcEndpoint: port:8085
  multiModel: true
  storageHelper:
    disabled: false
  supportedModelFormats:
    - autoSelect: true
      name: watson-nlp
```

Ensure you have a [trial key](https://www.ibm.com/account/reg/uk-en/signup?formid=urx-51726), then create the ServingRuntime resource:

```sh
IBM_ENTITLEMENT_KEY=<your trial key>
oc create secret docker-registry watson-nlp --docker-server=cp.icr.io/cp --docker-username=cp --docker-password=$IBM_ENTITLEMENT_KEY
git clone https://github.com/deleeuwblue/watson-embed-demos.git
oc apply -f watson-embed-demos/nlp/modelmesh-serving/servingruntime.yaml
```

Now we can see the new watson NLP serving runtime, in addition to those provided by default:

```sh
oc get servingruntimes

NAME                 DISABLED   MODELTYPE     CONTAINERS           AGE
mlserver-0.x                    sklearn       mlserver             7m6s
ovms-1.x                        openvino_ir   ovms                 7m6s
triton-2.x                      keras         triton               7m6s
watson-nlp-runtime              watson-nlp    watson-nlp-runtime   7s
```

## Deploying a pretrained Watson NLP model to Kserve ModelMesh

The next step is to upload a model to object storage.  Watson NLP for embed provides pre-trained models as containers, which are usually run as init containers to copy their data to a volume shared with the watson-nlp-runtime, see [Deployments to Kubernetes using yaml files or helm charts]({% post_url 2023-1-5-Deploying-IBM-Watson-NLP-to-Kubernetes %}).  In this case, we can run the model container as a k8s Job and configure the container to write to COS instead of a local volume mount.

An example [Job](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/job.yaml) is provided which launches the model container for the [Syntax model](https://www.ibm.com/docs/en/watson-libraries?topic=catalog-syntax).

Note the storage-config secret mentioned previously is mounted as a volume in the container.  The `env` variables configure the model container to copy its model data to the COS, referencing the credentials from the `localMinIO` section of the `storage-config`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-upload
  namespace: modelmesh-serving
spec:
  template:
    spec:
      containers:
        - name: syntax-izumo-en-stock
          image: cp.icr.io/cp/ai/watson-nlp_syntax_izumo_lang_en_stock:1.0.7
          env:
            - name: UPLOAD
              value: "true"
            - name: ACCEPT_LICENSE
              value: "true"
            - name: S3_CONFIG_FILE
              value: /storage-config/localMinIO
            - name: UPLOAD_PATH
              value: models
          volumeMounts:
            - mountPath: /storage-config
              name: storage-config
              readOnly: true
      volumes:
        - name: storage-config
          secret:
            defaultMode: 420
            secretName: storage-config
      restartPolicy: Never
      imagePullSecrets:
        - name: watson-nlp
  backoffLimit: 2
```

Create the Job:
```sh
oc apply -f watson-embed-demos/nlp/modelmesh-serving/job.yaml
```

Finally, an InferenceService CR needs to be created to make the model available via the watson-nlp Serving Runtime that we already created.  This resource defines the location for model `syntax-izumo-en` in COS.  It also specifies a modelFormat of `watson-nlp` which will assosiate the model with the `watson-nlp-runtime` serving runtime created previously.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: syntax-izumo-en
  namespace: modelmesh-serving
  annotations:
    serving.kserve.io/deploymentMode: ModelMesh
spec:
  predictor:
    model:
      modelFormat:
        name: watson-nlp
      storage:
        path: models/syntax_izumo_lang_en_stock
        key: localMinIO
```
Create the InferenceService:

```sh
oc apply -f watson-embed-demos/nlp/modelmesh-serving/inferenceservice.yaml
```

The status of the InferenceService can be verified:

```sh
oc get InferenceService

NAME              URL                                               READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
syntax-izumo-en   grpc://modelmesh-serving.modelmesh-serving:8033   True   
```

## Test the model 

```sh
oc port-forward service/modelmesh-serving 8033:8033
```

```sh
grpcurl -plaintext -proto ./common-service.proto \
-H 'mm-vmodel-id: syntax-izumo-en' \
-d '
{
  "parsers": [
    "TOKEN"
  ],
  "rawDocument": {
    "text": "This is a test."
  }
}
' \
127.0.0.1:8033 watson.runtime.nlp.v1.NlpService.SyntaxPredict
```


