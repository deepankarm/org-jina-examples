# south_park_cc

[![Jina](https://github.com/jina-ai/jina/blob/master/.github/badges/jina-badge.svg?raw=true  "We fully commit to open-source")](https://get.jina.ai)

A demo neural search project that uses Jina. Jina is the cloud-native neural search framework powered by state-of-the-art AI and deep learning.

## Features

## Install

```bash
pip install .
```

To install it in editable mode

```bash
pip install -e .
```

## Run

| Command | Description |
| :--- | :--- |
|``python app.py index`` | To index files/data |
| ``python app.py query`` | To run query on the index | 
| ``python app.py dryrun`` | Sanity check on the topology | 

## Run as a Docker Container

To build the docker image
```bash
docker build -t jinaai/hub.app.south_park_cc:0.0.1 .
```

To mount local directory and run:
```bash
docker run -v "$(pwd)/j:/workspace" jinaai/hub.app.south_park_cc:0.0.1
``` 

To query
```bash
docker run -p 65481:65481 -e "JINA_PORT=65481" jinaai/hub.app.south_park_cc:0.0.1 search
```

## License

Copyright (c) 2020 Jina's friend. All rights reserved.


