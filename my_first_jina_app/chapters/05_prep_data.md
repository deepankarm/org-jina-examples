## Preparing Data

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
