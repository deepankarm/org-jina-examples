# Building Your First Jina App

## 👋 Introduction

In this series of tutorials we'll guide you on the process of building your own Jina search app. Don't worry if you're new to machine learning or search. We'll spell it all out right here.

Our example program will be a simple neural search engine for text. It will take a user's typed input, and return a list of lines from South Park that match most closely.

## 📖 Contents

* [Set up](chapters/01_setup.md)
* [Run Jina](chapters/02_run.md)
* [Troubleshooting](chapters/03_troubleshooting.md)
* [Understanding Jina](chapters/04_understanding.md)
* [Wrap up](05_wrap_up.md) - not done yet

Need support? First check out our [troubleshooting section](chapters/03_troubleshooting.md). If that doesn't work, hit up our #support channel on our [Slack](https://jina-ai.slack.com/messages/support/)!

## ⛏️ Next Up: Dig Deeper

**Note: Not done yet, but including here as a TODO for now**

If you've gone through the above steps you've already got one project under your belt. Nice! Stay tuned to expand South Park search even further:

* Run your encoding remotely
* Run Jina in Docker
* Scale up
* Search different things (images, videos, you name it)

---

# Notes

This is a different approach to our existing tutorials. Instead of many tutorials to do and explain different things, I want:

## One simple tutorial to rule them all

I want a logical flow for new users. Right now it goes:
1. Jina 101: Cute pics, very layman explanations
2. Run `hello world` - easy enough for anyone who knows Docker, but not really practical for understanding how Jina *works*, just how to *run* it
3. Use Flow API to compose your workflow - wait, what? I don't even have a clear idea of the big picture of a Jina app yet
4. A few more tutorials without example data or use case - I'm a beginner - I want examples!
5. First practical example is [South Park search](https://github.com/jina-ai/examples/tree/master/southpark-search) which barely references the complex stuff above
6. And then more tutorials that start from almost scratch and introduce new features/search media each time, never referencing South Park tutorial

And then we mention cookiecutter on our README as well! And that doesn't do things the same way as our existing examples!

Based on feedback from team members like Bing and Lucia (feel free to chip in if I'm wrong - I don't want to misrepresent you!), we need a simpler on boarding process.

My idea:

1. Jina 102: Meet the Jina family (i.e. how Jina works) - if ppl are unclear on what neural search is, we have a blog post about it
2. Hello world - with Jinabox
3. Build your first Jina app - get started with a real example that performs a real task on existing data (i.e. this tutorial I'm writing now, based on South Park)
4. Further tutorials that expand on that first app (e.g. running in Docker, hosting online, all based on "My First Jina App"). Let's not confuse things by adding new use cases (e.g. image search) AND new features (e.g. Docker) at the same time.
5. Other tutorials (e.g. image/video search) build on the above tutorial but just adapt it for other media, not introduce too many new concepts

## Broken down

The South Park tutorial is really long already. By the time I adapt it to be "My First Jina App" it'll be even longer. I want to break it into smaller chunks to make it more digestible. This:

* Allows for simpler linking - just link to that particular section
* Is less intimidating: There's a *lot* of text and code and diagrams in some of those tutorials
* Breaks down theory sections and practical sections. Current tutorials mingle the both

## Uses our latest tech

* We've got cookiecutter and jinabox. Let's get users using them from day one. This means rewriting things like `get_data.sh` but it's not much work and gets them into the cookiecutter mindset and using best practices early on

## Is cute and could go viral

* "Everybody" knows South Park
* (Everybody also knows Pokemon, but image search takes more resources and we don't want to alienate low-hardware users too early on)

## Note: This is very much a work in progress

None of the below is final. I need to add (and correct) info and will restructure as needed. This is just for an idea of where I'm going with the idea for now.
