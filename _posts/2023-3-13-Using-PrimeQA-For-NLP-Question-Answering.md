---
title: Using PrimeQA For NLP Question Answering
date: 2023-03-13 09:00:00 +/-0000
categories: [IBM Watson for Embed, General Information]
tags: [ai, nlp]     # TAG names should always be lowercase
image: https://raw.githubusercontent.com/deleeuwblue/deleeuwblog/main/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/qamainimage.png
---
Question Answering models are one of the mainstays of NLP. In this blog I demonstrate how to use the open-source project [PrimeQA](https://github.com/primeqa) to install a framework which indexes data, and provides a Q&A capability which can easy be incorporated into an application.

In this blog you'll learn how to install primeQA and index a children's book available as public domain.

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

Install git:
```
apt install git
```

Install Python & pip:
```
apt install python-pip
apt-get install python3-venv
```

Install Java 11 JDK:
```
apt-get install openjdk-11-jdk
```

## Installing PrimeQA on Ubuntu

Clone the [create-primea-app](https://github.com/primeqa/create-primeqa-app) repo:
```
mkdir /root/gitRepos
cd /root/gitRepos
git clone https://github.com/primeqa/create-primeqa-app
```

Set an environment variable to your public IP. This can be localhost if you plan to use a browser locally on Ubuntu via VNC:
```
export PUBLIC_IP=<your IP or localhost>
```

Launch primeQA by launching three container images. Use the `-m gpu` only if your Ubuntu host includes a GPU:
```
launch.sh [-m gpu]
```

Validate the three containers started:
```
docker ps
```

The containers run the following components:

* primeqa/services: a gRPC/REST service layer on top of PrimeQA
* primeqa/orchestrator: an integration service for PrimeQA capabilities and external services
* primeqa/ui: ReactJS webapp providing a example UI

Ensure that ports 50051, 50059 and 82 are accessible. If you plan to access the web Ui via a public IP and you're using a cloud virtual server, you may need to edit the network security (e.g. Security Group on IBM Cloud).

Note that settings are in the `orchestrator-store/primeqa.json`. By default, both the Reader and Retriever processes will be provided locally by primeQA. It is possible to also add a configuration for Watson Discovery for the Reader. I'll explain later what these components do.

## Preparing the Corpus 

Our corpus is available here. It is a public domain children's story in text format. PrimeQA supports different models for information retrieval (provided by the Reader component of primeQA):

* There are two Dense IR engines supported: ColBERT and Direct Passage Retrieval (DPR).
* Sparse IR is a Pyserini-based IR Engine enabling BM25 ranking using bag of words representation.

In this tutorial, we will use Dr.Decr, a ColBERT based fine-tuned language model to create an index. It expects the corpus to be a collection of documents in a tsv file, in a tabular `id text title` arrangement, ideally in the 1-180 word range. The primeqa repo provides some [sample scripts](https://github.com/primeqa/create-primeqa-app/tree/main/examples/harry-potter-corpus) to help prepare the data into this format. The scripts are designed for another book which is no longer available in the public domain, but this is a good starting point.

For this blog, I used the following [dataset from Kaggle](https://www.kaggle.com/datasets/edenbd/children-stories-text-corpus). It contains multiple stories, but I copied lines a 67331-67543 to a new file `Book1.txt`, giving me just one story, "MY FATHER'S DRAGON". Copy `Book1.txt` to /examples/harrypotter.

Install the pre-requisites for processing the corpus:
```
cd /root/gitRepos/create-primeqa-app/examples/harrypotter
python -m venv .env           # will create directory .env
source .env/bin/activate      # activate virtualenv
alias pip=pip3
alias python=python3
pip install primeqa
pip install --upgrade pip
pip install spacy             # install stuff
python -m spacy download en_core_web_sm
```

Process the corpus:
```
./process.sh
```

The resulting corpus.tsv looks like this:
```
id	text	title
1	"MY FATHER'S DRAGON MY FATHER MEETS THE CAT One cold rainy day when my father was a little boy, he met an old alley cat on his street. The cat was very drippy and uncomfortable so my father said, ""Wouldn't you like to come home with me?""This surprised the cat she had never before met anyone who cared about old alley cats but she said, ""I'd be very much obliged if I could sit by a warm furnace, and perhaps have a saucer of milk.""""We have a very nice furnace to sit by,"" said my father, ""and I'm sure my mother has an extra saucer of milk."""	Book1 Paragraph 1
2	"My father and the cat became good friends but my father's mother was very upset about the cat. She hated cats, particularly ugly old alley cats. ""Elmer Elevator,"" she said to my father, ""if you think I'm going to give that cat a saucer of milk, you're very wrong. Once you start feeding stray alley cats you might as well expect to feed every stray in town, and I am not going to do it!""This made my father very sad, and he apologized to the cat because his mother had been so rude. He told the cat to stay anyway, and that somehow he would bring her a saucer of milk each day. My father fed the cat for three weeks, but one day his mother found the cat's saucer in the cellar and she was extremely angry. She whipped my father and threw the cat out the door, but later on my father sneaked out and found the cat."	Book1 Paragraph 2
```

## Package the corpus, index and models

The corpus, index and model need to be installed into the 'PrimeQA store' which is located in directory `primeqa-store`.

The model used as the Reader is not installed by default. Run the following script to download it to /examples/harrypotter:

```
./download-model.sh
```

Run the following script to copy the model to a new location in `primeqa-store`.

```sh
CHECKPOINT=$(./setup-checkpoint.sh DrDecr.dnn ../../primeqa-store)
```

The script returns a path which is stored in the CHECKPOINT variable for later use. Display the value with `echo $CHECKPOINT` and make a note of it.

Next, invoke a script which uses the DrDecr model to create an index, and copy the files to the `primeqa-store` directory. In addition, an SQLite database is created.

```
./setup-index.sh "${CHECKPOINT}" corpus.tsv ../../primeqa-store/
```

The `setup-index.sh` script created one or more <indexname> directories at `primeqa-store/indexes/`. If you find there are multiple directories with names such as `9329c257-a514-48a7-ac03-c53a05e3ec5f`, inspect each one in turn to determine if there are files in the subdirectory `9329c257-a514-48a7-ac03-c53a05e3ec5f/index`.  If this is empty, the <indexname> directory is not required and can be deleted. Repeat this process until only one <indexname> directory remains, which should have a populated `index` subdirectory. Rename this <indexname>:

```
mv primeqa-store/indexes/<indexname> primeqa-store/indexes/my_father_met_a_dragon
```

The `setup-index.sh` script also created file `primeqa-store/indexes/<indexname>/information.json`. At the time of writing, this file has been created incorrectly. Edit the file so it looks like this:

```
{
  "index_id": "my_father_met_a_dragon",
  "status": "READY",
  "configuration": {
    "engine_type": "ColBERT",
    "checkpoint": "<penultimate directory of your $CHECKPOINT path>"  #E.g. ColBERTIndexer_20230308101308  
  }
}
```

During the processing of `setup-index.sh`, you may have noticed that several transformer models were downloaded to `create-primeqa-app/cache/huggingface/hub`:

```
drwxrwxrwx 6 2000 2000 4096 Mar  8 08:57 models--PrimeQA--nq_tydi_sq1-reader-xlmr_large-20221110
drwxrwxrwx 2 2000 2000 4096 Mar  8 13:27 models--nq_tydi_sq1-reader-xlmr_large-20221110
drwxrwxrwx 6 2000 2000 4096 Mar  8 13:24 models--xlm-roberta-base
-rwxrwxrwx 1 2000 2000    1 Mar  7 16:49 version.txt
```

These models are used for question answering, and must now be copied to the `PrimeA store`.  Note that the directory `models--nq_tydi_sq1-reader-xlmr_large-20221110` is empty so cannot be copied:

```
mkdir /root/gitRepos/create-primeqa-app/primeqa-store/models/nq_tydi_sq1-reader-xlmr_large-20221110
cd /root/gitRepos/create-primeqa-app/primeqa-store/models/nq_tydi_sq1-reader-xlmr_large-20221110
cp -L create-primeqa-app/cache/huggingface/hub/models--PrimeQA--nq_tydi_sq1-reader-xlmr_large-20221110/snapshots/59c0ac1e8c43a3c7f6d5e26e4bf1e9c3c53b850c/config.json .
cp -L create-primeqa-app/cache/huggingface/hub/models--PrimeQA--nq_tydi_sq1-reader-xlmr_large-20221110/snapshots/59c0ac1e8c43a3c7f6d5e26e4bf1e9c3c53b850c/pytorch_model.bin .
cp -L create-primeqa-app/cache/huggingface/hub/models--PrimeQA--nq_tydi_sq1-reader-xlmr_large-20221110/snapshots/59c0ac1e8c43a3c7f6d5e26e4bf1e9c3c53b850c/tokenizer.json .

mkdir /root/gitRepos/create-primeqa-app/primeqa-store/models/xlm-roberta-base
cd /root/gitRepos/create-primeqa-app/primeqa-store/models/xlm-roberta-base
cp -L /root/gitRepos/create-primeqa-app/cache/huggingface/hub/models--xlm-roberta-base/snapshots/42f548f32366559214515ec137cdd16002968bf6/config.json .
cp -L /root/gitRepos/create-primeqa-app/cache/huggingface/hub/models--xlm-roberta-base/snapshots/42f548f32366559214515ec137cdd16002968bf6/pytorch_model.bin .
cp -L /root/gitRepos/create-primeqa-app/cache/huggingface/hub/models--xlm-roberta-base/snapshots/42f548f32366559214515ec137cdd16002968bf6/tokenizer.json .
```

## Testing with PrimeQA UI

Open a browser to `PUBLIC_IP:82/qa`.

To help with composing some test searching, you can read a [brief summary of My Father Met a Dragon](https://www.supersummary.com/my-fathers-dragon/summary/).

### Test Retrieval

Retrieval searches the document index. Note, the first query will appear to fail while the index is loaded for the first time. The query results will appear after a few minutes.

![testRetrieval](/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/retrieval.png)

### Test Reading

Reading finds an answer only from the provided context information.

![testReading](/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/reading.png)

### Test Question Answering

Question Answering uses the Retriever to find documents in the index, and a Transformer model to propose a specific answer to the question. There are various settings which can influence the results, for example the minimum/maximum number of tokens for the answer. In this example, I reduced the number of answer tokens as I wanted to encourage short answers listing the animals the father met.

![testQA](/assets/img/2023-3-13-Using-PrimeQA-For-NLP-Question-Answering/qa.png)

