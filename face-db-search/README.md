<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Build an Face Search System in 3 minutes](#build-an-face-search-system-in-3-minutes)
  - [<a name="custom-encoder">Overview</a>](#a-namecustom-encoderoverviewa)
  - [Prerequirements](#prerequirements)
  - [Prepare the data](#prepare-the-data)
  - [Define the Flows](#define-the-flows)
  - [<a name="custom-encoder">Custom Encoder</a>](#a-namecustom-encodercustom-encodera)
  - [Run the Flows](#run-the-flows)
  - [Next Steps](#next-steps)
  - [Documentation](#documentation)
  - [Community](#community)
  - [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Build an Face Search System in 3 minutes
<p align="center">
 
[![Jina](https://github.com/jina-ai/jina/blob/master/.github/badges/jina-badge.svg "We fully commit to open-source")](https://jina.ai)
[![Jina](https://github.com/jina-ai/jina/blob/master/.github/badges/jina-hello-world-badge.svg "Run Jina 'Hello, World!' without installing anything")](https://github.com/jina-ai/jina#jina-hello-world-)
[![Jina](https://github.com/jina-ai/jina/blob/master/.github/badges/license-badge.svg "Jina is licensed under Apache-2.0")](#license)
[![Jina Docs](https://github.com/jina-ai/jina/blob/master/.github/badges/docs-badge.svg "Checkout our docs and learn Jina")](https://docs.jina.ai)
[![We are hiring](https://github.com/jina-ai/jina/blob/master/.github/badges/jina-corp-badge-hiring.svg "We are hiring full-time position at Jina")](https://jobs.jina.ai)
<a href="https://twitter.com/intent/tweet?text=%F0%9F%91%8DCheck+out+Jina%3A+the+New+Open-Source+Solution+for+Neural+Information+Retrieval+%F0%9F%94%8D%40JinaAI_&url=https%3A%2F%2Fgithub.com%2Fjina-ai%2Fjina&hashtags=JinaSearch&original_referer=http%3A%2F%2Fgithub.com%2F&tw_p=tweetbutton" target="_blank">
  <img src="https://github.com/jina-ai/jina/blob/master/.github/badges/twitter-badge.svg"
       alt="tweet button" title="👍Share Jina with your friends on Twitter"></img>
</a>
[![Python 3.7 3.8](https://github.com/jina-ai/jina/blob/master/.github/badges/python-badge.svg "Jina supports Python 3.7 and above")](#)
[![Docker](https://github.com/jina-ai/jina/blob/master/.github/badges/docker-badge.svg "Jina is multi-arch ready, can run on differnt architectures")](https://hub.docker.com/r/jinaai/jina/tags)

</p>

In this demo, we use the Labeled Faces in the Wild (LFW) dataset from [the University of Massachusetts](http://vis-www.cs.umass.edu/lfw/#download) to build a search system so you can run a facial search on your own dataset. Make sure you've gone through [Jina 101](https://github.com/jina-ai/jina/tree/master/docs/chapters/101) and understood the [take-home-message](https://github.com/jina-ai/examples/tree/master/urbandict-search#wrap-up) in [our bert-based semantic search demo](https://github.com/jina-ai/examples/tree/master/urbandict-search) before moving on. 

  


**Table of Contents**

- [Overview](#overview)
- [Prerequirements](#prerequirements)
- [Prepare the data](#prepare-the-data)
- [Define the Flows](#define-the-flows)
- [Custom Encoder](#custom-encoder)
- [Run the Flows](#run-the-flows)
- [Next Steps](#next-steps)
- [Documentation](#documentation)
- [Community](#community)
- [License](#license)






## <a name="custom-encoder">Overview</a>

The overall design is similar to the semantic search demo. We consider each image as a Document and put the RGB array in the Chunk. Therefore, each Document has a single Chunk. The pretrained `facenet` model is used to encode the Chunks into vectors. 

In this demo, we will show how to define a custom Encoder to support a variety of models and use the pretrained model for indexing and searching.

<p align="center">
  <img src=".github/dataset.png" alt="Dataset Example" width="90%">
</p>


<details>
<summary>Click here to see the query outputs</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details> 

## Prerequirements

This demo requires Python 3.7.

```bash
pip install -r requirements.txt
```


## Prepare the data
In total, there are over 13,000 images of faces. 1,680 of the people pictured have two or more distinct photos in the [LFW Dataset](http://vis-www.cs.umass.edu/lfw/#explore). The following script will download the data and decompress it into `/tmp/jina/celeb/lfw`.

```bash
cd face-db-seach
bash ./get_data.sh
```

## Define the Flows

We start by defining the index and query Flows with the YAML files as follows: If you find it the YAML files a bit confusing, we highly recommend you go through our [bert-based semantic search demo](https://github.com/jina-ai/examples/tree/master/urbandict-search) before moving forwards.

<table style="margin-left:auto;margin-right:auto;">
<tr>
<td> </td>
<td> YAML</td>
<td> Dashboard </td>

</tr>
<tr>
<td> Index Flow </td>
<td>
  <sub>

```yaml
!Flow
metas:
  prefetch: 10
pods:
  loader:
    yaml_path: yaml/craft-load.yml
    replicas: $REPLICAS
    read_only: true
  flipper:
    yaml_path: yaml/craft-flip.yml
    replicas: $REPLICAS
    read_only: true
  normalizer:
    yaml_path: yaml/craft-normalize.yml
    replicas: $REPLICAS
    read_only: true
  encoder:
    image: jinaai/hub.executors.encoders.image.facenet
    replicas: $REPLICAS
    timeout_ready: 600000
    read_only: true
  chunk_indexer:
    yaml_path: yaml/index-chunk.yml
    replicas: $SHARDS
    separated_workspace: true
  doc_indexer:
    yaml_path: yaml/index-doc.yml
    needs: loader
  join_all:
    yaml_path: _merge
    needs: [doc_indexer, chunk_indexer]
    read_only: true
```

</sub>

</td>
<td>
<img align="right" height="420px" src=".github/index-flow.png"/>
</td>
</tr>
<tr>
<td> Query Flow </td>
<td>
  <sub>

```yaml
!Flow
with:
  read_only: true  # better add this in the query time
  rest_api: true
  port_grpc: $JINA_PORT
pods:
  loader:
    yaml_path: yaml/craft-load.yml
    read_only: true
    replicas: $REPLICAS
  flipper:
    yaml_path: yaml/craft-flip.yml
    replicas: $REPLICAS
    read_only: true
  normalizer:
    yaml_path: yaml/craft-normalize.yml
    read_only: true
    replicas: $REPLICAS
  encoder:
    image: jinaai/hub.executors.encoders.image.facenet
    timeout_ready: 600000
    replicas: $REPLICAS
    read_only: true
  chunk_indexer:
    yaml_path: yaml/index-chunk.yml
    separated_workspace: true
    polling: all
    replicas: $SHARDS
    reducing_yaml_path: _merge_topk_chunks
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

Let's look at the index Flow: Just like the [bert-based semantic search demo](https://github.com/jina-ai/examples/tree/master/urbandict-search), we define a two pathway Flow for indexing. For each image, we put the image filename in the request message so each image is considered a Document. The `loader` Pod reads the image file and saves the image's RGB values into the Chunk. Note that in this case we have only one Chunk per Document. 

Afterwards, the Flow splits into two parallel pathways. In the pathway on the left, the `normalizer` Pod resizes and normalizes the image in the Chunks so they can be properly handled in the downstream Pods. The `encoder` Pod encodes the Chunks into vectors, which are then saved into the index by the `chunk_indexer` Pod. 

In the other pathway, the `doc_indexer` Pod uses key-value storage to save the Document IDs and Document contents, i.e. the image filenames. At the end, the `join_all` Pod merges the results from the `chunk_indexer` and `doc_indexer`. In this case, the `join_all` Pods simply wait for both incoming messages to arriv because neither of the upstream Pods write into the request message.

The double-pathway Flow, as a common practice in Jina, stores the vectors and the Documents independently and in parallel. Of course, you can squeeze the two pathways into one by concatenating `doc_indexer` after `chunk_indexer` and removing the `join_all` Pod. However, this will slow down the indexing process. 

For the query Flow, it's pretty much the same as the index Flow, so we don't really need go into it too much. You may have noticeed there's something new in the YAML files. Let's dig into them!

### Hello, Docker!🐳
In the YAML file, we add the `encoder` Pod differently to the other Pods. Instead of using the YAML file to configure the Pods, we define the `encoder` with a Docker image using the `image` argument. By doing this, we have the `encoder` Pod running in its own Docker container. This is a key feature of Jina. By wrapping Pods into a Docker image, we can safely forget about complicated dependencies and environment settings required to run the Pods. 


```yaml
!Flow
pods:
  encoder:
    image: jinaai/hub.executors.encoders.image.facenet
```

This way we can pass the image tag of the Docker container containing the Encoder

## <a name="custom-encoder">Custom Encoder</a>

Jina has documentation on using available Executors and overriding them with a Custom Executor according to your needs. Here we build a custom Encoder using [FaceNet](https://arxiv.org/abs/1503.03832).

### Write Custom YAML File

```yaml
!FaceNetTorchEncoder
with:
  pretrained: 'vggface2'
  img_shape: 160

metas:
  name: face-encoder
  py_modules: FaceNetEncoder.py
  workspace: ./
requests:
  on:
    [SearchRequest, IndexRequest, TrainRequest]:
      - !EncodeDriver
        with:
          method: encode
```

The tags in `with` are passed to FaceNetEncoder.py class as parameters.
The 'py_modules' tag defines the Python Class containing your Encoder logic. The class should override an existing Executor.

### Dockerfile

```docker
FROM pytorch/pytorch:latest

ADD FaceNetEncoder.py encode.yml requirements.txt install.sh ./

RUN apt-get update && apt-get install --no-install-recommends -y libglib2.0-0 libsm6 libxext6 libxrender-dev git
RUN pip install -r requirements.txt
RUN bash install.sh

RUN python -c "from facenet_pytorch import InceptionResnetV1; model = InceptionResnetV1(pretrained='vggface2').eval()"

COPY . /

ENTRYPOINT ["jina", "pod", "--yaml-path", "encode.yml"]

```

Docker can be made on jina[devel] or any other image (pytorch in this case). We add dependancies like jina, opencv etc. The model weights are downloaded and cached in the 'RUN python -c ...' command. The entrypoint runs this container as a Jina Pod. You can override all the arguments of the Jina Pod during runtime (like specifying another YAML file).

We build this Docker image with the following command:


```bash
docker build -t jinaai/hub.executors.encoders.image.facenet .

```

Once built, we can pass this image tag to the Flow API.


## Run the Flows
### Index 
Now we start indexing with the following command:
 
```bash
python app.py index
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/index-demo.png?raw=true" alt="index flow console output">
</p>

</details> 

Here we define the flow in the YAML file and use it to index the data. The `index_files()` function loads the image filenames in the format of `bytes`, which will be further wrapped in an `IndexRequest` and sent to the Flow. 

```python
from jina.flow import Flow
f = Flow.load_config('flow-index.yml')
with f:
    f.index_files(image_src, batch_size=8, read_mode='rb', size=num_docs)
```

### Query

In Jina, Querying images located in `image_src` folder is as simple as the following command:

'print_result' is the callback function used. Here the top k (here k=5) is returned. In this example, we use the meta_info tag in response to create the image. 

```python
from jina.flow import Flow
f = Flow.load_config('flow-index.yml')
with f:
    f.search_files(image_src, sampling_rate=.01, batch_size=8, output_fn=print_result, top_k=5)
```

Run the following command to search images and display them in a web page.

```bash
python make_html.py
```

<details>
<summary>Click here to see result of the old API</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details> 

<img src=".github/face-demo.jpg" alt="query flow console output">
</p>
Note: We have used only 500 images with only one or two images per person. Hence the results are only as good as the database you have. It returns the closest-matching face in the dataset.

<br /><br />

Congratulations! Now you have an image search engine in your hands. We won't go into the details of the Pods' YAML files because they are quite self-explanatory. If you feel a bit lost when reading the YAML files, please check out the [bert-based semantic search demo](https://github.com/jina-ai/examples/tree/master/urbandict-search#dive-into-the-pods).

## Next Steps

- Write your own executors.
- Check out the Docker images at the Jina Hub.

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

## Community

- [Slack channel](https://join.slack.com/t/jina-ai/shared_invite/zt-dkl7x8p0-rVCv~3Fdc3~Dpwx7T7XG8w) - a communication platform for developers to discuss Jina
- [Community newsletter](mailto:newsletter+subscribe@jina.ai) - subscribe to the latest update, release and event news of Jina
- [LinkedIn](https://www.linkedin.com/company/jinaai/) - get to know Jina AI as a company and find job opportunities
- [![Twitter Follow](https://img.shields.io/twitter/follow/JinaAI_?label=Follow%20%40JinaAI_&style=social)](https://twitter.com/JinaAI_) - follow us and interact with us using hashtag `#JinaSearch`  
- [Company](https://jina.ai) - know more about our company, we are fully committed to open-source!



## License

Copyright (c) 2020 Jina AI Limited. All rights reserved.

Jina is licensed under the Apache License, Version 2.0. See [LICENSE](https://github.com/jina-ai/jina/blob/master/LICENSE) for the full license text.
