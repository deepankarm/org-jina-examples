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
