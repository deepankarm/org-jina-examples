# Building Your First Jina App

## Requirements

You'll need:

* Basic knowledge of Python
* Python 3.7 or higher installed, and pip
* A Mac or Linux computer (we don't currently support Windows)
* A computer with a GPU (machine learning won't run quickly without it)
* 8 gigabytes or more of RAM
* Plenty of time - training a model can take a while!

You should have also read [key concepts](01_concepts.md) to get a good overview of how Jina and this example work.

## Set Up

### Clone this repo

Let's get the basic files we need to get moving:

```
git clone blah/tutorial
cd tutorial
```

### Cookiecutter

We use [cookiecutter](https://github.com/cookiecutter/cookiecutter) to spin up a basic Jina app to save you having to do a lot of typing and setup. In your terminal, install cookiecutter with:

```
pip install -U cookiecutter
```

And then generate a Jina project:

```
cookiecutter gh:jina-ai/cookiecutter-jina
```

For our South Park example, we recommend the following settings:

* Project name: South Park
* Project slug: south_park
* Task type: nlp
* Index type: strings

All other fields you can just fill in however you please.

### Install Requirements

In your terminal:

```
cd south_park
pip install -r requirements.txt
```
