---
title: Running IBM Watson NLP locally in Containers
date: 2023-01-01 09:00:00 +/-0000
categories: [IBM Watson for Embed, NLP]
tags: [ai, nlp, containers]     # TAG names should always be lowercase
---

In this blog, I will demonstrate how to run the Watson for NLP Library locally using containers with Docker.

For initial context, read my blog [introducing IBM Watson for Embed]({{ site.baseurl }}/Introducing-IBM-Watson-for-Embed)

## Select Models

The IBM Watson for NLP Library comprises two components:

* An [NLP runtime framework](https://www.ibm.com/docs/en/watson-libraries?topic=watson-natural-language-processing-library-embed-home)
* [One or more AI models](https://www.ibm.com/docs/en/watson-libraries?topic=models-catalog) which are loaded by the runtime

You first need to decide which models you want to use, according to the use cases.  This helps minimise the size of the resulting container.  You can also train your own models which I will blog about separately.

The pre-trained models provided by IBM are delivered as containers which are usually run as init containers when deploying to Kubernetes [see other deployment options](#further-options-are-available-to-deploy-watson-nlp).  To use the pre-trained models with a standalone container (e.g. locally using Docker), you need to extract the model from the model container and combine it with the runtime to create a custom image, using a Dockerfile.

There are two approaches to extract the model from the provided containers:

* Launch the model containers with a local volume mount

The containers copy the model files to a specifed path, then exit.  If that path is a local volume mount, you then have the model data locally.  You can then use a Dockerfile to copy the files into the custom image.

* Use a multi-stage Dockerfile

The Dockerfile runs the 'copy script' in each model container, which copies the files from the model container directly to the new custom image.

The resulting custom image can be run locally.  If you were to look in /app/data, you'd see the models you added.

I'll demonstrate both approaches.  Note that the Watson NLP Library will not run with ARM architecture (e.g. Apple M1), so you'll need to test these steps using x86.  I used an Ubuntu virtual server, onto which I [installed Docker engine](https://docs.docker.com/engine/install/ubuntu/).

## Option 1: local volume mount

Some environment variables should be set to define the models to download, and login to the IBM entitled registry.

```sh
IBM_ENTITLEMENT_KEY=<your trial key>
echo $IBM_ENTITLEMENT_KEY | docker login -u cp --password-stdin cp.icr.io
REGISTRY=cp.icr.io/cp/ai
MODELS="watson-nlp_syntax_izumo_lang_en_stock:1.0.7 watson-nlp_classification_ensemble-workflow_lang_en_tone-stock:1.0.7"
mkdir models
chmod 777 models  
```

The following script pulls each model image from the registry and runs it using a volume mount between the container and the local filesystem.  The containers copy their model files to /app/models, then terminate.  Therefore the models end up on the local filesystem at ./models.  

```sh
for i in $(echo "$MODELS")
do
  image=${REGISTRY}/$i
  docker run -it --rm -e ACCEPT_LICENSE=true -v `pwd`/models:/app/models $image
done
```

Create a Dockerfile with the following contents.  Note the contents of your local /models directory are copied into the new container image at /app/models:

```sh
ARG REGISTRY
ARG TAG=1
FROM ${REGISTRY}/watson-nlp-runtime:${TAG}
COPY models /app/models
```

Create the image:

```sh
docker build . -t my-watson-nlp-runtime:latest --build-arg REGISTRY=${REGISTRY}
```

You can now proceed to [run the container](#running-the-nlp-container-locally-with-docker)

## Option 2: multi-stage Dockerfile

The following Dockerfile uses multiple model containers.  In the build process, the script `unpack_model.sh` is run for each model container, which unzips the model files to /app/models.  A new image is created from WATSON_RUNTIME_BASE, and the unzipped models from the previous stages are copied to the new image.  This approach does not require volume mounts to the local filesystem.


```sh
ARG WATSON_RUNTIME_BASE="cp.icr.io/cp/ai/watson-nlp-runtime:1"
ARG SYNTAX_MODEL="cp.icr.io/cp/ai/watson-nlp_syntax_izumo_lang_en_stock:1.0.7"
ARG CLASSIFICATION_MODEL="cp.icr.io/cp/ai/watson-nlp_classification_ensemble-workflow_lang_en_tone-stock:1.0.7"

FROM ${SYNTAX_MODEL} as model1
RUN ./unpack_model.sh

FROM ${CLASSIFICATION_MODEL} as model2
RUN ./unpack_model.sh

FROM ${WATSON_RUNTIME_BASE} as release

RUN true && \
    mkdir -p /app/models

ENV LOCAL_MODELS_DIR=/app/models
COPY --from=model1 app/models /app/models
COPY --from=model2 app/models /app/models
```

Create the image:

```sh
docker build . -t my-watson-nlp-runtime:latest
```

You can now proceed to [run the container](#running-the-nlp-container-locally-with-docker)

## Running the NLP container locally with Docker

Run the image:

```sh
docker run -d -e ACCEPT_LICENSE=true -p 8085:8085 -p 8080:8080 my-watson-nlp-runtime:latest
```

Open a second terminal to test the models you loaded into the NLP runtime.  The runtime has REST and gRPC interfaces.  You can see the endpoints by launching the Swagger user interface at http://localhost:8080/swagger.

## Test the Syntax Model

The [Syntax model](https://www.ibm.com/docs/en/watson-libraries?topic=catalog-syntax) provides fundamental NLP tasks on the input text, for example:

* Sentence detection
* Tokenization: can't -> ca + n't
* Part-of-Speech tagging: I thought -> I/PRON, thought/VERB
* Lemmatization: I thought -> I/I, thought/think
* Dependency parsing: I -> nsubj -> thought -> root

Let's invoke it with cURL:

```sh
curl -s \
  "http://localhost:8080/v1/watson.runtime.nlp.v1/NlpService/SyntaxPredict" \
  -H "accept: application/json" \
  -H "content-type: application/json" \
  -H "Grpc-Metadata-mm-model-id: syntax_izumo_lang_en_stock" \
  -d '{ "raw_document": { "text": "This is a test sentence" }, "parsers": ["token"] }'
```

The response is as follows:

```sh
{
   "text":"This is a test sentence",
   "producerId":{
      "name":"Izumo Text Processing",
      "version":"0.0.1"
   },
   "tokens":[
      {
         "span":{
            "begin":0,
            "end":4,
            "text":"This"
         },
         "lemma":"",
         "partOfSpeech":"POS_UNSET",
         "dependency":null,
         "features":[
            
         ]
      },
      {
         "span":{
            "begin":5,
            "end":7,
            "text":"is"
         },
         "lemma":"",
         "partOfSpeech":"POS_UNSET",
         "dependency":null,
         "features":[
            
         ]
      },
      {
         "span":{
            "begin":8,
            "end":9,
            "text":"a"
         },
         "lemma":"",
         "partOfSpeech":"POS_UNSET",
         "dependency":null,
         "features":[
            
         ]
      },
      {
         "span":{
            "begin":10,
            "end":14,
            "text":"test"
         },
         "lemma":"",
         "partOfSpeech":"POS_UNSET",
         "dependency":null,
         "features":[
            
         ]
      },
      {
         "span":{
            "begin":15,
            "end":23,
            "text":"sentence"
         },
         "lemma":"",
         "partOfSpeech":"POS_UNSET",
         "dependency":null,
         "features":[
            
         ]
      }
   ],
   "sentences":[
      {
         "span":{
            "begin":0,
            "end":23,
            "text":"This is a test sentence"
         }
      }
   ],
   "paragraphs":[
      {
         "span":{
            "begin":0,
            "end":23,
            "text":"This is a test sentence"
         }
      }
   ]
}
```

## Test the Classification Model

The [Classification model](https://www.ibm.com/docs/en/watson-libraries?topic=catalog-classification) classifies the input text into one or more of a pre-determined set of labels.

Let's invoke it with cURL:

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

Note, if the cURL does not return a response, be aware the model can take a 1-2 minutes to load, so just try again.

## Multiple options are available to deploy Watson NLP

* Locally using container engines like Docker or Podman (the focus of this blog post)
* Deployments to Kubernetes using Minikube
* [Deployments to Kubernetes using yaml files or helm charts]({{ site.baseurl }}/Deploying-IBM-Watson-NLP-Kubernetes)
* Deployments to Kubernetes using KServe ModelMesh Serving
* Deployments to OpenShift via TechZone Deployer using Terraform and ArgoCD

