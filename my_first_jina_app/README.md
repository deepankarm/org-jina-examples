# Notes

This is a different approach to tutorials. Instead of many tutorials to do and explain different things, I want:

## One simple tutorial to rule them all

I want a logical flow for new users. Right now it goes:
1. Jina 101: Cute pics, very layman explanations
2. Run `hello world` - easy enough for anyone who knows Docker, but not really practical for understanding how Jina *works*, just how to *run* it
3. Use Flow API to compose your workflow - wait, what? I don't even have a clear idea of the big picture of a Jina app yet
4. A few more tutorials without example data or use case - I'm a beginner - I want examples!
5. First practical example is [South Park search](https://github.com/jina-ai/examples/tree/master/southpark-search) which barely references the complex stuff above
6. And then more tutorials that start from almost scratch and introduce new features/search media each time, never referencing South Park tutorial

My idea:

1. Jina 101: What is Jina, and why use neural search?
2. Jina 102: Meet the Jina family (i.e. how Jina works)
3. Hello world - with Jinabox
4. Build your first Jina app - get started with a real example that performs a real task on existing data (i.e. this tutorial I'm writing now)
5. Further tutorials that expand on that first app (e.g. running in Docker, hosting online)
6. Other tutorials (e.g. image/video search) build on the above tutorial but just adapt it for other media, not introduce too many new concepts

## Broken down

The South Park tutorial is really long already. By the time I adapt it to be "My First Jina App" it'll be even longer. I want to break it into smaller chunks to make it more digestible. This:

* Allows for simpler linking - just link to that particular section
* Delineates each section of tasks clearly (with traditional markdown links it's easy to know where something *starts* but easy to lose track of where it ends)
* Is less intimidating: There's a *lot* of text and code and diagrams in some of those tutorials
* Breaks down theory sections and practice sections. Current tutorials mingle the both

## Uses our latest tech

* We've got cookiecutter and jinabox. Let's get users using them from day one. This means rewriting things like `get_data.sh` but it's not much work and gets them into the cookiecutter mindset and using best practices early on

## Is cute and could go viral

* "Everybody" knows South Park
* (Everybody also knows Pokemon, but image search takes more resources and we don't want to alienate low-hardware users too early on)

## Note: This is very much a work in progress

None of the below is final. I need to add (and correct) info and will restructure as needed. This is just for an idea of where I'm going with the idea for now.

---

# Building Your First Jina App

## Introduction

In this series of tutorials we'll guide you on the process of building your own Jina search app. Don't worry if you're new to machine learning or search. We'll spell it all out right here.

Our example program will be a simple neural search engine for text. It will take a user's typed input, and return a list of lines from South Park that match most closely. In our example case, it will search all the lines of South Park dialog

## Contents

* [Key concepts](chapters/01_concepts.md)
* [Try it out!](chapters/02_try_it.md)
* [Requirements and set up](chapters/03_requirements.md)
* [Files and folders](chapters/04_files.md)
* [Prepare data](chapters/05_prep_data.md)
* [Index data]
* [Monitor progress](chapters/08_dashboard.md)
* [Query data]
* [Run in your browser] - with jinabox
* # [Define Flows](chapters/06_define_flows.md)
* # [Write your app](chapters/write_app.md)
* # [Run Flows](chapters/07_run_flows.md)
* [Wrap up](chapters/10_wrap_up.md)

## To Add

* Get Support - put this in early

## Next Tutorial Set: Dig Deeper

* [Dive into Pods](chapters/09_pods.md)
* Run your encoding remotely
* Run Jina in Docker
* Scale up

