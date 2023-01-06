---
title: Deploying IBM Watson NLP to Kubernetes with OpenShift
date: 2023-01-01 09:00:00 +/-0000
categories: [IBM Watson for Embed, NLP]
tags: [ai, nlp, kubernetes, openshift]     # TAG names should always be lowercase
---

In this blog, I will demonstrate how to deploy the Watson for NLP Library to OpenShift using either [Kubernetes resources in yaml files](#deployments-using-kubernetes-resources), or using a [helm chart](#deployments-using-a-helm-chart).

For initial context, read my blog [introducing IBM Watson for Embed]({{ site.baseurl }}/Introducing-IBM-Watson-for-Embed)

The IBM Watson for NLP Library comprises two components:

* An [NLP runtime framework](https://www.ibm.com/docs/en/watson-libraries?topic=watson-natural-language-processing-library-embed-home)
* [One or more AI models](https://www.ibm.com/docs/en/watson-libraries?topic=models-catalog) which are loaded by the runtime

The pre-trained models provided by IBM are delivered as containers which are usually run as init containers when deploying to Kubernetes.

## Deployments using Kubernetes resources

You'll need the following resources:

* A Deployment

The Deployment deploys the watson-nlp-runtime image and one or more model containers which are run as init containers.  The init containers can be either pre-defined models from IBM, or custom models you have created yourself.

An example [Deployment](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/k8s/deployment.yaml) is provided.

```yaml
    spec:
      initContainers:
      - name: ensemble-workflow-lang-en-tone-stock
        image: cp.icr.io/cp/ai/watson-nlp_classification_ensemble-workflow_lang_en_tone-stock:1.0.6
        volumeMounts:
        - name: model-directory
          mountPath: "/app/models"
...
      containers:
      - name: watson-nlp-runtime
        image: cp.icr.io/cp/ai/watson-nlp-runtime:1.0.18
        ports:
        - containerPort: 8080
        - containerPort: 8085
        volumeMounts:
        - name: model-directory
          mountPath: "/app/models"
...
      volumes:
      - name: model-directory
        emptyDir: {}
```

Note that the both the initContainer(s) and watson-nlp-runtime containers use a VolumeMount at /app/models, which is backed by storage type emptyDir.  This is local storage in the Pod, rather than a persistent volume and is shared by both the runtime and model containers.  Kubernetes launches the init containers first, which copy their model data to the shared volume, then terminate.  The watson-nlp-runtime loads the model data from this shared volume.

* A Service

An example [Service](https://github.com/deleeuwblue/watson-embed-demos/blob/main/nlp/k8s/service.yaml) is provided.

The defines the networking to access the watson-nlp-runtime container on ports 8080 (REST) and 8085 (GRPC).

* A pull secret

IBM deliver these container images via a registry (cp.icr.io) which requires authentication.  See my blog about [running Watson NLP locally]({{ site.baseurl }}/Running-IBM-Watson-NLP-locally-in-Containers) for details of how to register for a trial key. 

Create the Secret:

```sh
IBM_ENTITLEMENT_KEY=<your trial key>
oc new-project watson-nlp
oc create secret docker-registry watson-nlp --docker-server=cp.icr.io/cp --docker-username=cp --docker-password=$IBM_ENTITLEMENT_KEY
```

### Deploying the yaml resources

Run the following oc commands to deploy the resources:

```sh
git clone https://github.com/deleeuwblue/watson-embed-demos.git
oc apply -f watson-embed-demos/nlp/k8s/
oc get pods --watch
```

Wait for the pod to become ready.  This can up to 10 minutes due to the size of the container image.

Now proceed with [testing the deployment](#testing-the-deployment)

## Deployments using a Helm chart

If you prefer to use a Helm chart, you can find one [here](https://github.com/cloud-native-toolkit/terraform-gitops-watson-nlp/tree/main/chart/watson-nlp).  This repo also contains other resources related to automated deployments, which you can ignore for now.

Run the following commands:

```sh
oc new-project watson-nlp
git clone https://github.com/cloud-native-toolkit/terraform-gitops-watson-nlp
cd terraform-gitops-watson-nlp/chart/watson-nlp
```

Populate the `values.yaml` as follows:

```yaml
componentName: watson-nlp
acceptLicense: true
serviceType: ClusterIP
imagePullSecrets:
  - watson-nlp
registries:
  - name: watson
    url: cp.icr.io/cp/ai
runtime:
  registry: watson
  image: watson-nlp-runtime:1.0.18
models:
  - registry: watson
    image: watson-nlp_classification_ensemble-workflow_lang_en_tone-stock:1.0.6
```

Explanation of the values.yaml:

componentName
: The Deployment and Services will be named using a combination of the Helm release, and this property.

serviceType
: The type of Kubernetes Service used to expose the watson-runtime container. Valid values are according to the Kuberenetes specification.

registries
: A list of all registries assosiated with the Deployment. At a minimum, there will be a registry from which to pull the watson-runtime container and IBM provided pretrained models. Additionally, there could be a separate registry containing custom models.

imagePullSecrets
: A list of pull secret names that the Deployment will reference. At a minimum, the pull secret for the IBM entitled registry/Artifactory should be provided. Additional pull secrets can be specified if there is a separate registry for custom models.

runtime
: Specifies which of the defined registries should be used to pull the watson-runtime container, and its image name/tag.

models
: A list of models to include in the Deployment, each specifies which of the defined registries should be used and the image names/tags.

Run the following commands to deploy the helm chart:

```sh
IBM_ENTITLEMENT_KEY=<your trial key>
oc new-project watson-nlp
oc create secret docker-registry watson-nlp --docker-server=cp.icr.io/cp --docker-username=cp --docker-password=$IBM_ENTITLEMENT_KEY
helm install -f values.yaml watson-embedded .
oc get pods --watch
```

Now proceed with [testing the deployment](#testing-the-deployment)

## Testing the Deployment

Run the following commands:

```sh
oc get svc
oc port-forward svc/<service name> 8080
```

In a second terminal, test the [Classification model](https://www.ibm.com/docs/en/watson-libraries?topic=catalog-classification) that was loaded by the init container, using cURL:

```sh
curl -s \
  "http://localhost:8080/v1/watson.runtime.nlp.v1/NlpService/ClassificationPredict" \
  -H "accept: application/json" \
  -H "content-type: application/json" \
  -H "Grpc-Metadata-mm-model-id: classification_ensemble-workflow_lang_en_tone-stock" \
  -d '{ "raw_document": { "text": "I hate school. School is bad." } }'
```

The response is as follows:

```sh
{
   "classes":[
      {
         "className":"frustrated",
         "confidence":0.74309075
      },
      {
         "className":"sad",
         "confidence":0.20021307
      },
      {
         "className":"impolite",
         "confidence":0.0734328
      },
      {
         "className":"excited",
         "confidence":0.029446114
      },
      {
         "className":"sympathetic",
         "confidence":0.02796789
      },
      {
         "className":"polite",
         "confidence":0.016257433
      },
      {
         "className":"satisfied",
         "confidence":0.0113145085
      }
   ],
   "producerId":{
      "name":"Voting based Ensemble",
      "version":"0.0.1"
   }
}
```

> Note, if the cURL does not return a response, be aware the model can take a 1-2 minutes to load, so just try again.
{: .prompt-tip }