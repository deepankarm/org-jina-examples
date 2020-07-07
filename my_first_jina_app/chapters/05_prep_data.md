## Preparing the Data

The goal of our project is to find out who said what in the South Park episodes when a user queries a sentence. The SouthPark data contains the characters and lines from seasons 1 to 19. Many thanks to [BobAdamsEE](https://github.com/BobAdamsEE) for sharing this awesome dataset!👏

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

Or, in a more human-readable format:

| Season | Episode | Character | Line                                    | 
| ---    | ---     | ---       | ---                                     | 
| 10     | 1       | Stan      | "You guys you guys! Chef is going away. | 
| 10     | 1       | Kyle      | "Going away? For how long?              | 
| 10     | 1       | Stan      | "Forever.                               | 
| 10     | 1       | Chef      | "I'm sorry boys.                        | 

Now let's ensure we're back in our repo's base folder (from the repo you cloned, *not* the `south_park` folder) and get this dataset by running:

```bash
bash ./get_data.sh
```

## Loading the Data

Now that we've got the data, we need to pass it into `app.py`. Let's make some changes to that file:

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
