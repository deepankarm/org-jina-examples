## Tweaking Your App

By default the cookiecutter template only provides a very rudimentary template. We need to make some adjustments:

## Reducing the number of documents

Let's make sure our search works before going all in. On line 8, we see `num_docs = os.environ.get('MAX_DOCS', 5000)` . Let's reduce that for now:

```python
num_docs = os.environ.get('MAX_DOCS', 50)
```

### Defining our data file

Now we need to load our dataset. Let's create a path to our data file below `num_docs`:

```python
data_path = os.path.join('data','character-lines.csv')
```

Note: this adds a new line, and this is reflected in the line numbers below.

### Reducing Shards

Since we're only indexing 50 lines for now, let's reduce the amount of shards on line 13:

`shards = 2`

### Load our data into the flow

On line 13 we're loading a simple list of strings by default. Let's load up our data file instead:

```python
f.index_lines(filepath=data_path, batch_size=8, read_mode='r', size=num_docs)
```
