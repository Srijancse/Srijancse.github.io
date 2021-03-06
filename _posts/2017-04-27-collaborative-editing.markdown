---
layout: post
title:  "How Real-Time Collaborative Editors work? [Operational Transformation]"
date:   2017-04-27 90:43:39 -0600
tags: technical javascript js code algorithm kde
---


Recently, I've been diving deep on how **real-time collaborative** editing works. I have been reading various papers on both the methods of implementing collaborative editing : **Operational Transformation**, which goes back to 1989, when it was first implemented by the [GROVE (GRoup Outtie Viewing Editor)](https://www.lri.fr/~mbl/ENS/CSCW/2012/papers/Ellis-SIGMOD89.pdf) system (this algorithm is quite old, and Google uses this algorithm for collaborative editing for Google Docs, Google Slides, Wave, etc) and **Conflict-Free Replicated Data Types**, which is a much newer approach to real-time editing. 

### Real challenge of collaborative editing

If you know how real-time collaborative editing works, then you may know that handling concurrent editing in multi user environment gracefully is very challenging. However, a few simple concepts can simplify this problem. The main challenge, as mentioned, with collaborative editing is the concurrency control [**concurrent edits**] to the document are not commutative. This needs to be causally ordered before applying either by undoing history, or by transforming the operations [**operational transformation**] before applying them to make them seem commutative.

### Bringing Latency into action

Introducing latency between the client and server is where the problems arise. Latency in a collaborative editor introduces the possibility of version conflicts. Here, where, I said Operational Transformation will come into action. 

Let's take an example :

**Starting Client's state :**

` ABCD `

**Starting Server's state :**

` ABCD `

Now, let's say, *Client* enters `X` in between `C` and `D` , the operation would look something like this :

`insert(X,3) //where 3 is the position where x is going to be added (0=A, 1=B, 2=C ..)`

And at the same time, *Server* deletes `b` , the operation would be :

`delete(B,1)`

What actually should happen is that the client and server should both end with ```ACXD``` but in reality, *client* ends with ```ACXD``` 

```
Starting Document State -> ABCD
"Insert 'X'" operation at offset 3 [local] -> ABCXD
"Delete 'B'" operation at offset 1 [remote] -> ACXD
```

but the *server* ends with ```ACDX```. 

```
Starting Document State -> ABCD
"Delete 'B'" operation at offset 1 [local] -> ACD
"Insert 'X'" operation at offset 3 [remote] -> ACDX
```

Ofcourse, ```ACXD != ACDX``` and the document which is shared now is in wrong state.

Here is where the **Operational Transformation** algorithm comes to the rescue. 

### Operational Transformation Algorithm

Operational Transformation (OT) is an algorithm/technique for the transformation of operations such that they can be applied to documents whose states have diverged, bringing them both back to the same state.

#### How does Operational Transformation work?

A short overview of how OT works :

* Every change (insertion or deletion) is represented as an **operation**. An operation can be applied to the current document which results into a new document state.


* To handle concurrent operations, we use the **tranform** function that takes two operations that have been applied to the same document state (but on different clients) and computes a new operation that can be applied after the second operation and that preserves the first operation’s intended change. 

Let's apply Operational Transformation in the original example.

If we apply OT, *Client* will see :

```
Starting Document State -> ABCD
"Insert 'X'" operation at offset 3 [local] -> ABCXD
"Delete 'B'" operation at offset 1 [transformed] -> ACXD
```

and *Server* will see :

```
Starting Document State -> ABCD
"Delete 'B'" operation at offset 1 [local] -> ACD
"Insert 'X'" operation at offset 2 [transformed] -> ACXD //Transform function would add add it in the new (3 - 1 = 2) position 
```

### Client - Server [OT] Approach to Collaborative Editing


The **transform** function is used to build a client-server protocol that can handle collaboration between any number of clients.

Choosing a Client-Server architecture will allow scouting a large number of clients without actually complicating the environment. Also, there will be a single system which holds the source of truth i.e. the server, so even if the clients crash/go offline for a long time, we can go back to the server and fetch the document easily. 

This source of truth also forces the client to wait for the server to **acknowledge** the operation that the client has just sent which would mean that the client always stays on the server's OT path. This would help in keeping a single history of operations without actually having to keep a mirror of the state for each client that is connected. That would eventually mean the number of clients that are connected to the server would have only one single copy of the document on the server. 

### Conclusion

**Operational Transformation** is a very powerful tool that allows to build great collaborative apps with support for non-blocking concurrent editing. I am looking forward to dive deep into the algorithm (and ofcourse extending this blogpost) and possibly working on an editor which supports it as well. 


### Resources 

I've read the following papers and articles to learn about Operational Transformation.

* [High Latency, Low-Bandwidth Windowing in the Jupiter Collaboration System](http://lively-kernel.org/repository/webwerkstatt/projects/Collaboration/paper/Jupiter.pdf) *Authored by David A. Nichols, Pavel Curtis, Michael Dixon, and John Lamping.*


* [Operational Transformation in Real-Time Group Editors: Issues, Algorithms, and Achievements](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.53.933&rep=rep1&type=pdf) *Authored by Chengzheng Sun and Clarence (Skip) Ellis.*


* [Google's whitepaper on Operational Transformation](http://www.waveprotocol.org/whitepapers/operational-transform)


* [Concurrency Control in Groupware Systems](https://www.lri.fr/~mbl/ENS/CSCW/2012/papers/Ellis-SIGMOD89.pdf) *Authored by C.A. Ellis, S.J. Gibbs* 


* [Evaluating CRDTs for Real-time Document Editing](https://hal.archives-ouvertes.fr/file/index/docid/629503/filename/doce63-ahmednacer.pdf) *Authored by Mehdi Ahmed-Nacer, Claudia-Lavinia Ignat, G´erald Oster, Hyun-Gul Roh, Pascal Urso*


* [Merging OT and CRDT Algorithms](https://hal.inria.fr/hal-00957167/document) *Authored by Mehdi Ahmed-Nacer, Pascal Urso, Valter Balegas, Nuno Pregui¸ca*


* [Wikipedia on Operational Transformation](https://en.wikipedia.org/wiki/Operational_transformation) *One of the most informative articles, I have found on wikipedia, suprisingly.*




