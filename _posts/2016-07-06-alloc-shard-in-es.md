---
layout: post
title:  "Manually allocate orphaned ES shards to a node"
date:   2016-07-06 15:28:48 -0500
categories: elasticsearch
---

## ...or: why are no shards being allocated to a node?

Do a

	GET _cat/shards

and you'll see something like

```
logstash-2015.01.08-raw 4 p STARTED     10238    1.1mb 172.17.0.2 es-01
logstash-2015.01.08-raw 4 r UNASSIGNED                                  
test                    4 p UNASSIGNED                                  
test                    4 r UNASSIGNED                                  
test                    0 p UNASSIGNED                                  
test                    0 r UNASSIGNED                                  
test                    3 p UNASSIGNED                                  
test                    3 r UNASSIGNED                                  
test                    1 p UNASSIGNED                                  
test                    1 r UNASSIGNED                                  
test                    2 p UNASSIGNED                                  
test                    2 r UNASSIGNED                                  

```

but that's annoying because we added a second node...but nothing is allocated to it:

Doing

	GET _cat/allocation

I'm getting

	516 13.6gb 1gb 14.7gb 92 58fc33034c60 172.17.0.2  es-01      
	  0 13.6gb 1gb 14.7gb 92 ff50dd0b4d0e 172.17.0.26 es-00      
	594                                               UNASSIGNED

What's up with `es-00`? Let's try manually allocating a shard to it:

	POST _cluster/reroute
	{
	  "commands": [
	    {
	      "allocate": {
	        "index": "test",
	        "shard": 0,
	        "node": "es-00",
	        "allow_primary": true
	      }
	    }
	  ]
	}

which responds with (newlines inserted for clarity)

	{
	  "error": "RemoteTransportException[[es-01][inet[/172.17.0.2:9300]][cluster:admin/reroute]];
	    nested: ElasticsearchIllegalArgumentException[
	      [allocate] allocation of [test][0] on node [es-00][lE3e-Po_QQiXMk_n10jO9Q][ff50dd0b4d0e][inet[/192.168.0.100:9300]] is not allowed,
			YES(shard is not allocated to same node or host)]
			[YES(shard is primary)][YES(below primary recovery limit of [4])]
			[YES(no allocation awareness enabled)][YES(total shard limit disabled: [-1] <= 0)]
			[NO(less than required [15.0%] free disk on node, free: [7.08796015987245%])]
			[YES(no snapshots are currently running)]
		]; ",
	  "status": 400
	}

Looking for the `NO` items, we find the cause:

	NO(less than required [15.0%] free disk on node, free: [7.08796015987245%])

> Written with [StackEdit](https://stackedit.io/).
