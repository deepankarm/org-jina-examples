# My First Jina App: Understanding

This is where we dive a little deeper to learn what happens inside each Flow and how they're built up from Pods.

## Flows

Everything in Jina is controlled by high-level [Flows](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#flow) that coordinate tasks and components. We define Flows for indexing and querying the content in our South Park dataset in `app.py`.

In this case we'll write our Flows in YAML format and load them into `app.py` with:

```python
from jina.flow import Flow
def index():
    f = Flow.load_config('flows/index.yml')
```

It really is that simple! Alternatively you can build Flows in `app.py` itself [without specifying Flows in YAML](https://docs.jina.ai/chapters/flow/index.html).

### Indexing

Every Flow has well, a flow to it. Different Pods pass data along the Flow, with one Pod's output becoming another Pod's input. Look at our indexing flow as an example:

![](images/index-flow.png)

Right now our data is just one big text file. Our Flow needs to process it into a useful state, which we do with Pods. Each Pod in the Flow performs a different task.

In Jina 101, we discussed [Documents and Chunks](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#document--chunk). In our indexing Flow, we:

* Break our giant text file into sentences. We'll regard each sentence as a Document (For simplicity, each Document has only one Chunk, containing the same sentence as the Document)
* Encode each sentence, as a Chunk, into a vector (in this case, using a Pod which specifies DistilBert from the [🤗Transformers library](https://huggingface.co/transformers))
* Build indexes for each Chunk and Document for fast lookup
* Store the vectors in our indexes

Our Pods perform all the tasks needed to make this happen:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `splitter`      | Split the text file into sentences                   |
| `encoder`       | Encode each sentence into a vector                   |
| `chunk_indexer` | Build an index of Chunks                             |
| `doc_indexer`   | Store the document content                           |
| `join_all`      | Join the `chunk_indexer` and `doc_indexer` pathways |

For indexing, we define which Pods to use in `flows/index.yml`. As you may have seen, cookiecutter already created some YAML files in `flows/` for us to start with. Let's break them down, starting with indexing:

```yaml
!Flow
with:
  logserver: true
```

Starts the flow and enables the logserver, so we can monitor our Flows with [Jina Dashboard](https://github.com/jina-ai/dashboard) later.

```yaml
pods:
  crafter:
    yaml_path: pods/craft.yml
    read_only: true
```
Starts our Pods section, and specifies our first Pod, named `crafter` which is defined in `pod/craft.yml`. `pods/craft.yml` is in turn is a YAML file which specifies the Pod's [Executor](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#executors) and other attributes.

```yaml
  encoder:
    yaml_path: pods/encode.yml
    replicas: $REPLICAS
    timeout_ready: 600000
    read_only: true
```
This code specifies:

* The encoder Pod and its path
* Replicas for XXX
* Timeout limits
* Read only attribute, so the Pod can't adjust the input data

```yaml
  chunk_idx:
    yaml_path: pods/chunk.yml
    replicas: $SHARDS
    separated_workspace: true
```

Similar to the above, but includes the `separated_workspaces` attribute to XXX

```yaml
  doc_idx:
    yaml_path: pods/doc.yml
    needs: gateway
```

This Pod specifies prerequisites, namely the `gateway` Pod. We can see this in the Flow diagram above.

```yaml
  join_all:
    yaml_path: _merge
    needs: [doc_idx, chunk_idx]
```

`join_all` looks like just another Pod - but what's with the `_merge` path? This is a built-in YAML that merges all the messages that come in from the Pods `needs`.

So, is that all of the Pods? Not quite! We always have another Pod working in silence - the `gateway` pod. Most of the time we can safely ignore it because it basically does all the dirty orchestration work for the Flow.

With all these Pods defined in our Flow, we're all set up to index all the character lines in our dataset.

### Querying

Just like indexing, the querying Flow is also defined in a YAML file. Much of it is similar to indexing:

![](images/flow-query.png)

```yaml
!Flow
with:
  read_only: true  # better add this in the query time
  rest_api: true
  port_grpc: $JINA_PORT
pods:
  chunk_seg:
    yaml_path: pods/craft.yml
    replicas: $REPLICAS
  tf_encode:
    yaml_path: pods/encode.yml
    replicas: $REPLICAS
    timeout_ready: 600000
  chunk_idx:
    yaml_path: pods/chunk.yml
    replicas: $SHARDS
    separated_workspace: true
    polling: all
    reducing_yaml_path: _merge_topk_chunks
    timeout_ready: 100000 # larger timeout as in query time will read all the data
  ranker:
    yaml_path: BiMatchRanker
  doc_idx:
    yaml_path: pods/doc.yml
```

In indexing we have to break down the Document into Chunks and index it. For querying we need to do the same with a user's query, and we can use many of the same Pods. There are a few differences though:

So in the query Flow we've got the following Pods:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `splitter`      | Split the user query into meaningful words           |
| `encoder`       | Encode each word into a vector                       |
| `chunk_indexer` | Build an index for the words for fast lookup         |
| `ranker`        | Combine query results into one document              |
| `doc_indexer`   | Store the document content                           |

Since many of the Pods are the same as in indexing, they share the same YAML but perform differently based on the task at hand.

### Index vs Query

Now that both our Flows are ready for action, let's take a quick look at the differences between them:

#### Code

Compared to `index.yml`, we have some extra features in `query.yml`:

| Line | Code                                     | Meaning                                                   |
| ---  | ---                                      | ---                                                       |
| 4    | `rest_api:true`                          | Use Jina's REST API, allowing jinabox and curl to connect |
| 5    | `port_grpc: $JINA_PORT`                  | Set GRPC port to environment variable                     |
| 18   | `polling: all`                           | XXX                                                       |
| 19   | `reducing_yaml_path: _merge_topk_chunks` | XXX                                                       |
| 21   | `ranker:`                                | A Pod to rank results by relevance                        |

#### Structures

While the two Flows share (most of) the same Pods, there are some differences in structure:

* Index has a two-pathway design which deals with both Document and Chunk indexing in parallel, which speeds up message passing
* Query has a single pipeline

#### Request Messages

* For indexing, we send an **IndexRequest** to the Flow
* For querying, we send a **SearchRequest** to the Flow

This is how Pods in both Flows can play different roles while sharing the same YAML files.

## Pods

As a convention in Jina, A YAML config is used to describe the properties of an object so that we can easily configure the behavior of the Pods without touching the application code.

Let's start by looking at our indexing Flow, `flows/index.yml`. Instead of the first Pod `crafter`, let's look at `encoder` which is a bit simpler:

```
pods:
...
  encoder:
    yaml_path: pods/encode.yml
    replicas: $REPLICAS
    timeout_ready: 600000
    read_only: true
```

The `encoder` Pod's YAML file is stored in `pods/encode.yml`:

```yaml
!TransformerTorchEncoder
with:
  pooling_strategy: cls
  model_name: distilbert-base-cased
  max_length: 96
```

We first use the built-in `TransformerTorchEncoder` as the **[Executor](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#executors)** in the Pod. The `with` field is used to specify the arguments passing to the `__init__()` function (XXX does this refer to the __init__ function of the `TransformerTorchEncoder` executor?):

| Argument           | Effect                                         |
| ---                | ---                                            |
| `pooling_strategy` | XXX                                            |
| `model_name`       | Name of the model we're using                  |
| `max_length`       | Maximum length of XXX                          |
