## Run the Flows

Now we're going to dive into writing our app and running our Flows!

### Indexing

Indexing can take a long time, so let's start small by indexing just 1,000 lines so we can see our results:

```bash
python app.py index -n 1000
```

<p align="center">
  <img src=".github/index-demo.png?raw=true" alt="index flow console output">
</p>

</details>

### Querying

```bash
python app.py search
```

<details>
<summary>Click here to see the console output</summary>

<p align="center">
  <img src=".github/query-demo.png?raw=true" alt="query flow console output">
</p>

</details>

### Searching

Now that the app is running query mode, we can search from the command line:

```bash
curl --request POST -d '{"top_k": 10, "mode": "search",  "data": ["text:hey, dude"]}' -H 'Content-Type: application/json' 'http://0.0.0.0:45678/api/search'
```


