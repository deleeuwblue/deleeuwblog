---
title: Deploying IBM Watson NLP to Kubernetes using KServe Modelmesh
date: 2023-01-06 09:00:00 +/-0000
categories: [IBM Watson for Embed, NLP]
tags: [ai, nlp, kubernetes, kserve]     # TAG names should always be lowercase
image: https://raw.githubusercontent.com/deleeuwblue/deleeuwblog/main/assets/img/2023-1-6-Deploying-IBM-Watson-NLP-to-KServe-Modelmesh-OpenShift/mesh.png
---

In this blog, I will demonstrate how to deploy the Watson for NLP Library to Kubernetes using KServe Modelmesh.

For initial context, read my blog [introducing IBM Watson for Embed]({% post_url 2023-1-2-Introducing-IBM-Watson-for-Embed %}).

For deployment to OpenShift, see this [blog]({% post_url 2023-1-6-Deploying-IBM-Watson-NLP-to-KServe-Modelmesh-OpenShift %}).

## Introducing KServe

[KServe](https://github.com/kserve/kserve) is a standard model inference platform on k8s.  It is built for highly scalable use cases and supports existing third party model servers and standard ML/DL model formats, or it can be extended to support additional runtimes like the Watson NLP runtime.

[Modelmesh Serving](https://github.com/kserve/modelmesh-serving) is intended to further increase KServe's scalability, especially when there are a large number of models which change frequently.  It intelligently loads and unloads models into memory from from cloud object storage (COS), to strike a trade-off between responsiveness to users and computational footprint.

## Install Kserve Modelmesh on Kubernetes

KServe Modelmesh requires etcd, S3 storage and optionally Knative and Istio.

Two approaches are available for installation:

* A [quick start approach](https://github.com/kserve/modelmesh-serving/blob/release-0.9/docs/quickstart.md) which includes all the pre-reqs, i.e. etcd and even local cloud object storage (COS) with minIO.
* A [customizable approach](https://github.com/kserve/modelmesh-serving/blob/release-0.9/docs/install/install-script.md) which requires Etcd to be already installed.

I took the quick start approach and installed to an Kubernetes cluster with the following commands:

```sh
RELEASE=release-0.9
git clone -b $RELEASE --depth 1 --single-branch https://github.com/kserve/modelmesh-serving.git
cd modelmesh-serving
kubectl create namespace modelmesh-serving
kubectl config set-context --current --namespace=modelmesh-serving
./scripts/install.sh --namespace modelmesh-serving --quickstart
```

After the script completes, you will find these pods running:

```sh
kubectl get pods

NAME                                    READY   STATUS    RESTARTS   AGE
etcd                                    1/1     Running   0          76m
minio                                   1/1     Running   0          76m
modelmesh-controller-77b8bf999c-2knhf   1/1     Running   0          75m
```

By default, there are some default serving runtimes defined:

```sh
kubectl get servingruntimes

NAME           DISABLED   MODELTYPE     CONTAINERS   AGE
mlserver-0.x              sklearn       mlserver     4m11s
ovms-1.x                  openvino_ir   ovms         4m11s
triton-2.x                keras         triton       4m11s
```

The quick start installation has also created a secret with credentials for the local minIO object storage.

```sh
kubectl get secret/storage-config -n modelmesh-serving

NAME             TYPE     DATA   AGE
storage-config   Opaque   1      117m
```

The secret contains connection details for the "localMinIO" COS endpoint.  This secret becomes important later when uploading the models to be served.

## Create a Pull Secret and ServiceAccount

Ensure you have a [trial key](https://www.ibm.com/account/reg/uk-en/signup?formid=urx-51726).

```sh
IBM_ENTITLEMENT_KEY=<your trial key>
kubectl create secret docker-registry ibm-entitlement-key --docker-server=cp.icr.io/cp --docker-username=cp --docker-password=$IBM_ENTITLEMENT_KEY
```

An example [ServiceAccount](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/serviceaccount.yaml) is provided.  Create a ServiceAccount that references the pull secret.

```sh
git clone https://github.com/deleeuwblue/watson-embed-demos.git
kubectl apply -f watson-embed-demos/nlp/modelmesh-serving/serviceaccount.yaml
```

Configure Modelmesh Serving to use this ServiceAccount, giving the controller access to the IBM entitled registry.  Use the Kubernetes console to edit the Config and Storage->ConfigMap `model-serving-config-defaults` in the `modelmesh-serving` namespace.  

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
kubectl scale deployment/modelmesh-controller --replicas=0 --all
kubectl scale deployment/modelmesh-controller --replicas=1 --all
```

Patch all service accounts:

```sh
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "ibm-entitlement-key"}]}'
kubectl patch serviceaccount modelmesh -p '{"imagePullSecrets": [{"name": "ibm-entitlement-key"}]}'
kubectl patch serviceaccount modelmesh-controller -p '{"imagePullSecrets": [{"name": "ibm-entitlement-key"}]}'
```

## Configure a ServingRuntime for Watson NLP

An example [ServingRuntime resource](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/servingruntime.yaml) is provided.  The serving runtime defines the `cp.icr.io/cp/ai/watson-nlp-runtime` container image should be used to serve models that specify `watson-nlp` as their model format.

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
kubectl apply -f watson-embed-demos/nlp/modelmesh-serving/servingruntime.yaml --namespace modelmesh-serving
```

Now you see the new watson NLP serving runtime, in addition to those provided by default:

```sh
kubectl get servingruntimes

NAME                 DISABLED   MODELTYPE     CONTAINERS           AGE
mlserver-0.x                    sklearn       mlserver             7m6s
ovms-1.x                        openvino_ir   ovms                 7m6s
triton-2.x                      keras         triton               7m6s
watson-nlp-runtime              watson-nlp    watson-nlp-runtime   7s
```

## Upload a pre-trained Watson NLP model to Cloud Object Storage

The next step is to upload a model to object storage.  Watson NLP provides pre-trained models as containers, which are usually run as init containers to copy their data to a volume shared with the watson-nlp-runtime, see [Deployments to Kubernetes using yaml files or helm charts]({% post_url 2023-1-5-Deploying-IBM-Watson-NLP-to-Kubernetes %}).  When using Modelmesh, the goal is to copy the model data to COS.  To achieve this, we can run the model container as a k8s Job, where the model container is configured to write to COS instead of a local volume mount.

An example [Job](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/modelmesh-serving/job.yaml) is provided which launches the model container for the [Syntax model](https://www.ibm.com/docs/en/watson-libraries?topic=catalog-syntax).  The `env` variables which configure the model container to copy its data to COS, referencing the credentials from the `localMinIO` section of the `storage-config` secret, which is mounted as a volume.

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
  backoffLimit: 2
```

Create the Job:
```sh
kubectl apply -f watson-embed-demos/nlp/modelmesh-serving/job.yaml --namespace modelmesh-serving
```

## Create a InferenceService for the Syntax model

Finally, an InferenceService CR needs to be created to make the model available via the watson-nlp Serving Runtime that we already created.  This resource defines the location for model `syntax-izumo-en` in COS.  It also specifies a `modelFormat` of `watson-nlp` which will associate the model with the `watson-nlp-runtime` serving runtime.

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
kubectl apply -f watson-embed-demos/nlp/modelmesh-serving/inferenceservice.yaml
```

The status of the InferenceService can be verified:

```sh
kubectl get InferenceService

NAME              URL                                               READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
syntax-izumo-en   grpc://modelmesh-serving.modelmesh-serving:8033   True   
```

> Note, the watson-nlp-runtime container can take 5-10 minutes to download.  Until this has completed, the InferenceService will show a status of false.
{: .prompt-tip }


## Test the model 

The `modelmesh-serving` Service does not expose a REST port, only GRPC.  Interacting with GRPC requires the proto files.  They are published [here](https://github.com/IBM/ibm-watson-embed-clients).  Enter the following commands to test the Syntax model using [grpcurl](https://formulae.brew.sh/formula/grpcurl):


```sh
kubectl port-forward service/modelmesh-serving 8033:8033
```

Open a second terminal and run commands:

```sh
git clone https://github.com/IBM/ibm-watson-embed-clients
cd ibm-watson-embed-clients/watson_nlp/protos
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

The GRPC call is routed by the `modelmesh-serving` Service to the appropriate serving runtime pod for the model requested.  Modelmesh ensures there are enough Serving Runtime pods to meet demand.  The response from the `watson-nlp-runtime` should look like this:

```sh
{
  "text": "This is a test.",
  "producerId": {
    "name": "Izumo Text Processing",
    "version": "0.0.1"
  },
  "tokens": [
    {
      "span": {
        "end": 4,
        "text": "This"
      }
    },
    {
      "span": {
        "begin": 5,
        "end": 7,
        "text": "is"
      }
    },
    {
      "span": {
        "begin": 8,
        "end": 9,
        "text": "a"
      }
    },
    {
      "span": {
        "begin": 10,
        "end": 14,
        "text": "test"
      }
    },
    {
      "span": {
        "begin": 14,
        "end": 15,
        "text": "."
      }
    }
  ],
  "sentences": [
    {
      "span": {
        "end": 15,
        "text": "This is a test."
      }
    }
  ],
  "paragraphs": [
    {
      "span": {
        "end": 15,
        "text": "This is a test."
      }
    }
  ]
}
```