## Run the Flows

Now we're going to dive into writing our app and running our Flows!


### Loading Data

Before we can index or query, we need to load our data into the flow. On line 22 of `flows/index.yml` we can see the indexing function currently loads an example set of strings:

```python
def index():
    f = Flow.load_config('flows/index.yml')

    with f:
        f.index_lines(['abc', 'cde', 'efg'], batch_size=64, read_mode='r', size=num_docs)
```

We want to change that `['abc', 'cde', 'efg']` to our South Park lines. Let's add a new function `read_data`:

```python

def read_data(f_path, max_sample_size=-1):
    if not os.path.exists(f_path):
        print('file not found: {}'.format(f_path))
    doc_list = []
    with open(f_path, 'r') as f:
        for l in f:
            doc_list.append(l.strip('\n').lower())
    if max_sample_size > 0:
        random.shuffle(doc_list)
        doc_list = doc_list[:max_sample_size]
    print('\n'.join(doc_list))
    for d in doc_list:
        yield d.encode('utf8')
```

And then we'll edit our `index` function to call our `read_data` function:

```python

def index():
    f = Flow.load_config('flows/index.yml')
    f_path = os.path.join('data', 'character-lines.csv')

    with f:
        f.index_lines(read_data(f_path), batch_size=64, read_mode='r', size=num_docs)
```

### Indexing

```bash
python app.py index -n 10000
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/index-demo.png?raw=true" alt="index flow console output">
</p>

</details>

With the Flows, we can now write the code to run the Flow. For indexing, we start by defining the Flow with a YAML file. Afterwards, the `build()` function will do the magic to construct Pods and connect them together. After that, the `IndexRequest` will be sent to the flow by calling the `index()` function.

### Query Flow

In `app.py` go to line 37 and edit the `dryrun` function:

```python
def dryrun():
    f = Flow.load_config('flows/query.yml')

    with f:
        while True:
            text = input('please type a sentence: ')
            if not text:
                break
            ppr = lambda x: print_topk(x, text)
            f.search(read_query_data(text), callback=ppr, topk=top_k)
```



    with f:
        while True:
            text = input('please type a sentence: ')
            if not text:
                break
            ppr = lambda x: print_topk(x, text)
            f.search(read_query_data(text), callback=ppr, topk=top_k)




------














```python
def main(num_docs):
    flow = Flow().load_config('flow-index.yml')
    with flow.build() as fl:
        data_fn = os.path.join('/tmp/jina/southpark', 'character-lines.csv')
        fl.index(buffer=read_data(data_fn, num_docs))

```

The content of the `IndexRequest` is fed from `read_data()`, which loads the processed JSON file and outputs each word together with its definition formatted into `bytes`.
Encoding the text with bert-family models takes a long time. To save your time, here we limit the number of indexed documents to 10,000.

```python
def read_data(f_path, max_sample_size=-1):
    if not os.path.exists(f_path):
        print('file not found: {}'.format(f_path))
    doc_list = []
    with open(f_path, 'r') as f:
        for l in f:
            doc_list.append(l.strip('\n').lower())
    if max_sample_size > 0:
        random.shuffle(doc_list)
        doc_list = doc_list[:max_sample_size]
    print('\n'.join(doc_list))
    for d in doc_list:
        yield d.encode('utf8')
```

### Querying

```bash
python app.py -t query
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details>

For querying, we follow the same process to define and build the Flow from the YAML file. The `search()` function is used to send a `SearchRequest` to the Flow. Here we accept the user's query input from the terminal and wrap it into request message formatted into `bytes`.

```python
def read_query_data(text):
    yield '{}'.format(text).encode('utf8')

def main(top_k):
    flow = Flow().load_config('flow-query.yml')
    with flow.build() as fl:
        while True:
            text = input('please type a sentence: ')
            if not text:
                break
            ppr = lambda x: print_topk(x, text)
            fl.search(read_query_data(text), callback=ppr, topk=top_k)
```

The `callback` argument is used to post-process the returned message. In this demo, we define a simple `print_topk` function to show the results. The returned message `resp` in a Protobuf message. `resp.search.docs` contains all the Documents for searching, and in our case there is only one Document. For each query Document, the matched Documents, `.match_doc`, together with the matching score, `.score`, are stored under the `.topk_results` as a repeated variable.

```python
ppr = lambda x: print_topk(x, text)
fl.search(read_query_data(text), callback=ppr, topk=top_k)
```

```python
def print_topk(resp, word):
    for d in resp.search.docs:
        print(f'Ta-Dah🔮, here are what we found for: {word}')
        for idx, kk in enumerate(d.topk_results):
            score = kk.score.value
            if score <= 0.0:
                continue
            doc = kk.match_doc.buffer.decode()
            name, line = doc.split('!', maxsplit=1)
            print('> {:>2d}({:f}). {} said: {}'.format(
                idx, score, name.upper(), line))
```
