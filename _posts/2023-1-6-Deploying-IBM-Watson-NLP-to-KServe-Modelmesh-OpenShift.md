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
