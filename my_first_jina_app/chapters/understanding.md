# My First Jina App: Understanding

This is where we dive deeper to learn what happens inside each Flow and how they're built up from Pods.

## 🌱 Flows

<img src="https://raw.githubusercontent.com/jina-ai/jina/master/docs/chapters/101/img/ILLUS10.png" width="30%" align="left">

As we saw in [Jina 101](https://github.com/jina-ai/jina/tree/master/docs/chapters/101), just as a plant manages nutrient flow and growth rate for its branches, a Flow manages the states and context of a group of Pods, orchestrating them to accomplish one task. Whether a Pod is remote or running in Docker, one Flow rules them all!

We define Flows in `app.py` to index and query the content in our South Park dataset.

In this case we'll write our Flows in YAML format and load them into `app.py` with:

```python
from jina.flow import Flow
def index():
    f = Flow.load_config('flows/index.yml')
```

It really is that simple! Alternatively you can build Flows in `app.py` itself [without specifying Flows in YAML](https://docs.jina.ai/chapters/flow/index.html).

### Indexing

Every Flow has well, a flow to it. Different Pods pass data along the Flow, with one Pod's output becoming another Pod's input. Look at our indexing Flow as an example:

<p align="center">
<img src="images/flow-index.png">
</p>

Right now our South Park dataset is just one big text file. Our Flow will process it into a useful state, which is handled by the Pods in the Flow. Each Pod performs a different task.

In Jina 101, we discussed [Documents and Chunks](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#document--chunk). In our indexing Flow, we:

* Break our giant text file into sentences. We'll regard each sentence as a Document (For simplicity, each Document has only one Chunk, containing the same sentence as the Document)
* Encode each sentence, as a Chunk, into a vector (in this case, using a Pod which specifies `distilbert` from the [🤗Transformers library](https://huggingface.co/transformers))
* Build indexes for each Chunk and Document for fast lookup
* Store the vectors in our indexes

Our Pods perform all the tasks needed to make this happen:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `crafter`       | Split the Document into Chunks                       |
| `encoder`       | Encode each Chunk into a vector                      |
| `chunk_idx`     | Build an index of Chunks                             |
| `doc_idx`       | Store the Document content                           |
| `join_all`      | Join the `chunk_indexer` and `doc_indexer` pathways |


#### Diving into `index.yml`

For indexing, we define which Pods to use in `flows/index.yml`. As you may have seen, cookiecutter already created some YAML files in `flows/` for us to start with. Let's break them down, starting with indexing:

<table>
<tr>
<th scope="col">
Code
</td>
<th scope="col">
What it does
</td>
</tr>
<tr>
<td>

```yaml
!Flow
  with:
  logserver: true
```

</td>
<td>

Starts the flow and enables the logserver, so we can monitor our Flows with [Jina Dashboard](https://github.com/jina-ai/dashboard) later.

</td>
</tr>
<tr>
<td>

```yaml
pods:
  crafter:
    yaml_path: pods/craft.yml
    read_only: true
```

</td>
<td>

Starts our Pods section, and specifies our first Pod, named `crafter` which is defined in `pod/craft.yml`. `pods/craft.yml` is in turn is a YAML file which specifies the Pod's [Executor](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#executors) and other attributes.

</td>
</tr>
<tr>
<td>

```yaml
  encoder:
    yaml_path: pods/encode.yml
    replicas: $REPLICAS
    timeout_ready: 600000
    read_only: true
```

</td>
<td>

This code specifies:

* The encoder Pod and its path
* Replicas for parallel processing
* Timeout limits
* Read only attribute, so the Pod can't adjust the input data

</td>
</tr>
<tr>
<td>

```yaml
  chunk_idx:
    yaml_path: pods/chunk.yml
    replicas: $SHARDS
    separated_workspace: true
```

</td>
<td>

Similar to the above, but includes the `separated_workspaces` attribute in order to XXX

</td>
</tr>
<td>

```yaml
  doc_idx:
    yaml_path: pods/doc.yml
    needs: gateway
```

</td>
<td>

This Pod specifies prerequisites, namely the `gateway` Pod. We can see this in the Flow diagram above.

</td>
</tr>
<tr>
<td>

```yaml
  join_all:
    yaml_path: _merge
    needs: [doc_idx, chunk_idx]
```

</td>
<td>

`join_all` looks like just another Pod - but what's with the `_merge` path? This is a built-in YAML that merges all the messages that come in from the Pods `needs`.

</td>
</tr>
</table>

So, is that all of the Pods? Not quite! We always have another Pod working in silence - the `gateway` pod. Most of the time we can safely ignore it because it basically does all the dirty orchestration work for the Flow.

With all these Pods defined in our Flow, we're all set up to index all the character lines in our dataset.

### Querying

Just like indexing, the querying Flow is also defined in a YAML file. Much of it is similar to indexing:

<table>
<tr>
<td>

![](images/flow-query.png)

</td>
<td>

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

</td>
</tr>
</table>

In indexing we have to break down the Document into Chunks and index it. For querying we do the same, regarding the query as a Document, and we can use many of the same Pods. There are a few differences though:

So, in the query Flow we've got the following Pods:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `chunk_seg`     | Segments the user query into meaningful Chunks       |
| `encoder`       | Encode each word into a vector                       |
| `tf_encode`     | XXX                                                  |
| `chunk_indexer` | Build an index for the Chunks for fast lookup        |
| `ranker`        | Combine returned results into one Document           |
| `doc_idx`       | Store the Document content                           |

Since many of the Pods are the same as in indexing, they share the same YAML but perform differently based on the task at hand.

### Index vs Query

Now that both our Flows are ready for action, let's take a quick look at the differences between them:

#### Code

Compared to `index.yml`, we have some extra features in `query.yml`:

| Code                                     | Meaning                                                                  |
| ---                                      | ---                                                                      |
| `rest_api:true`                          | Use Jina's REST API, allowing clients like jinabox and `curl` to connect |
| `port_grpc: $JINA_PORT`                  | The port for connecting to Jina's API                                    |
| `polling: all`                           | Setting `polling` to `all` ensures all workers poll the message          |
| `reducing_yaml_path: _merge_topk_chunks` | Use `_merge_topk_chunks` to reduce result from all replicas              |
| `ranker:`                                | A Pod to rank results by relevance                                       |

#### Structures

While the two Flows share (most of) the same Pods, there are some differences in structure:

* Index has a two-pathway design which deals with both Document and Chunk indexing in parallel, which speeds up message passing
* Query has a single pipeline

#### Request Messages

In our RESTful API we set the `mode` field in the JSON body and send the request to the corresponding API:

| API endpoint | JSON
| ---          | ---                  |
| `api/index`  | `{"mode": "index"}`  |
| `api/search` | `{"mode": "search"}` |

This is how Pods in both Flows can play different roles while sharing the same YAML files.

## Pods

<img src="https://raw.githubusercontent.com/jina-ai/jina/master/docs/chapters/101/img/ILLUS8.png" width="20%" align="left">

You can think of the Flow as telling Jina *what* tasks to perform on the dataset. The Pods comprise the Flow and tell Jina *how* to perform each task, and they define the actual neural networks we use in neural search, namely the machine-learning models like `distilbert-base-cased`.

As you may recall from [Jina 101](https://github.com/jina-ai/jina/tree/master/docs/chapters/101), A Pod is a group of Peas with the same property, running in parallel on a local host or over the network. A Pod provides a single network interface for its Peas, making them look like one single Pea from the outside. Beyond that, a Pod adds further control, scheduling, and context management to the Peas.

As a convention in Jina, we use YAML files to describe objects like Flows and Pods. This way we can easily configure the behavior of the Pods without touching their application code.

Let's start by looking at our indexing Flow, `flows/index.yml`. Instead of the first Pod `crafter`, let's look at `encoder` which is a bit simpler:

```yaml
pods:

  <other pods here>

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

We first use the built-in `TransformerTorchEncoder` as the **[Executor](https://github.com/jina-ai/jina/tree/master/docs/chapters/101#executors)** in the Pod. The `with` field is used to specify the parameters that we pass to `TransformerTorchEncoder`.

| Parameter          | Effect                                                 |
| ---                | ---                                                    |
| `pooling_strategy` | Strategy to merge word embeddings into chunk embedding |
| `model_name`       | Name of the model we're using                          |
| `max_length`       | Maximum length to truncate tokenized sequences to      |

<table width="100%">
  <tr>
    <td align="left" style="text-align:right">
      <strong><a href="./run.md">⬅️ Previous: Running</a></strong>
    </td>
    <td align="right" style="text-align:right">
      <strong><a href="./troubleshooting.md">Next: Troubleshooting ➡️</a></strong>
    </td>
  </tr>
</table>
