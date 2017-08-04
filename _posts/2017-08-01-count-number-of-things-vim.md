---
layout: post
title: "Count number of things in vim"
date: 2017-08-01
---

![counting](https://upload.wikimedia.org/wikipedia/en/2/29/Count_von_Count_kneeling.png)

There are times when I am parsing a file and need to know the number of times a `thing` occurs.

For example, lets say we have a file with peoples names.  I know, this is really silly...but I needed filler on such a small example.


```
Bob
Mary
Akira
Aiden
Leilani
Aiden
Brad
```

Lets see how many times __Aiden__ occurs. Type the following.

```bash
:%s/Aiden//gn
```


"Drum Roll"

```bash
2 matches on 2 lines
```


There you have it!  I have used more words to describe a simple 12 character command then needed--blogging at it's best =]
