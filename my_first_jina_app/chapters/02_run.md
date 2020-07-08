# My First Jina App: Running

## Prepare the Data

Our goal is to find out who said what in South Park episodes when a user queries a phrase. The [SouthPark dataset](https://github.com/BobAdamsEE/SouthParkData/) contains the characters and lines from seasons 1 to 19. Many thanks to [BobAdamsEE](https://github.com/BobAdamsEE) for sharing this awesome resource!👏

The raw data contains season, episode, character, and line information in the `.csv` format as follows:

```csv
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
| 10     | 1       | Stan      | "You guys you guys! Chef is going away." | 
| 10     | 1       | Kyle      | "Going away? For how long?"              |
| 10     | 1       | Stan      | "Forever."                               |
| 10     | 1       | Chef      | "I'm sorry boys."                        |

Now let's ensure we're back in our base folder (from the repo you cloned earlier, *not* the `south_park` folder) and download this dataset by running:

```bash
bash ./get_data.sh
```

<details>
  <summary>See console output</summary>

```bash
Cloning into './south_park/data'...
remote: Enumerating objects: 3852, done.
remote: Total 3852 (delta 0), reused 0 (delta 0), pack-reused 3852
Receiving objects: 100% (3852/3852), 5.11 MiB | 2.37 MiB/s, done.
Resolving deltas: 100% (40/40), done.
```

</details>

## Load the Data

Now that we've got the data, we need to pass it into `app.py`. `app.py` is pretty simple out of the box, so we'll have to make some changes:

### Add `filepath`

On **line 25**, we've got:

```python
    with f:
        f.index_lines(['abc', 'cde', 'efg'], batch_size=64, read_mode='rb', size=num_docs)
```

As you can see, this indexes just 3 strings. Let's load up our South Park file instead with the `filepath` parameter:

```python
    with f:
        f.index_lines(filepath='data/character-lines.csv', batch_size=64, read_mode='r', size=num_docs)
```

Note we've also changed `read_mode` to `r`, since we're reading strings, not bytes.

### Index Fewer Documents

While we're here, let's reduce the number of documents we're indexing, just to speed things up while we're testing. We don't want to spend hours indexing only to have bugs later on!

On **line 17**, let's change:

```python
num_docs = os.environ.get('MAX_DOCS', 50000)
```

to:

```python
num_docs = os.environ.get('MAX_DOCS', 500)
```

That should speed up our testing by a factor of 100! Once we've verified everything works we can set it back to `50000` to index more of our dataset.

## Run the Flows

Now that we've loaded our data, we're going to dive into writing our app and running our Flows!

### Index Mode

First up we need to build up an index of our file, which is what we search:

```bash
python app.py index
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src="images/index-demo.png?raw=true" alt="index flow console output">
</p>

</details>

### Search Mode

Run:

```bash
python app.py search
```

Note: This doesn't pop up a search interface - for that you'll need to connect via `curl`, Jinabox, or another client.

### Searching Data

Now that the app is running in search mode, we can search from the web browser with Jinabox or the terminal with `curl`:

#### Jinabox

![](https://raw.githubusercontent.com/jina-ai/jinabox.js/master/.github/jinabox.gif)
 
1. Go to [jinabox](https://jina.ai/jinabox.js) in your browser
2. Ensure you have the server endpoint set to `http://localhost:65481/api/search`
3. Type a phrase into the search bar and see which South Park lines come up

#### Curl

```bash
curl --request POST -d '{"top_k": 10, "mode": "search", \
"data": ["text:hey, dude"]}' -H 'Content-Type: application/json' \
'http://0.0.0.0:65481/api/search'
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src="images/query-demo.png?raw=true" alt="query flow console output">
</p>

</details>



<table width="100%">
  <tr>
    <td align="left" style="text-align:right">
      <strong><a href="./01.md">⬅️ Previous: Installing</a></strong>
    </td>
    <td align="right" style="text-align:right">
      <strong><a href="./03_troubleshooting.md">Next: Troubleshooting ➡️</a></strong>
    </td>
  </tr>
</table>
