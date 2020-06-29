## Building Your First Jina App

### Running a search in the browser

Now we've queried our data using curl to access the REST API, let's look at a more visually attractive way of doing things.

[jinabox.js](https://github.com/jina-ai/jinabox.js/) is a lightweight, customizable omnibox. You can use it for searching text, images, videos, audio or any kind of data with a Jina backend.

All you need to do is make a note of the port that Jina is running on (`65481`, the same port you used for `curl`), and then head over to http://www.jinabox.js.

Once there, you'll see a box labelled *server endpoint*. Type in:
```
http://0.0.0.0:65481/api/search
```

And then type a line from South Park to see who said it. Boom! Instant neural search in your browser!
