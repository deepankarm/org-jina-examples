# My First Jina App: Running

⚠️ Now we're going to get our hands dirty, and if we're going to run into trouble, this is where we'll find it. If you hit any snags, check our **[troubleshooting](./troubleshooting.md)** section!

ℹ️ In this tutorial we're using Jina's RESTful API. In [other examples](https://github.com/jina-ai/examples) we currently use gRPC. So if the code seems a litte different, that's why!

## Prepare the Data

Our goal is to find out who said what in South Park episodes when a user queries a phrase. The [SouthPark dataset](https://github.com/BobAdamsEE/SouthParkData/) contains the characters and lines from seasons 1 to 19. Many thanks to [BobAdamsEE](https://github.com/BobAdamsEE) for sharing this awesome resource!👏

Now let's ensure we're back in our base folder and download and process this dataset by running:

```bash
cd ..
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

Now that `get_data.sh` has downloaded the data (and called `process_data.py` to process it), we've got `character-lines.csv`. We need to pass this file into `app.py`. `app.py` is pretty simple out of the box, so we'll have to make some changes:

### Check the Data

Let's just make sure the file has everything we want:

```shell
head data/character-lines.csv
```

You should see output like:

```csv
Stan! I don't wanna talk about it, I jus' wanna leave.
Mysterion! Mrs.
Sally! Pa-pa.
Canadians! We want respect!
Phillip! That smelly Saddam Hussein.
Cartman! Strike me down while you can!
Morpheus! What if I were to tell you.
Kanye! Yep, got it.
Jimbo! Here we are at Shafer's Crossing, lookin' for some animals.
Kyle! it's okay.
```

In the lines above, `!` acts as a separator between the character and what they say.

### Add `filepath`

In the `index` function, we currently have:

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

In the section above the `config` function, let's change:

```python
num_docs = os.environ.get('MAX_DOCS', 50000)
```

to:

```python
num_docs = os.environ.get('MAX_DOCS', 500)
```

That should speed up our testing by a factor of 100! Once we've verified everything works we can set it back to `50000` to index more of our dataset. If it still seems to slow, reduce that number down to 50 or so.

## Run the Flows

Now that we've got the code to load our data, we're going to dive into writing our app and running our Flows!

### Index Mode

First up we need to build up an index of our file. We'll search through this index when we use the query Flow later.

```bash
python app.py index
```

<details>
<summary>See console output</summary>

```console
index [====                ] 📃    256 ⏱️ 52.1s 🐎 4.9/s      4      batch        encoder@273512[I]:received "control" from gateway▸crafter▸encoder-head▸encoder-2▸⚐
        encoder@273512[I]:received "index" from gateway▸crafter▸⚐               
        encoder@273516[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-2▸⚐
        encoder@273525[I]:received "index" from gateway▸crafter▸encoder-head▸⚐    
      chunk_idx@273529[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-2▸encoder-tail▸⚐
      chunk_idx@273537[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-2▸encoder-tail▸chunk_idx-head▸⚐
      chunk_idx@273529[I]:received "control" from gateway▸crafter▸encoder-head▸encoder-2▸encoder-tail▸chunk_idx-head▸chunk_idx-1▸⚐
      chunk_idx@273533[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-2▸encoder-tail▸chunk_idx-head▸chunk_idx-1▸⚐
       join_all@273549[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-2▸encoder-tail▸chunk_idx-head▸chunk_idx-1▸chunk_idx-tail▸⚐
       join_all@273549[I]:collected 2/2 parts of IndexRequest                    
index [=====               ] 📃    320 ⏱️ 71.2s 🐎 4.5/s      5      batch        encoder@273512[I]:received "control" from gateway▸crafter▸encoder-head▸encoder-1▸⚐
        encoder@273512[I]:received "index" from gateway▸crafter▸⚐
        encoder@273516[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸⚐
        encoder@273520[I]:received "index" from gateway▸crafter▸encoder-head▸⚐    
      chunk_idx@273529[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸⚐                        
      chunk_idx@273541[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸⚐
      chunk_idx@273529[I]:received "control" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸chunk_idx-2▸⚐
      chunk_idx@273533[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸chunk_idx-2▸⚐                           
       join_all@273549[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸chunk_idx-2▸chunk_idx-tail▸⚐
       join_all@273549[I]:collected 2/2 parts of IndexRequest                       
index [======              ] 📃    384 ⏱️ 71.4s 🐎 5.4/s      6      batch        encoder@273512[I]:received "control" from gateway▸crafter▸encoder-head▸encoder-1▸⚐
        encoder@273516[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸⚐
      chunk_idx@273529[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸⚐
      chunk_idx@273537[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸⚐
      chunk_idx@273529[I]:received "control" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸chunk_idx-1▸⚐
      chunk_idx@273533[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸chunk_idx-1▸⚐
       join_all@273549[I]:received "index" from gateway▸crafter▸encoder-head▸encoder-1▸encoder-tail▸chunk_idx-head▸chunk_idx-1▸chunk_idx-tail▸⚐
       join_all@273549[I]:collected 2/2 parts of IndexRequest
```

</details>

### Search Mode

Run:

```bash
python app.py search
```

After a while you should see the console stop scrolling and display output like:

```console
Flow@85144[S]:flow is started at 0.0.0.0:65481, you can now use client to send request!
```

⚠️  Be sure to note down the port number. We'll need it for `curl` and jinabox! In our case we'll assume it's `65481`, and we use that in the below examples. If your port number is different, be sure to use that instead.

ℹ️  `python app.py search` doesn't pop up a search interface - for that you'll need to connect via `curl`, Jinabox, or another client.

### Searching Data

Now that the app is running in search mode, we can search from the web browser with Jinabox or the terminal with `curl`:

#### Jinabox

![](./images/jinabox-southpark.gif)
 
1. Go to [jinabox](https://jina.ai/jinabox.js) in your browser
2. Ensure you have the server endpoint set to `http://localhost:65481/api/search`
3. Type a phrase into the search bar and see which South Park lines come up

#### Curl

`curl` will spit out a *lot* of information in JSON format - not just the lines you're searching for, but all sorts of metadata about the search and the lines it returns. Look for the lines starting with `"matchDoc"` to find the matches.

```bash
curl --request POST -d '{"top_k": 10, "mode": "search", "data": ["text:hey, dude"]}' -H 'Content-Type: application/json' 'http://0.0.0.0:65481/api/search'
```

<details>
<summary>See console output</summary>

```json
{
  "search": {
    "docs": [
      {
        "chunks": [
          {
            "chunkId": 2859771895,
            "embedding": {},
            "weight": 1.0,
            "length": 1,
            "topkResults": [
              {
                "matchChunk": {
                  "docId": 454,
                  "chunkId": 797264081,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    8,
                    55
                  ]
                },
                "score": {
                  "value": 2.6883826,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 287,
                  "chunkId": 1142950297,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    8,
                    69
                  ]
                },
                "score": {
                  "value": 2.747776,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 418,
                  "chunkId": 3396351234,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    8,
                    33
                  ]
                },
                "score": {
                  "value": 2.7522104,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 158,
                  "chunkId": 1398208945,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    8,
                    25
                  ]
                },
                "score": {
                  "value": 2.9864397,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 345,
                  "chunkId": 3441934356,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    13,
                    40
                  ]
                },
                "score": {
                  "value": 2.994561,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 42,
                  "chunkId": 2326393068,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    5,
                    30
                  ]
                },
                "score": {
                  "value": 3.03432,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 374,
                  "chunkId": 3848825176,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    5,
                    35
                  ]
                },
                "score": {
                  "value": 3.080478,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 169,
                  "chunkId": 174461633,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    6,
                    17
                  ]
                },
                "score": {
                  "value": 3.0987353,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 70,
                  "chunkId": 614007298,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    8,
                    38
                  ]
                },
                "score": {
                  "value": 3.1020787,
                  "opName": "NumpyIndexer"
                }
              },
              {
                "matchChunk": {
                  "docId": 102,
                  "chunkId": 3182665395,
                  "offset": 1,
                  "weight": 1.0,
                  "length": 2,
                  "mimeType": "text/plain",
                  "location": [
                    8,
                    21
                  ]
                },
                "score": {
                  "value": 3.1413307,
                  "opName": "NumpyIndexer"
                }
              }
            ],
            "mimeType": "text/plain",
            "location": [
              0,
              14
            ]
          }
        ],
        "weight": 1.0,
        "length": 1,
        "topkResults": [
          {
            "matchDoc": {
              "docId": 454,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Cartman! Wendy, don't forget: I'll tell my mom on you.\n"
            },
            "score": {
              "value": 0.74899185,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 287,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Michael! Yeah, so listen: call up Firkle and meet me at Village Inn.\n"
            },
            "score": {
              "value": 0.74896955,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 418,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Cartman! Hey-where are you guys?\n"
            },
            "score": {
              "value": 0.74896795,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 158,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Cartman! Oh, shit, dude!\n"
            },
            "score": {
              "value": 0.7488801,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 345,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "CityWokOwner! I'm not stereotype, okay!\n"
            },
            "score": {
              "value": 0.74887705,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 42,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Stan! No, dude, I feel worse!\n"
            },
            "score": {
              "value": 0.74886215,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 374,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Kyle! Hey, uh, Jimmy, can we talk?\n"
            },
            "score": {
              "value": 0.7488448,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 169,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Randy! Hey yeah.\n"
            },
            "score": {
              "value": 0.74883795,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 70,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Michael! Hey, you wanna play with me?\n"
            },
            "score": {
              "value": 0.7488367,
              "opName": "BiMatchRanker"
            }
          },
          {
            "matchDoc": {
              "docId": 102,
              "weight": 1.0,
              "mimeType": "text/plain",
              "text": "Cartman! Well hello.\n"
            },
            "score": {
              "value": 0.748822,
              "opName": "BiMatchRanker"
            }
          }
        ],
        "mimeType": "text/plain",
        "text": "text:hey, dude"
      }
    ],
    "topK": 10
  },
  "status": {}
}
```
</details>



<table width="100%">
  <tr>
    <td align="left" style="text-align:right">
      <strong><a href="./setup.md">⬅️ Previous: Set up</a></strong>
    </td>
    <td align="right" style="text-align:right">
      <strong><a href="./understanding.md">Next: Understanding Jina ➡️</a></strong>
    </td>
  </tr>
</table>
