---
title: Deployments to Kubernetes using KServe Modelmesh
date: 2023-01-06 09:00:00 +/-0000
categories: [IBM Watson for Embed, NLP]
tags: [ai, nlp, kubernetes, kserve]     # TAG names should always be lowercase
image: https://raw.githubusercontent.com/deleeuwblue/deleeuwblog/main/assets/img/2023-1-6-Deploying-IBM-Watson-NLP-to-KServe-Modelmesh-OpenShift/mesh.png
---

In this blog, I will demonstrate how to deploy the Watson for NLP Library to OpenShift using KServe Modelmesh.

For initial context, read my blog [introducing IBM Watson for Embed]({{ site.baseurl }}/Introducing-IBM-Watson-for-Embed)

## Introducing KServe

[KServe](https://github.com/kserve/kserve) is a standard model inference platform on k8s.  It is built for highly scalable use cases and supports existing thrid party model servers and standard ML/DL model formats, or it can be extended to support additional runtimes like the Watson NLP runtime.

[Modelmesh Serving](https://github.com/kserve/modelmesh-serving) is intended to further increase KServe's scalability, especially when there are a large number of models which change frequently.  It intelligently loads and unloads models into memory from from cloud object storage (COS), to strike a trade-off between responsiveness to users and computational footprint.

## Install Kserve Modelmesh on a Kubernetes or OpenShift cluster

KServe Modelmesh requires etcd, S3 storage and optionally Knative and Istio.

Two approaches are available for installation:

* A [quick start approach](https://github.com/kserve/modelmesh-serving/blob/release-0.9/docs/quickstart.md) which includes all the pre-reqs, i.e. etcd and even local cloud object storage (COS) with minIO.
* A [customizable approach](https://github.com/kserve/modelmesh-serving/blob/release-0.9/docs/install/install-script.md) which requires Etcd to be already installed.

I took the quick start approach and installed to an OpenShift cluster with the following commands:

```sh
RELEASE=release-0.9
git clone -b $RELEASE --depth 1 --single-branch https://github.com/kserve/modelmesh-serving.git
cd modelmesh-serving
oc new-project modelmesh-serving
```

The quickstart at `release-0.9` currently has limitations meaning the etcd and Minio pods will crash at startup on OpenShift (although it works fine on Kubernetes).  This can be resolved by editing the `etcd` and `minio` Deployments in `/config/dependencies/quickstart.yaml`

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

For `minio`, change the `-data1` to `-/tmp/data1` as shown in the last line below:

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

Now run the quickstart script:

```
./scripts/install.sh --namespace modelmesh-serving --quickstart
```

After the script completes, you will find these pods running:

```sh
oc get pods

NAME                                    READY   STATUS    RESTARTS   AGE
etcd                                    1/1     Running   0          76m
minio                                   1/1     Running   0          76m
modelmesh-controller-77b8bf999c-2knhf   1/1     Running   0          75m
```

By default, there are some default serving runtimes defined:

```sh
oc get servingruntimes

NAME           DISABLED   MODELTYPE     CONTAINERS   AGE
mlserver-0.x              sklearn       mlserver     4m11s
ovms-1.x                  openvino_ir   ovms         4m11s
triton-2.x                keras         triton       4m11s
```

## Create Bucket

The installation created a secret with credentials for the local minIO object storage.

```sh
oc get secret/storage-config

NAME             TYPE     DATA   AGE
storage-config   Opaque   1      117m
```

The secret contains connection details for the "localMinIO" COS endpoint.  This secret becomes important later when uploading the models to be served.  The secret also defines the default bucket of `modelmesh-example-models`, which needs to be created on mino.  This can either be achieved using the [mc](https://min.io/download#/linux), or you can access the minio GUI for this simple task:

```sh
oc port-forward service/minio 9000:9000
```

Open `localhost:9000` in a browser.  Login using the credentials in the secret which you can view either via the OpenShift console or cli:

```sh
oc get secret/storage-config -oyaml
```

For example:

```yaml
{
  "type": "s3",
  "access_key_id": "XXXXX",
  "secret_access_key": "XXXXX",
  "endpoint_url": "http://minio:9000",
  "default_bucket": "modelmesh-example-models",
  "region": "us-south"
}
```

In the minio GUI, click the '+' button to add a bucket named `modelmesh-example-models`

![minIOGUI](https://raw.githubusercontent.com/deleeuwblue/deleeuwblog/main/assets/img/2023-1-6-Deploying-IBM-Watson-NLP-to-KServe-Modelmesh-OpenShift/minioGUI.png)

## Create a Pull Secret and ServiceAccount

Ensure you have a [trial key](https://www.ibm.com/account/reg/uk-en/signup?formid=urx-51726).

```sh
IBM_ENTITLEMENT_KEY=<your trial key>
oc create secret docker-registry ibm-entitlement-key --docker-server=cp.icr.io/cp --docker-username=cp --docker-password=$IBM_ENTITLEMENT_KEY
```

An example [ServiceAccount](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/serviceaccount.yaml) is provided.  Create a ServiceAccount that references the pull secret.

```sh
git clone https://github.com/deleeuwblue/watson-embed-demos.git
oc apply -f watson-embed-demos/nlp/modelmesh-serving/serviceaccount.yaml
```

Configure Modelmesh Serving to use this ServiceAccount, giving the controller access to the IBM entitled registry.  Use the OpenShift console to edit the Workloads->ConfigMap `model-serving-config-defaults` in the `modelmesh-serving` namespace.  

Set `serviceAccountName` to `pull-secret-sa`.  Also disable `restProxy` as this is not supported by Watson NLP:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-serving-config
data:
  config.yaml: |
    #Sample config overrides
    serviceAccountName: pull-secret-sa
    restProxy:
      enabled: false
```

Restart the `modelmesh-controller` pod:

```sh
oc scale deployment/modelmesh-controller --replicas=0 --all
oc scale deployment/modelmesh-controller --replicas=1 --all
```

## Configure a ServingRuntime for Watson NLP

An example [ServingRuntime resource](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/servingruntime.yaml) is provided.  The serving runtime defines the `cp.icr.io/cp/ai/watson-nlp-runtime` container image should be used to serve models that specify `watson-nlp` as their model format.  Note that `ServingRuntime` recommended by the [official documentation](https://www.ibm.com/docs/en/watson-libraries?topic=containers-run-kubernetes-kserve-modelmesh-serving) includes resource limits.  Becasause I was testing with a small OpenShift cluster, I needed to comment these out.

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
 #   resources:
 #     limits:
 #       cpu: 2
 #       memory: 8Gi
 #     requests:
 #       cpu: 1
 #       memory: 8Gi
  grpcDataEndpoint: port:8085
  grpcEndpoint: port:8085
  multiModel: true
  storageHelper:
    disabled: false
  supportedModelFormats:
    - autoSelect: true
      name: watson-nlp
```

Create the ServingRuntime resource:

```sh
oc apply -f watson-embed-demos/nlp/modelmesh-serving/servingruntime.yaml
```

Now you see the new watson NLP serving runtime, in addition to those provided by default:

```sh
oc get servingruntimes

NAME                 DISABLED   MODELTYPE     CONTAINERS           AGE
mlserver-0.x                    sklearn       mlserver             7m6s
ovms-1.x                        openvino_ir   ovms                 7m6s
triton-2.x                      keras         triton               7m6s
watson-nlp-runtime              watson-nlp    watson-nlp-runtime   7s
```