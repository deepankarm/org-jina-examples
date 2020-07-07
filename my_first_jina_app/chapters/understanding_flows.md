## Understanding Flows

Everything in Jina is controlled by high-level Flows that coordinate tasks and components. We're going to define Flows for indexing and querying the content in our South Park dataset.

In this case we'll define our Flows in YAML format and reference them in [the next chapter](07_run_flows.md) when we build our app. In later tutorials we'll look at writing Flows directly in Python, without so much YAML.

### Indexing

Every Flow has well, a flow to it. Different Pods pass data along the Flow, with one Pod's output becoming another Pod's input. Look at our indexing flow as an example:

XXX change image for cookiecutter
<img height="420px" src="./images/index-flow.png"/>

Right now our data is just one big text file. We need to process it into a useful state, which we do with Pods. Each Pod in the flow performs a different task.

In Jina 101, we discussed [Documents and Chunks](). In our indexing Flow, we:

* Break our giant text file into sentences. We'll regard each sentence as a Document (For simplicity, each Document has only one Chunk, containing the same sentence as the Document)
* Encode each sentence, as a Chunk, into a vector (in this case, DistilBert from the 🤗Transformers library)
* Store the vectors in our index

Our Pods perform all the tasks needed to make this happen:

| Pod             | Task                                                 |
| ---             | ---                                                  |
| `splitter`      | Split the text file into sentences                   |
| `encoder`       | Encode each sentence into a vector                   |
| `chunk_indexer` | Build an index for the sentences for fast lookup     |
| `doc_indexer`   | Store the document content                          |
| `join_all`      | Join the `chunk_indexer` and `doc_indexer` pathways |

We define which Pods to use in `flows/index.yml`. As you may have seen, cookiecutter already created some YAML files in `flows/` for us to start with. Let's break them down, starting with indexing:

TODO: Rewrite for cookiecutter standard files

```yaml
!Flow
with:
  logserver: true
```

Starts the flow and enables the logserver, so we'll have access to the dashboard later.

```yaml
pods:
  crafter:
    yaml_path: pods/craft.yml
    read_only: true
```
Starts our Pods section, and specifies our first Pod, named `crafter` which is defined in `pod/craft.yaml`. This in turn is a YAML file which specifies the Pod's Executor and other attributes.

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
* Read only attribute, so it won't adjust the input data

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

This Pod specifies prerequisites, namely the gateway Pod. We can see this in the Flow diagram above.

```yaml
  join_all:
    yaml_path: _merge
    needs: [doc_idx, chunk_idx]
```

`join_all` looks like just another Pod - but what's with the `_merge` path? This is a built-in YAML that merges all the messages that come in from the Pods `needs`.

So, is that all of the Pods? Not quite! We always have another Pod working in silence - the **gateway** pod. Most of the time we can safely ignore it because it basically does all the dirty orchestration work for the Flow.

With all these Pods defined in our Flow, we can index all the character lines in our dataset.

### Querying

Just like indexing, the querying Flow is also defined in a YAML file. Much of it is similar to indexing:

XXX change image for cookiecutter
<img height="420px" src="./images/query-flow.png"/>

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

| Line | Code                                     | Meaning                                                   |
| ---  | ---                                      | ---                                                       |
| 4    | `rest_api:true`                          | Use Jina's REST API, allowing jinabox and curl to connect |
| 5    | `port_grpc: $JINA_PORT`                  | Set GRPC port to environment variable                     |
| 18   | `polling: all`                           | XXX                                                       |
| 19   | `reducing_yaml_path: _merge_topk_chunks` | XXX                                                       |
| 21   | `ranker:`                                | A Pod to rank results by relevance                        |

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

#### Structures

While the two Flows share (most of) the same Pods, there are some differences in structure:

* Index has a two-pathway design which deals with both Document and Chunk indexing in parallel, which speeds up message passing
* Query has a single pipeline

#### Request Messages

* For indexing, we send an **IndexRequest** to the Flow
* For querying, we send a **SearchRequest** to the Flow

This is how Pods in both Flows can play different roles while sharing the same YAML files.
