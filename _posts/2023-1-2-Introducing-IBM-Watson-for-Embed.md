---
title: Introducing IBM Watson for Embed
date: 2023-01-01 09:00:00 +/-0000
categories: [IBM Watson for Embed, NLP]
tags: [ai, nlp]     # TAG names should always be lowercase
---

IBM has recently released a framework that is specifically designed for developers to embed AI into their solutions.  IBM Watson for Embed lowers the barrier for AI adoption by helping address the skills shortage and development costs that are required to build AI models from scratch.  A common framework is used to run AI libraries including natural language processing (NLP) and Speech.  More AI libraries are coming soon.  

The AI libraries run as containers and provide REST and gRPC interfaces, making them easily embeddable into solutions.  Although this embeddable approach is new, the underlying functionality is already optimised and used in existing IBM cloud services like Watson Assistant, NLU (Natural Language Understanding), Speech to Text and Text to Speech.

The [IBM Watson NLP Library](https://www.ibm.com/products/ibm-watson-natural-language-processing) provides pre-trained models for NLP tasks including sentiment analysis, phrase extraction and text classification.  It is built on leading open source software, with IBM providing the benefit of stable and supported interfaces, a wide choice of languages and enterprise support.  Optionally, custom models can be created and served using the same framework as the pre-trained models.

The [IBM Watson Speech Library](https://www.ibm.com/products/watson-speech-embed-libraries) provides customisable speech-to-text, and text-to-speech using a selection of male and female voices.

To try IBM Watson for Embed, a [trial](https://www.ibm.com/account/reg/uk-en/signup?formid=urx-51726) is available. The container images are stored in an IBM container registry that is accessed via an IBM Entitlement Key.

The following options are available to deploy Watson NLP:

* [Locally using container engines like Docker or Podman]({{ site.baseurl }}/Running-IBM-Watson-NLP-locally-in-Containers)
* Deployments to Kubernetes using Minikube
* [Deployments to Kubernetes using yaml files or helm charts]({{ site.baseurl }}/Deploying-IBM-Watson-NLP-to-Kubernetes)
* Deployments to Kubernetes using KServe ModelMesh Serving
* Deployments to OpenShift via TechZone Deployer using Terraform and ArgoCD

