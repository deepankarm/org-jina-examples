# Building Your First Jina App

## Introduction

In this series of tutorials we'll guide you on the process of building your own Jina search app. Don't worry if you're new to machine learning or search. We'll spell it all out right here.

Our example program will be a simple neural search engine for text. It will take a user's typed input, and return a list of lines from South Park that match most closely.

## Contents

* Key concepts
* Requirements
* Try it out!
* Set up
* Prepare data
* Define Flows
* Run Flows
* Dive into Pods

In this demo, we use Jina to build a semantic search system on the [SouthParkData](https://github.com/BobAdamsEE/SouthParkData/). The goal is to find out who said what in the South Park episodes when the user query a sentence. Many thanks to [BobAdamsEE](https://github.com/BobAdamsEE) for sharing this awesome dataset!👏. This demo will show you how to quickly build a search system from scratch with Jina. 



<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Try out this demo by yourself](#try-out-this-demo-by-yourself)
- [Prerequirements](#prerequirements)
- [Prepare the data](#prepare-the-data)
- [Define the Flows](#define-the-flows)
- [Run the Flows](#run-the-flows)
- [Dive into the Pods](#dive-into-the-pods)
- [Wrap up](#wrap-up)
- [Next Steps](#next-steps)
- [Documentation](#documentation)
- [Stay tuned](#stay-tuned)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->



## Overview

In this tutorial we'll build a BERT-based semantic search system using Natural Language Processing. We're using the SouthPark dataset, so a user can type in their input text, and the system will return lines from South Park that are similar.

<p align="center">
  <img src=".github/southpark.gif?raw=true" alt="Jina banner" width="90%">
</p>


## Try It Out

We've prepared a Docker image with indexed data. You can run it by the following command:

```bash
docker run -p 45678:45678 jinaai/hub.app.distilbert-southpark:latest
```

Now you can open your shell and check out the results via the RESTful API. The matched results are stored in `topkResults`.

```bash
curl --request POST -d '{"top_k": 10, "mode": "search",  "data": ["text:hey, dude"]}' -H 'Content-Type: application/json' 'http://0.0.0.0:45678/api/search'
```

Check out more details about the Docker image [here](rest-api/README.md).


## Prerequisites

Before starting, we assume you already have some knowledge of Jina. Before moving forward, We highly suggest to going through our lovely [Jina 101](https://github.com/jina-ai/jina/tree/master/docs/chapters/101) and [Jina "Hello, World!"👋🌍](https://github.com/jina-ai/jina#jina-hello-world-).

This demo requires Python 3.7.

```bash
pip install -r requirements.txt
```


## Prepare the Data

The raw data contains season, episode, character, and line information in the `.csv` format as follows:

```
Season,Episode,Character,Line
10,1,Stan,"You guys, you guys! Chef is going away.
"
10,1,Kyle,"Going away? For how long?
"
10,1,Stan,"Forever.
"
10,1,Chef,"I'm sorry boys.
```

Run the following script to clone the data and do some data wrangling. 

```bash
cd southpark-search
bash ./get_data.sh
```


## Define the Flows

Everything in Jina is controlled by a high-level Flow that coordinates tasks and components. We're going to define Flows for indexing and querying the content

### Indexing

Every Flow has well, a flow to it. Different Pods pass data along the Flow, with one Pod's output becoming another Pod's input. Look at our indexing flow as an example:

<img height="420px" src=".github/index-flow.png"/>

Right now our data is just one big text file. We need to process it into a useful state, which we do with Pods. Each Pod in the flow performs a different task:

In Jina 101, we discussed Documents and Chunks. In our indexing Flow, we:

* Break our giant text file into sentences. We'll regard each sentence as a Document
* For simplicity, each Document has only one Chunk, containing the same sentence as the Document
* Each sentence, as a Chunk, is encoded into a vector (in this case, DistilBert from the 🤗Transformers library)
* Store the vectors in our index

Our Pods perform all the tasks needed to make this happen:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `splitter`      | Split the text file into sentences                   |
| `encoder`       | Encode each sentence into a vector                   |
| `chunk_indexer` | Build an index for the sentences for fast lookup     |
| `doc_indexer`   | Store the document content                          |
| `join_all`      | Join the `chunk_indexer` and `doc_indexer` pathways |

We define which Pods to use in our `flow-index.yml` file. As you may have seen, we already have all of our finished YAML files in this directory for you to test, but let's break down `flow-index.yml`:

```yaml
!Flow
pods:
```

Initiate the flow and start specifying pods

```yaml
  splitter:
    yaml_path: yaml/craft-split.yml
```

Define our first pod, `splitter`, and point it to a YAML config file. Just as our Flow is defined in YAML, so too is each Pod.

```yaml
  encoder:
    yaml_path: yaml/encode.yml
    timeout_ready: 60000
```

Just as above, we define a Pod and point to a YAML file. But here we also specify `timeout_ready` to ensure we don't keep the Pod waiting too long.

Let's look at a couple more Pods:

```yaml
  chunk_indexer:
    yaml_path: yaml/index-chunk.yml
  doc_indexer:
    yaml_path: yaml/index-doc.yml
    needs: gateway
```
  
We can see from the above that we can define prerequisites via `needs:`. For other YAML options, please check [our documentation]()
  
```yaml
  join_all:
    yaml_path: _merge
    needs: [doc_indexer, chunk_indexer]
```

`join_all` looks like just another Pod - but what's with the `_merge` path? This is a built-in YAML that merges all the messages that come in from the Pods `needs`.

So, is that all of the Pods? Not quite! We always have another Pod working in silence - the **gateway** pod. Most of the time we can safely ignore it because it basically does all the dirty orchestration work for the Flow.

With all these Pods defined in our Flow, we can index all the character lines in our dataset.

### Querying

Just like indexing, the querying Flow is also defined in a YAML file. Much of it is similar to indexing:

<img height="420px" src=".github/query-flow.png"/>

```yaml
!Flow
with:
  read_only: true
pods:
  splitter:
    yaml_path: yaml/craft-split.yml
  encoder:
    yaml_path: yaml/encode.yml
    timeout_ready: 60000
  chunk_indexer:
    yaml_path: yaml/index-chunk.yml
  ranker:
    yaml_path: MinRanker
  doc_indexer:
    yaml_path: yaml/index-doc.yml
```

In indexing we have to break down the Document into Chunks and index it. For querying we need to do the same with a user's query, so we use many of the same Pods. There are a couple of differences though:

We've specified querying to be read-only, by using:

```yaml
with:
  read_only: true
```

And we've added a new Pod, `ranker`, which combines the query result Chunks into one Document to present to the end-user. In this example, we use the built-in `MinRanker` to do the job.

```yaml
ranker:
  yaml_path: MinRanker
```

So in the query Flow we've got the following Pods:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `splitter`      | Split the user query into meaningful words           |
| `encoder`       | Encode each word into a vector                       |
| `chunk_indexer` | Build an index for the words for fast lookup         |
| `ranker`        | Combine query results into one document              |
| `doc_indexer`   | Store the document content                           |

Since many of the Pods are the same as in indexing, they share the same YAML.

### Index vs Query

Now that both our Flows are ready for action, let's take a quick look at the differences between them:

#### Structures

While the two Flows share (most of) the same Pods, there are some differences in structure:

* Index has a two-pathway design which deals with both Document and Chunk indexing in parallel, which speeds up message passing
* Query has a single pipeline

#### Request Messages

* For indexing, we send an **IndexRequest** to the Flow
* For querying, we send a **SearchRequest** to the Flow

This is how Pods in both Flows can play different roles while sharing the same YAML files.

## Build the App

Enough definition! Now's time to get our Flows running in an app.

### Index

Start a new file (let's call it app.py) in your editor and we'll define some functions:

```python
def main(num_docs):
    flow = Flow().load_config('flow-index.yml')
    with flow.build() as fl:
        data_fn = os.path.join('/tmp/jina/southpark', 'character-lines.csv')
        fl.index(buffer=read_data(data_fn, num_docs))
```

In the code above, we're:

* Defining our main function
* Defining our new flow, to be loaded from `flow-index.yml`
* Defining our data filename
* Passing our data and the number of Documents through the `read_data` function to the Flow indexing function

Now we can run it with:

```bash
python app.py -t index -n 10000
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/index-demo.png?raw=true" alt="index flow console output">
</p>

</details>

#### The `read_data` Function

The content of the `IndexRequest` is fed from `read_data()`, which loads our processed CSV file and outputs each word together with its definition formatted into `bytes`.
Encoding the text with bert-family models takes a long time so we're limiting the number of indexed documents to 10,000.

```python
def read_data(f_path, max_sample_size=-1):
    if not os.path.exists(f_path):
        print('file not found: {}'.format(f_path))
    doc_list = []
    with open(f_path, 'r') as f:
        for l in f:
            doc_list.append(l.strip('\n').lower())
    if max_sample_size > 0:
        random.shuffle(doc_list)
        doc_list = doc_list[:max_sample_size]
    print('\n'.join(doc_list))
    for d in doc_list:
        yield d.encode('utf8')
```

### Query

```bash
python app.py -t query
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details>

We can follow the same process to define and build the query Flow from the YAML file. The `search()` function is used to send a `SearchRequest` to the Flow. Here we accept the user's query input from the terminal and wrap it into request message formatted into `bytes`.

```python
def read_query_data(text):
    yield '{}'.format(text).encode('utf8')

def main(top_k):
    flow = Flow().load_config('flow-query.yml')
    with flow.build() as fl:
        while True:
            text = input('please type a sentence: ')
            if not text:
                break
            ppr = lambda x: print_topk(x, text)
            fl.search(read_query_data(text), callback=ppr, topk=top_k)
```

The `callback` argument post-processes the returned message. In this demo, we define a simple `print_topk` function to show the results. The returned message `resp` in a Protobuf message. `resp.search.docs` contains all the Documents for searching, and in our case there is only one Document. For each query Document, the matched Documents, `.match_doc`, together with the matching score, `.score`, are stored under the `.topk_results` as a repeated variable.

```python
ppr = lambda x: print_topk(x, text)
fl.search(read_query_data(text), callback=ppr, topk=top_k)
```

```python
def print_topk(resp, word):
    for d in resp.search.docs:
        print(f'Ta-Dah🔮, here are what we found for: {word}')
        for idx, kk in enumerate(d.topk_results):
            score = kk.score.value
            if score <= 0.0:
                continue
            doc = kk.match_doc.buffer.decode()
            name, line = doc.split('!', maxsplit=1)
            print('> {:>2d}({:f}). {} said: {}'.format(
                idx, score, name.upper(), line))
```

## Dive into the Pods

**This should be its own doc**

If you want to know more about Pods, keep reading. As shown above, we defined the Pods by giving the YAML files. Now let's move on to see what is exactly written in these magic YAML files.

### `splitter`
As a convention in Jina, A YAML config is used to describe the properties of an object so that we can easily configure the behavior of the Pods without touching the code.

Here is the YAML file for the `splitter`. We first use the built-in **Sentencizer** as the **executor** in the Pod. The `with` field is used to specify the arguments passing to the `__init__()` function.


```yaml
!Sentencizer
with:
  min_sent_len: 3
  max_sent_len: 128
  punct_chars: '.,;!?:'
requests:
  on:
    [SearchRequest, IndexRequest, TrainRequest]:
      - !SegmentDriver
        with:
          method: craft
```

In the `requests` field, we define the different behaviors of the Pod for different requests. Remember that both the index and query Flows share the same Pods with the YAML files while they behave differently to the requests. This is how the magic works: For `SearchRequest`, `IndexRequest`, and `TrainRequest`, the `splitter` will use the `SegmentDriver`. On the one hand, the Driver encodes the request messages in [Protobuf format](https://en.wikipedia.org/wiki/Protocol_Buffers) into a format that the Executor can understand (e.g. a Numpy array). On the other hand, the `SegmentDriver` will call the `craft()` function from the Executor to handle the message and encode the processed results back into Protobuf format. For the time being, the `splitter` shows the same behavior for both requests.


```yaml
requests:
  on:
    [SearchRequest, IndexRequest]:
      - !SegmentDriver
        with:
          method: craft
```

### `encoder`
The YAML file of the `encoder` is pretty similar to the `splitter`. As one can see, we specify the `distilbert-base-cased` model. One can easily switch to other fancy pretrained models from **transformers** by giving another `model_name`.

```yaml
!TransformerTorchEncoder
with:
  pooling_strategy: cls
  model_name: distilbert-base-cased
  max_length: 96
requests:
  on:
    [SearchRequest, IndexRequest]:
      - !EncodeDriver
        with:
          method: encode
```

### `doc_indexer`
In contrast to the Pods above, the `doc_indexer` behave differently depending on different requests. For the `IndexRequest`, the Pod uses `DocPruneDriver` and the `DocKVIndexDRiver` in sequence. The `DocPruneDriver` prunes the redundant data in the message that is not used by the downstream Pods. Here we discard all the data in the `chunks` field because we only want to save the Document level data.

```yaml
!DocPbIndexer
with:
  index_filename: doc.gzip
metas:
  name: docIndexer
  workspace: $TMP_WORKSPACE
requests:
  on:
    SearchRequest:
      - !DocKVSearchDriver
        with:
          method: query

    IndexRequest:
      - !DocPruneDriver
        with:
          pruned: ['chunks']
      - !DocKVIndexDriver
        with:
          method: add
```

For the `SearchRequest`, the Pod uses the same `DocKVSearchDriver` just like the `IndexRequest`, but the Driver calls different functions in the Executor, `LeveldbIndexer`.
```yaml
requests:
  on:
    SearchRequest:
      - !DocKVSearchDriver
      with:
        method: query
```

### `chunk_indexer`
The YAML file for `chunk_indexer` is a little bit cumbersome. But take it easy: It is as straightforward as it should be. The executor in the `chunk_indexer` Pod is called `ChunkIndexer`, which wraps two other executors. The `components` field specifies the two wrapped executors: The `NumpyIndexer` stores the Chunks' vectors, and the `BasePbIndexer` is used as a key-value storage to save the Chunk ID and Chunk details.

```yaml
!ChunkIndexer
components:
  - !NumpyIndexer
    with:
      index_filename: vec.gz
      metrix: cosine
  - !ChunkPbIndexer
    with:
      index_filename: chunk.gz
requests:
  on:
    IndexRequest:
      - !VectorIndexDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVIndexDriver
        with:
          level: chunk
          executor: BasePbIndexer
    SearchRequest:
      - !VectorSearchDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVSearchDriver
        with:
          level: chunk
          executor: BasePbIndexer

```

Just like the `doc_indexer`, the `chunk_indexer` has different behaviors for different requests. For the `IndexRequest`, the `chunk_indexer` uses three Drivers in serial, namely, `VectorIndexDriver`, `PruneDriver`, and `KVIndexDriver`. The idea is to first use `VectorIndexDriver` to call the `index()` function from the `NumpyIndexer` so that the vectors for all the Chunks are indexed. Then the `PruneDriver` prunes the message, and the `KVIndexDriver` calls the `index()` function from the `BasePbIndexer`. This behavior is defined in the `executor` field.

```yaml
requests:
  on:
    IndexRequest:
      - !VectorIndexDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVIndexDriver
        with:
          level: chunk
          executor: BasePbIndexer
```

For the `SearchRequest`, the same procedure continues, but under the hood the `KVSearchDriver` and the `VectorSearchDriver` call the `query()` function from the `NumpyIndexer` and `BasePbIndexer` correspondingly.

```yaml
requests:
  on:
    SearchRequest:
      - !VectorSearchDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVSearchDriver
        with:
          level: chunk
          executor: BasePbIndexer
```

## Wrap up

Congratulations! Now you've got your very own neural search engine explained!

Let's wrap up what we've covered in this demo.

1. Chunks are basic elements in Jina. Documents are the final inputs and outputs from Jina.
2. Flows are our good friends to build the search engine. To either index or query, we need to define and build a Flow.
3. The index and the query Flow shares most Pods.
4. Pods in the Flow can run either in serial or in parallel.
5. Pods can behave differently to different types of requests. The Pods' YAML file defines their behaviors.
6. The data in the requests should be in `bytes` format.
7. Inside the Pods, Drivers are used to interpret the messages for Executors, call the Executors to handle the messages, and wrap the results back into the message.
8. The query results in the return response are saved in the Protobuf format.

**Enjoy Coding with Jina!**

## Next Steps
- Try other encoders or rankers.
- Run the Pods in docker containers.
- Scale up the indexing procedure.
- Speed up the procedure by using `read_only`
- Explore the `metas` field in the YAML file.

## Documentation

<a href="https://docs.jina.ai/">
<img align="right" width="350px" src="https://github.com/jina-ai/jina/blob/master/.github/jina-docs.png" />
</a>

The best way to learn Jina in depth is to read our documentation. Documentation is built on every push, merge, and release event of the master branch. You can find more details about the following topics in our documentation.

- [Jina command line interface arguments explained](https://docs.jina.ai/chapters/cli/main.html)
- [Jina Python API interface](https://docs.jina.ai/api/jina.html)
- [Jina YAML syntax for executor, driver and flow](https://docs.jina.ai/chapters/yaml/yaml.html)
- [Jina Protobuf schema](https://docs.jina.ai/chapters/proto/main.html)
- [Environment variables used in Jina](https://docs.jina.ai/chapters/envs.html)
- ... [and more](https://docs.jina.ai/index.html)

## Stay tuned

- [Slack chanel](https://join.slack.com/t/jina-ai/shared_invite/zt-dkl7x8p0-rVCv~3Fdc3~Dpwx7T7XG8w) - a communication platform for developers to discuss Jina
- [Community newsletter](mailto:newsletter+subscribe@jina.ai) - subscribe to the latest update, release and event news of Jina
- [LinkedIn](https://www.linkedin.com/company/jinaai/) - get to know Jina AI as a company
- ![Twitter Follow](https://img.shields.io/twitter/follow/JinaAI_?label=Follow%20%40JinaAI_&style=social) - follow us and interact with us using hashtag `#JinaSearch`
- [Join Us](mailto:hr@jina.ai) - want to work full-time with us at Jina? We are hiring!
- [Company](https://jina.ai) - know more about our company, we are fully committed to open-source!


## License

Copyright (c) 2020 Jina AI Limited. All rights reserved.

Jina is licensed under the Apache License, Version 2.0. See [LICENSE](https://github.com/jina-ai/jina/blob/master/LICENSE) for the full license text.

---

Start 

To index the data we first need to define our **Flow** with a **YAML** file. In the Flow YAML file, we add **Pods** in sequence. In this demo, we have 5 pods defined with the names `splitter`, `encoder`, `chunk_indexer`, `doc_indexer`, and `join_all`.

However, we have another Pod working in silence. In fact, the input to the very first Pod is always the Pod with the name **gateway**, the "Forgotten" Pod. For most of the time, we can safely ignore the **gateway** because it basically does the dirty orchestration work for the Flow.

<table style="margin-left:auto;margin-right:auto;">
<tr>
<td> flow-index.yml</td>
<td> Flow in Dashboard</td>
</tr>
<tr>
<td>
  <sub>

```yaml
!Flow
pods:
  splitter:
    yaml_path: yaml/craft-split.yml
  encoder:
    yaml_path: yaml/encode.yml
    timeout_ready: 60000
  chunk_indexer:
    yaml_path: yaml/index-chunk.yml
  doc_indexer:
    yaml_path: yaml/index-doc.yml
    needs: gateway
  join_all:
    yaml_path: _merge
    needs: [doc_indexer, chunk_indexer]
```

</sub>

</td>
<td>
<img align="right" height="420px" src=".github/index-flow.png"/>
</td>
</tr>
</table>

By default, the input of each Pod is the Pod defined right above it, and the request message flows from one Pod to another. That's how Flow lives up to its name. Of course, you might want to have a Pod skipping over the Pods above it. In this case, you would use the `needs` argument to specify the source of the input messages. In our example, the `doc_indexer` actually get inputs directly from the `gateway`. By doing this, we have two parallel pathways as shown in the index Flow diagram.

```yaml
doc_indexer:
  yaml_path: yaml/index-doc.yml
  needs: gateway
```

As we can see, for most Pods, we only need to define the YAML file path. Given the YAML files, Jina will automatically build the Pods. Plus, `timeout_ready` is a useful argument when adding a Pod, which defines the waiting time before the Flow gives up on the Pod initializing.

```yaml
encoder:
  yaml_path: yaml/encode.yml
  timeout_ready: 60000
```

You might also notice the `join_all` Pod has a special YAML path. It denotes a built-in YAML path, which will merge all the incoming messages defined in `needs`.

```yaml
join_all:
  yaml_path: _merge
  needs: [doc_indexer, chunk_indexer]
```

Overall, the index Flow has two pathways, as shown in the Flow diagram. The idea is to save the index and the contents seperately so that we can quickly retrieve the Document IDs from the index and afterwards combine the Document ID with its content.

The pathway on the right side with a single `doc_indexer` stores the Document content. Under the hood it is basically a key-value storage. The key is the Document ID and the value is the Document itself.

The pathway on the other side is for saving the index. From top to bottom, the first Pod, `splitter`, splits the Document into Chunks, which are the basic units to process in Jina. Chunks are later encoded into vectors by the `encoder`. These vectors (together with other information in the Chunks) are saved in a vector storage by `chunk_indexer`. Finally, the two pathways are merged by `join_all` and the processing of that message is concluded.


### Query
As in the indexing time, we also need a Flow to process the request message during querying. Here we start with the `splitter` sharing exactly the same YAML with its conterpart in the index Flow. This means it plays the same role as before, which is to split the Document into Chunks. Afterwards, the Chunks are encoded into vectors by `encoder`, and later these vectors are used to retrieve the indexed Chunks by `chunk_indexer`. At the same as the `splitter`, both `encoder` and `chunk_indexer` share the YAML with their counterparts in the index Flow.

<table  style="margin-left:auto;margin-right:auto;">
<tr>
<td> flow-query.yml</td>
<td> Flow in Dashboard</td>
</tr>
<tr>
<td>
  <sub>

```yaml
!Flow
with:
  read_only: true
pods:
  splitter:
    yaml_path: yaml/craft-split.yml
  encoder:
    yaml_path: yaml/encode.yml
    timeout_ready: 60000
  chunk_indexer:
    yaml_path: yaml/index-chunk.yml
  ranker:
    yaml_path: MinRanker
  doc_indexer:
    yaml_path: yaml/index-doc.yml
```

</sub>

</td>
<td>
<img align="right" height="420px" src=".github/query-flow.png"/>
</td>
</tr>
</table>


Eventually, there comes a new Pod named `ranker`. Remember that Chunks are the basic units in Jina. In the deep core of Jina, both indexing and quering take place at the Chunk level. Chunks are the elements that the Jina core can understand and process. However, we need to ship the final query results in the form of a Document which is actually meaningful to users. This is precisely the job of `ranker`. `ranker` combines the query results from the Chunk level into the Document level. In this demo, we use the built-in `MinRanker` to do the job. It simple takes the `1 / (1 + s)` as the score of the Document, where `s` denotes the minimal matching score from all the Chunks that belong to this Document. Why do we take the minimal matching score for Chunks? Because here we use the **cosine distance** as the chunks' matching scores.

> `MaxRanker` calculates the score of the matched Document from the matched Chunks. For each matched Document, the score is the maximal score from all the matched Chunks belonging to that Document.

```yaml
ranker:
  yaml_path: MaxRanker
```

In the last step, the `doc_indexer` comes into play. Sharing the same YAML file, `doc_indexer` will load the stored key-value index and retrieve the matched Documents according to the Document ID.

### Let's take a closer look
Now the index and query Flows are both ready to work. Before proceeding, let's take a closer look at the two Flows and see the differences between them.

Obviously, they have different structures, although they share most Pods. This is a common practice in the Jina world for the sake of speed. Except for the `ranker`, both Flows can indeed use identical structures. The two-pathway design of the index Flow is intended to speed up message passing, because indexing Chunks and Documents can be done in parallel.

Another important difference is that the two Flows are used to process different types of request messages. To index a Document, we send an **IndexRequest** to the Flow. While querying, we send a **SearchRequest**. That's why Pods in both Flows can play different roles while sharing the same YAML files. Later, we will dive deep into into the YAML files, where we define the different ways of processing messages of various types.


## Run the Flows

### Index


```bash
python app.py -t index -n 10000
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/index-demo.png?raw=true" alt="index flow console output">
</p>

</details>

With the Flows, we can now write the code to run the Flow. For indexing, we start by defining the Flow with a YAML file. Afterwards, the `build()` function will do the magic to construct Pods and connect them together. After that, the `IndexRequest` will be sent to the flow by calling the `index()` function.

```python
def main(num_docs):
    flow = Flow().load_config('flow-index.yml')
    with flow.build() as fl:
        data_fn = os.path.join('/tmp/jina/southpark', 'character-lines.csv')
        fl.index(buffer=read_data(data_fn, num_docs))

```

The content of the `IndexRequest` is fed from `read_data()`, which loads the processed JSON file and outputs each word together with its definition formatted into `bytes`.
Encoding the text with bert-family models takes a long time. To save your time, here we limit the number of indexed documents to 10,000.

```python
def read_data(f_path, max_sample_size=-1):
    if not os.path.exists(f_path):
        print('file not found: {}'.format(f_path))
    doc_list = []
    with open(f_path, 'r') as f:
        for l in f:
            doc_list.append(l.strip('\n').lower())
    if max_sample_size > 0:
        random.shuffle(doc_list)
        doc_list = doc_list[:max_sample_size]
    print('\n'.join(doc_list))
    for d in doc_list:
        yield d.encode('utf8')
```

### Query

```bash
python app.py -t query
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details>

For querying, we follow the same process to define and build the Flow from the YAML file. The `search()` function is used to send a `SearchRequest` to the Flow. Here we accept the user's query input from the terminal and wrap it into request message formatted into `bytes`.

```python
def read_query_data(text):
    yield '{}'.format(text).encode('utf8')

def main(top_k):
    flow = Flow().load_config('flow-query.yml')
    with flow.build() as fl:
        while True:
            text = input('please type a sentence: ')
            if not text:
                break
            ppr = lambda x: print_topk(x, text)
            fl.search(read_query_data(text), callback=ppr, topk=top_k)
```

The `callback` argument is used to post-process the returned message. In this demo, we define a simple `print_topk` function to show the results. The returned message `resp` in a Protobuf message. `resp.search.docs` contains all the Documents for searching, and in our case there is only one Document. For each query Document, the matched Documents, `.match_doc`, together with the matching score, `.score`, are stored under the `.topk_results` as a repeated variable.

```python
ppr = lambda x: print_topk(x, text)
fl.search(read_query_data(text), callback=ppr, topk=top_k)
```

```python
def print_topk(resp, word):
    for d in resp.search.docs:
        print(f'Ta-Dah🔮, here are what we found for: {word}')
        for idx, kk in enumerate(d.topk_results):
            score = kk.score.value
            if score <= 0.0:
                continue
            doc = kk.match_doc.buffer.decode()
            name, line = doc.split('!', maxsplit=1)
            print('> {:>2d}({:f}). {} said: {}'.format(
                idx, score, name.upper(), line))
```

## Dive into the Pods
If you want to know more about Pods, keep reading. As shown above, we defined the Pods by giving the YAML files. Now let's move on to see what is exactly written in these magic YAML files.

### `splitter`
As a convention in Jina, A YAML config is used to describe the properties of an object so that we can easily configure the behavior of the Pods without touching the code.

Here is the YAML file for the `splitter`. We first use the built-in **Sentencizer** as the **executor** in the Pod. The `with` field is used to specify the arguments passing to the `__init__()` function.


```yaml
!Sentencizer
with:
  min_sent_len: 3
  max_sent_len: 128
  punct_chars: '.,;!?:'
requests:
  on:
    [SearchRequest, IndexRequest, TrainRequest]:
      - !SegmentDriver
        with:
          method: craft
```

In the `requests` field, we define the different behaviors of the Pod for different requests. Remember that both the index and query Flows share the same Pods with the YAML files while they behave differently to the requests. This is how the magic works: For `SearchRequest`, `IndexRequest`, and `TrainRequest`, the `splitter` will use the `SegmentDriver`. On the one hand, the Driver encodes the request messages in [Protobuf format](https://en.wikipedia.org/wiki/Protocol_Buffers) into a format that the Executor can understand (e.g. a Numpy array). On the other hand, the `SegmentDriver` will call the `craft()` function from the Executor to handle the message and encode the processed results back into Protobuf format. For the time being, the `splitter` shows the same behavior for both requests.


```yaml
requests:
  on:
    [SearchRequest, IndexRequest]:
      - !SegmentDriver
        with:
          method: craft
```

### `encoder`
The YAML file of the `encoder` is pretty similar to the `splitter`. As one can see, we specify the `distilbert-base-cased` model. One can easily switch to other fancy pretrained models from **transformers** by giving another `model_name`.

```yaml
!TransformerTorchEncoder
with:
  pooling_strategy: cls
  model_name: distilbert-base-cased
  max_length: 96
requests:
  on:
    [SearchRequest, IndexRequest]:
      - !EncodeDriver
        with:
          method: encode
```

### `doc_indexer`
In contrast to the Pods above, the `doc_indexer` behave differently depending on different requests. For the `IndexRequest`, the Pod uses `DocPruneDriver` and the `DocKVIndexDRiver` in sequence. The `DocPruneDriver` prunes the redundant data in the message that is not used by the downstream Pods. Here we discard all the data in the `chunks` field because we only want to save the Document level data.

```yaml
!DocPbIndexer
with:
  index_filename: doc.gzip
metas:
  name: docIndexer
  workspace: $TMP_WORKSPACE
requests:
  on:
    SearchRequest:
      - !DocKVSearchDriver
        with:
          method: query

    IndexRequest:
      - !DocPruneDriver
        with:
          pruned: ['chunks']
      - !DocKVIndexDriver
        with:
          method: add
```

For the `SearchRequest`, the Pod uses the same `DocKVSearchDriver` just like the `IndexRequest`, but the Driver calls different functions in the Executor, `LeveldbIndexer`.
```yaml
requests:
  on:
    SearchRequest:
      - !DocKVSearchDriver
      with:
        method: query
```

### `chunk_indexer`
The YAML file for `chunk_indexer` is a little bit cumbersome. But take it easy: It is as straightforward as it should be. The executor in the `chunk_indexer` Pod is called `ChunkIndexer`, which wraps two other executors. The `components` field specifies the two wrapped executors: The `NumpyIndexer` stores the Chunks' vectors, and the `BasePbIndexer` is used as a key-value storage to save the Chunk ID and Chunk details.

```yaml
!ChunkIndexer
components:
  - !NumpyIndexer
    with:
      index_filename: vec.gz
      metrix: cosine
  - !ChunkPbIndexer
    with:
      index_filename: chunk.gz
requests:
  on:
    IndexRequest:
      - !VectorIndexDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVIndexDriver
        with:
          level: chunk
          executor: BasePbIndexer
    SearchRequest:
      - !VectorSearchDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVSearchDriver
        with:
          level: chunk
          executor: BasePbIndexer

```

Just like the `doc_indexer`, the `chunk_indexer` has different behaviors for different requests. For the `IndexRequest`, the `chunk_indexer` uses three Drivers in serial, namely, `VectorIndexDriver`, `PruneDriver`, and `KVIndexDriver`. The idea is to first use `VectorIndexDriver` to call the `index()` function from the `NumpyIndexer` so that the vectors for all the Chunks are indexed. Then the `PruneDriver` prunes the message, and the `KVIndexDriver` calls the `index()` function from the `BasePbIndexer`. This behavior is defined in the `executor` field.

```yaml
requests:
  on:
    IndexRequest:
      - !VectorIndexDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVIndexDriver
        with:
          level: chunk
          executor: BasePbIndexer
```

For the `SearchRequest`, the same procedure continues, but under the hood the `KVSearchDriver` and the `VectorSearchDriver` call the `query()` function from the `NumpyIndexer` and `BasePbIndexer` correspondingly.

```yaml
requests:
  on:
    SearchRequest:
      - !VectorSearchDriver
        with:
          executor: NumpyIndexer
      - !PruneDriver
        with:
          level: chunk
          pruned:
            - embedding
            - buffer
            - blob
            - text
      - !KVSearchDriver
        with:
          level: chunk
          executor: BasePbIndexer
```

## Wrap up
Congratulations! Now you've got your very own neural search engine explained!

Let's wrap up what we've covered in this demo.

1. Chunks are basic elements in Jina. Documents are the final inputs and outputs from Jina.
2. Flows are our good friends to build the search engine. To either index or query, we need to define and build a Flow.
3. The index and the query Flow shares most Pods.
4. The Pods in the Flow can run either in serial or in parallel.
5. The Pods can behave differently to different types of requests. The Pods' YAML file defines their behaviors.
6. The data in the requests should be in `bytes` format.
7. Inside the Pods, Drivers are used to interpret the messages for Executors, call the Executors to handle the messages, and wrap the results back into the message.
8. The query results in the return response are saved in the Protobuf format.

**Enjoy Coding with Jina!**

## Next Steps
- Try other encoders or rankers.
- Run the Pods in docker containers.
- Scale up the indexing procedure.
- Speed up the procedure by using `read_only`
- Explore the `metas` field in the YAML file.

## Documentation

<a href="https://docs.jina.ai/">
<img align="right" width="350px" src="https://github.com/jina-ai/jina/blob/master/.github/jina-docs.png" />
</a>

The best way to learn Jina in depth is to read our documentation. Documentation is built on every push, merge, and release event of the master branch. You can find more details about the following topics in our documentation.

- [Jina command line interface arguments explained](https://docs.jina.ai/chapters/cli/main.html)
- [Jina Python API interface](https://docs.jina.ai/api/jina.html)
- [Jina YAML syntax for executor, driver and flow](https://docs.jina.ai/chapters/yaml/yaml.html)
- [Jina Protobuf schema](https://docs.jina.ai/chapters/proto/main.html)
- [Environment variables used in Jina](https://docs.jina.ai/chapters/envs.html)
- ... [and more](https://docs.jina.ai/index.html)

## Stay tuned

- [Slack chanel](https://join.slack.com/t/jina-ai/shared_invite/zt-dkl7x8p0-rVCv~3Fdc3~Dpwx7T7XG8w) - a communication platform for developers to discuss Jina
- [Community newsletter](mailto:newsletter+subscribe@jina.ai) - subscribe to the latest update, release and event news of Jina
- [LinkedIn](https://www.linkedin.com/company/jinaai/) - get to know Jina AI as a company
- ![Twitter Follow](https://img.shields.io/twitter/follow/JinaAI_?label=Follow%20%40JinaAI_&style=social) - follow us and interact with us using hashtag `#JinaSearch`
- [Join Us](mailto:hr@jina.ai) - want to work full-time with us at Jina? We are hiring!
- [Company](https://jina.ai) - know more about our company, we are fully committed to open-source!


## License

Copyright (c) 2020 Jina AI Limited. All rights reserved.

Jina is licensed under the Apache License, Version 2.0. See [LICENSE](https://github.com/jina-ai/jina/blob/master/LICENSE) for the full license text.
