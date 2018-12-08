---
title: "Visualizing my Github network"
layout: post
date: 2008-09-23 08:36
tag: 
- jekyll
- template
image: https://koppl.in/indigo/assets/images/jekyll-logo-light-solid.png
headerImage: true
projects: false
hidden: false
description: "Blog description"
category: template
author: johndoe
externalLink: false
---
A while ago, I read [this](https://towardsdatascience.com/visualising-my-facebook-network-clusters-346bac842a63) article by Ashris about visualising clusters in his Facebook friend network. I took a leaf out of that blog to understand my Github network better. 

## Getting the data
Unlike Facebook, Github provides an API to access a user's followers. I wrote a simple BFS-based function to get and queue a user's followers so their followers could be scraped down the line. Depending on the number of followers, fetching a user's followers can take a few seconds. Github's API returns followers with 30 in a page, which hence requires mutiple requests to obtain all followers of a user. I also set a limit of 200 on the number of followers before adding it queue. More followers, meant more branching and also potential deviation from my friend circle. 

```python
root='py-ranoid'
visited = set()
node_map = {}
queue = deque([root])

def spider():
    while True:
        seed = queue.popleft()
        if seed in visited:continue
        visited.add(seed)
        print seed,
        try:
            # Get all followers for a user
            nodes = getAllFoll(seed) 
        except TypeError:
            print "ERROR > ",seed
            continue
        node_map[seed] = nodes        
        if len(nodes) < 200:
            for n in nodes:
                if n not in visited:
                    queue.append(n)
        print '\t',len(visited),'\t',len(nodes),'\t',len(queue)
        if len(queue) == 0:
            break
```
After a few hours of uninterrupted querying, I had followers of 4670 users. Since we had to evenutally stop scraping at some point, there were many users in the queue whose followers we could not fetch. And since I had only 4670 nodes, I had to eliminate all followers whose followers I couldn't fetch (read that again). 

---

More **markdown** content

---
