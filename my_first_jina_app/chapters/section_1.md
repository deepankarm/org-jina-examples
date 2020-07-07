# Building Your First Jina App

## Key Concepts

First of all, read up on [Jina 101]() and [102]() so you have a clear understanding of how Jina works. We're going to refer to those concepts a lot. We assume you already have some knowledge of Python and machine learning.

## Try it Out!

### With Docker

We've prepared a Docker image with indexed data. You can run it with:

```bash
docker run -p 45678:45678 jinaai/hub.app.distilbert-southpark
```

Check out more details about the Docker image [here](rest-api/README.md).

Note: You'll need to run the Docker image before trying the steps below

#### Querying with Jinabox

1. Go to [jinabox](https://jina.ai/jinabox.js) in your browser
2. Ensure you have the server endpoint set to `http://localhost:45678/api/search`
3. Type a phrase into the search bar and see which South Park lines come up

Find out more about [jinabox.js](https://github.com/jina-ai/jinabox.js/), including how to use the front-end code in your own project.

#### Querying with `curl`

Alternatively, you can open your shell and check the results via the RESTful API. The matched results are stored in `topkResults`.

```bash
curl --request POST -d '{"top_k": 10, "mode": "search",  "data": ["text:hey, dude"]}' -H 'Content-Type: application/json' 'http://0.0.0.0:45678/api/search'
```

## Install

### Prerequisites

You'll need:

* Basic knowledge of Python
* Python 3.7 or higher installed, and pip
* A Mac or Linux computer (we don't currently support Windows)
* A GPU (machine learning won't run quickly without it)
* 8 gigabytes or more of RAM
* Plenty of time - Indexing can take a while!

You should have also read [key concepts](01_concepts.md) to get a good overview of how Jina and this example work.

### Clone this repo

Let's get the basic files we need to get moving:

```
git clone blah/tutorial
cd tutorial
```

### Cookiecutter

We use [cookiecutter](https://github.com/cookiecutter/cookiecutter) to spin up a basic Jina app to save you having to do a lot of typing and setup. In your terminal, install cookiecutter with:

```
pip install -U cookiecutter && cookiecutter gh:jina-ai/cookiecutter-jina
```

And then generate a Jina project:

For our South Park example, we recommend the following settings:

* Project name: `South Park`
* Project slug: `south_park`
* Task type: `nlp`
* Index type: `strings`

All other fields you can just fill in however you please.

### Install Requirements

In your terminal:

```
cd south_park
pip install -r requirements.txt
```
## Files and Folders

After running `cookiecutter`, you should see a bunch of files in the `south_park` folder:

| File               | What it Does                                                             |
| ---                | ---                                                                      |
| `app.py`           | The main Python script where you initialize and pass data into your Flow |
| `Dockerfile`       | Lets you spin up a Docker instance running your app                      |
| `flows/`           | Folder to hold your Flows                                                |
| `pods/`            | Folder to hold your Pods                                                 |
| `README`           | The index to this tutorial                                               |
| `requirements.txt` | A list of requirements for `pip`                                         |

In the `flows/` folder we can see `index.yml` and `query.yml` - these define the indexing and querying Flows for your app.

In `pods/` we see `chunk.yml`, `craft.yml`, `doc.yml`, and `encode.yml` - these Pods are called from the Flows to process data for indexing or querying.
