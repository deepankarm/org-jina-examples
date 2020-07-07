# My First Jina App: Running

## Prepare the Data

Our goal is to find out who said what in South Park episodes when a user queries a phrase. The [SouthPark dataset](https://github.com/BobAdamsEE/SouthParkData/) contains the characters and lines from seasons 1 to 19. Many thanks to [BobAdamsEE](https://github.com/BobAdamsEE) for sharing this awesome resource!👏

The raw data contains season, episode, character, and line information in the `.csv` format as follows:

```
Season,Episode,Character,Line
10,1,Stan,"You guys, you guys! Chef is going away.
"
10,1,Kyle,"Going away? For how long?
"
10,1,Stan,"Forever.
"
10,1,Chef,"I'm sorry boys.
```

If you're a human and want to read that more easily:

| Season | Episode | Character | Line                                    | 
| ---    | ---     | ---       | ---                                     | 
| 10     | 1       | Stan      | "You guys you guys! Chef is going away. | 
| 10     | 1       | Kyle      | "Going away? For how long?              | 
| 10     | 1       | Stan      | "Forever.                               | 
| 10     | 1       | Chef      | "I'm sorry boys.                        | 

Now let's ensure we're back in our base folder (from the repo you cloned, *not* the `south_park` folder) and download this dataset by running:

```bash
bash ./get_data.sh
```

## Load the Data

Now that we've got the data, we need to pass it into `app.py`. `app.py` is pretty simple out of the box, so we'll have to make some changes:

On line 25, we've got the line:

```python
    with f:
        f.index_lines(['abc', 'cde', 'efg'], batch_size=64, read_mode='rb', size=num_docs)
```

Let's change this to:

```python
    with f:
        f.index_lines(filepath='data/character-lines.csv', batch_size=8, read_mode='r', size=num_docs)
```

While we're here, let's also reduce the number of documents we're indexing, just to speed things up while we're testing.

On line 17, let's change:

```python
num_docs = os.environ.get('MAX_DOCS', 50000)
```

to:

```python
num_docs = os.environ.get('MAX_DOCS', 500)
```

That should speed up our testing by a factor of 100. Once we've verified everything works we can set it back to `50000`

## Run the Flows

Now we're going to dive into writing our app and running our Flows!

### Index Mode

Indexing can take a long time, so let's start small by indexing just 1,000 lines so we can see our results:

```bash
python app.py index -n 1000
```

<p align="center">
  <img src=".github/index-demo.png?raw=true" alt="index flow console output">
</p>

</details>

### Query Mode

```bash
python app.py search
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details>

### Search

Now that the app is running in query mode, we can search from the command line with `curl`:

```bash
curl --request POST -d '{"top_k": 10, "mode": "search",  "data": ["text:hey, dude"]}' -H 'Content-Type: application/json' 'http://0.0.0.0:45678/api/search'
```

Or from `jinabox.js`. You can refer back to that [section of our docs](./section_1.md) to get jinabox running your browser.

![](https://raw.githubusercontent.com/jina-ai/jinabox.js/master/.github/jinabox.gif)

