---
title: Using PrimeQA For NLP Question Answering
date: 2023-03-13 09:00:00 +/-0000
categories: [IBM Watson for Embed, General Information]
tags: [ai, nlp]     # TAG names should always be lowercase
image: https://raw.githubusercontent.com/deleeuwblue/deleeuwblog/main/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/qamainimage.png
---
Question Answering models are one of the mainstays of NLP. In this blog I demonstrate how to use the open-source project [PrimeQA](https://github.com/primeqa) to install a framework which indexes data, and provides a Q&A capability which can easy be incorporated into an application.

## What is PrimeQA?

PrimeQA is a public open source repository that enables you to train state-of-the-art models for question answering (QA). PrimeQA can perform 
document retrieval and reading comprehension based on neural networks.  It provides a front-end UI search engine, in addition to APIs.

## Requirements for Installing PrimeQA

- Ubuntu Linux 20.04.4 LTS VM - Although primeQA recommend a high specification, I used a more modest 4vCPU, 16GB RAM without GPU
- Docker
- Git 
- Python & pip
- Java 11 JDK

## Instal Pre-requites

[Install Docker](https://docs.docker.com/engine/install/ubuntu/)

Install Docker Compose:
```sh
apt  install docker-compose
```

Install git:
```sh
apt install git
```

Install Python & pip:
```sh
apt install python3-pip
apt-get install python3-venv
```

Install Java 11 JDK:
```sh
apt-get install openjdk-11-jdk
```

## Installing PrimeQA on Ubuntu

Clone the [create-primea-app](https://github.com/primeqa/create-primeqa-app) repo:
```sh
mkdir /root/gitRepos
cd /root/gitRepos
git clone https://github.com/primeqa/create-primeqa-app
```

Set an environment variable to your public IP. This can be localhost if you plan to use a browser locally on Ubuntu via VNC:
```sh
export PUBLIC_IP=<your IP or localhost>
```

Launch primeQA by launching three container images. Use the `-m gpu` only if your Ubuntu host includes a GPU:
```sh
cd create-primeqa-app
./launch.sh [-m gpu]
```

Validate the three containers started:
```sh
docker ps
```

The containers run the following components:

* primeqa/services: a gRPC/REST service layer on top of PrimeQA
* primeqa/orchestrator: an integration service for PrimeQA capabilities and external services
* primeqa/ui: ReactJS webapp providing a example UI

Ensure that ports 50051, 50059 and 82 are accessible. If you plan to access the web Ui via a public IP and you're using a cloud virtual server, you may need to edit the network security (e.g. Security Group on IBM Cloud).

Settings are in the `orchestrator-store/primeqa.json`. Edit this file as follows:

```sh
{
    "application_id": "primeqa",
    "name": "PrimeQA",
    "description": "PrimeQA is a public open source repository that enables researchers and developers to train state-of-the-art models for question answering (QA).",
    "settings": {
        "retrievers": {
	  "PrimeQA": {
            "service_endpoint": "primeqa:50051"
          }
        },
        "readers": {
	  "PrimeQA": {
            "service_endpoint": "primeqa:50051",	
            "beta": 0.7
          }
        }
    }
}
```

This sets the Reader and Retriever processes to be provided locally by primeQA. It is possible to also add a configuration for Watson Discovery for the Reader. I'll explain later what the Reader and Retriever components do.

## Preparing the Corpus 

PrimeQA supports different models for information retrieval (provided by the Reader component of primeQA):

* There are two Dense IR engines supported: ColBERT and Direct Passage Retrieval (DPR).
* Sparse IR is a Pyserini-based IR Engine enabling BM25 ranking using bag of words representation.

In this tutorial, we will use Dr.Decr, a ColBERT based fine-tuned language model to create an index. It expects the corpus to be a collection of documents in a tsv file, in a tabular `id text title` arrangement, ideally in the 1-180 word range. The primeqa repo provides some [sample scripts](https://github.com/primeqa/create-primeqa-app/tree/main/examples/harry-potter-corpus) to help prepare the data into this format. The scripts are designed for another book which is no longer available in the public domain, but this is a good starting point.

For this blog, I used the following [dataset from Kaggle](https://www.kaggle.com/datasets/edenbd/children-stories-text-corpus). It contains multiple stories, but I copied lines 67331-67543 to a new file `Book1.txt`, giving me just one story, "MY FATHER'S DRAGON" which is public domain. 

```sh
cd /root/gitRepos/create-primeqa-app/examples
cp -R harry-potter-corpus my_fathers_dragon
cd my_fathers_dragon
```

Copy `Book1.txt` from your workstation to `/examples/my_fathers_dragon`.

Install the pre-requisites for processing the corpus:
```sh
alias pip=pip3
alias python=python3
python -m venv .env           # will create directory .env
source .env/bin/activate      # activate virtualenv
pip install primeqa
pip install --upgrade pip
pip install spacy             
python -m spacy download en_core_web_sm
```

Process the corpus:
```sh
./process.sh
```

The resulting corpus.tsv looks like this:
```sh
id	text	title
1	"MY FATHER'S DRAGON MY FATHER MEETS THE CAT One cold rainy day when my father was a little boy, he met an old alley cat on his street. The cat was very drippy and uncomfortable so my father said, ""Wouldn't you like to come home with me?""This surprised the cat she had never before met anyone who cared about old alley cats but she said, ""I'd be very much obliged if I could sit by a warm furnace, and perhaps have a saucer of milk.""""We have a very nice furnace to sit by,"" said my father, ""and I'm sure my mother has an extra saucer of milk."""	Book1 Paragraph 1
2	"My father and the cat became good friends but my father's mother was very upset about the cat. She hated cats, particularly ugly old alley cats. ""Elmer Elevator,"" she said to my father, ""if you think I'm going to give that cat a saucer of milk, you're very wrong. Once you start feeding stray alley cats you might as well expect to feed every stray in town, and I am not going to do it!""This made my father very sad, and he apologized to the cat because his mother had been so rude. He told the cat to stay anyway, and that somehow he would bring her a saucer of milk each day. My father fed the cat for three weeks, but one day his mother found the cat's saucer in the cellar and she was extremely angry. She whipped my father and threw the cat out the door, but later on my father sneaked out and found the cat."	Book1 Paragraph 2
```

## Package the corpus, index and models

The corpus, index and model need to be installed into the 'PrimeQA store' which is located in directory `primeqa-store`.

The model used as the Reader is not installed by default. Run the following script to download it to `/examples/my_fathers_dragon`:

```sh
./download-model.sh
```

Run the following script to copy the model to a new location in `primeqa-store`.

```sh
CHECKPOINT=$(./setup-checkpoint.sh DrDecr.dnn ../../primeqa-store)
```

The script returns a path which is stored in the CHECKPOINT variable for later use. Display the value with `echo $CHECKPOINT` and make a note of it.

Next, invoke a script which uses the DrDecr model to create an index, and copy the files to the `primeqa-store` directory. In addition, an SQLite database is created.

```sh
./setup-index.sh "${CHECKPOINT}" corpus.tsv ../../primeqa-store/
```

The `setup-index.sh` script created one or more <indexname> directories at `primeqa-store/indexes/`. 

> If you find there are multiple directories with names such as `9329c257-a514-48a7-ac03-c53a05e3ec5f`, inspect each one in turn to determine if there are files in the subdirectory `9329c257-a514-48a7-ac03-c53a05e3ec5f/index`.  If this is empty, the <indexname> directory is not required and can be deleted. Repeat this process until only one <indexname> directory remains, which should have a populated `index` subdirectory. 
{: .prompt-tip }

Rename the <indexname> directory:

```sh
cd /root/gitRepos/create-primeqa-app/primeqa-store/indexes
mv <indexname> my_fathers_dragon
```

The `setup-index.sh` script also created file `primeqa-store/indexes/<indexname>/information.json`. At the time of writing, this file has been created incorrectly. Replace the contents so it looks like this:

```sh
{
  "index_id": "my_fathers_dragon",
  "status": "READY",
  "configuration": {
    "engine_type": "ColBERT",
    "checkpoint": "<penultimate directory of your $CHECKPOINT path, e.g. ColBERTIndexer_20230308101308>"  
  }
}
```

## Restart primeQA

It may not be strictly necessary, but after making the configuration I restarted the containers:

```
cd /root/gitRepos/create-primeqa-app
./terminate.sh
./launch.sh
```

## Testing with PrimeQA UI

Open a browser to `PUBLIC_IP:82/qa`.

To help with composing some test searching, you can read a [brief summary of My Father Met a Dragon](https://www.supersummary.com/my-fathers-dragon/summary/).

As you test the three capabilities below (Retrieval, Reading, Question Answering), you will notice the first query takes a long time. This is because primeQA is downloading models in the background. These are the transformer (large language) models used for Reading and Question Answering. PrimeQA allows these models can be replaced according to the specific requirements.

You will see the model appear here `create-primeqa-app/cache/huggingface/hub`:

```sh
-rwxrwxrwx 1 2000 2000    1 Mar 14 09:21 version.txt
drwxr-xr-x 6 2000 2000 4096 Mar 14 13:39 models--xlm-roberta-base
drwxr-xr-x 6 2000 2000 4096 Mar 14 13:41 models--PrimeQA--nq_tydi_sq1-reader-xlmr_large-20221110
```

### Test Retrieval

Retrieval searches the document index. Select the Retrieval icon on the left and query 'What animals did Father meet?'.

You will see documents from the index which mention the animals such as cat, bear, rhinoceras, lion, monkey etc. However you do not get one word answers.

This uses the Retriever model (Dr.Decr) to find documents in the index.

![testRetrieval](/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/retrieval.png)

### Test Reading

Reading finds an answer only from the provided context information. Select the Reading icon on the left and query 'Did Mother link cats?', providing the context:

```
"My father and the cat became good friends but my father's mother was very upset about the cat. She hated cats, particularly ugly old alley cats. ""Elmer Elevator,"" she said to my father, ""if you think I'm going to give that cat a saucer of milk, you're very wrong. Once you start feeding stray alley cats you might as well expect to feed every stray in town, and I am not going to do it!""This made my father very sad, and he apologized to the cat because his mother had been so rude. He told the cat to stay anyway, and that somehow he would bring her a saucer of milk each day. My father fed the cat for three weeks, but one day his mother found the cat's saucer in the cellar and she was extremely angry. She whipped my father and threw the cat out the door, but later on my father sneaked out and found the cat."
```

The answer 'She hated cats' will be generated only from the context provided (in this case a document from the index).

This uses the Reader model (nq_tydi_sq1-reader-xlmr_large-20221110).

![testReading](/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/reading.png)

### Test Question Answering

Question Answering proposes a short answer and presents the document from the index which contained the information. There are various settings which can influence the results, for example the minimum/maximum number of tokens for the answer. In this example, I reduced the number of answer tokens as I wanted to encourage short answers listing the animals which the father met.

Select the QA icon on the left and query 'What animals did Father meet?'.

This uses the Retriever model (Dr.Decr) to find documents in the index, and the Reader model (nq_tydi_sq1-reader-xlmr_large-20221110) to propose a specific answer to the question.

![testQA](/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/qa.png)

