---
layout: post
title:  "Sharing Files (You Could've Invented BitTorrent)"
date:   2020-08-30 12:00:00 -0400
tags: concepts 
description: How to share files with friends (and re-build BitTorrent from scratch in the process).
---
The decentralized nature of the Internet and the ability to self-host content has always been appealing to me. 

I'm really interested in taking problems and re-working them so that they can be solved without an obvious single point of failure instead of requiring a single central coordinating actor. This can be accomplished using [peer-to-peer](https://en.wikipedia.org/wiki/Peer-to-peer) networking &mdash; instead of connecting to a single server, one would connect with multiple other peers on their network. Coincidentally, I also enjoy collecting data. Thus, the **[BitTorrent protocol](https://en.wikipedia.org/wiki/BitTorrent)** is particularly interesting to me. For this post, I thought I would walk through a conceptual example to help de-mystify how the protocol works. 

Upfront warning: there will be a decent amount of hand-waving around the actual networking component of the protocol, since I believe it would just bog down this post (and, realistically, I'd make a fool of myself).

### Setting the Scene

We just finished running an event. Awesome. Maybe it was a conference, an unconference, or somewhere in the middle. It went well and we're feeling real proud of ourselves. This was also the first year we had talk transcripts as well as a professional photographer. We want to share this content with our organizers! There's one issue though... the organizers are located all over the world!

There's plenty of options here &mdash; you could share a BigCloudProvider Folder™... maybe run an FTP server... what about physically mailing USB sticks to everyone? All very plausible options. But you're hesitating. That sounds like a lot of work for you, whether it be paying money for the storage, making sure a device stays plugged in 24/7, or licking tons of envelopes. 

What if there was a way your organizers could help forward along the content to other organizers? Maybe some kind of organizer-to-organizer protocol... 🤔

### A First Attempt

Let's think of what we have here:

- A pre-defined list of files we want to share.
- A list of clients (ie. organizers, IP addresses... basically the same thing) we want to share these files with. This is called a **swarm**.

So we develop a simple application. The application will keep track of which files you've already downloaded and which ones you're missing. It'll connect you to the swarm and open up individual connections to each person. This might be the entire swarm, or it could be just a few people. 

These connections then support the following operations:

1. We can tell the entire swarm which files we have already downloaded. We could do this when we first connect to the swarm, when others connect to the swarm, or when our set of downloaded files changes.
2. We can send/receive a file for a given connection.

The process of sharing works by having a client *A* check which files it is missing, waiting for another client (*B*) to announce it has the file, and then communicating with client *B* to receive the file. Then client *A* would announce that it now has this file for anyone else who might need it. This lets content propagate through our swarm without always needing to get the files from the same person! That way we are not stuck content-less just because client *B* decided to go to bed early.

But there seems to be a problem here. Someone needs to provide the content initially, otherwise everyone will just be constantly announcing that they have no files. This is called the **initial seed**. 

How would this work in practice? Suppose we have four clients involved and we need to share `IMG_3051.jpg`. Initially there's only three clients in the swarm and none of them have the image. But in the shadows there lurks a mythical fourth client who has yet to connect and who has the image.

![Image showing a swarm of three clients missing the file, with a disconnected fourth client possessing the file](/assets/images/file-share/scene1.svg){:class="centered-image" width="300px"}

When the client connects, it tells everyone which files it has.

![The fourth client joins the swarm announcing it has the image file.](/assets/images/file-share/scene2.svg){:class="centered-image" width="300px"}

And now things start moving! One of our clients has a fast internet connection, hears this message, realizes "hey I need that!" and asks for the file. The send completes!

![A send occurs for the file and now a second client has the file.](/assets/images/file-share/scene3.svg){:class="centered-image" width="300px"}

Now that it has the file, this client decides to re-announce what it has.

![Two clients are now announcing they have the file.](/assets/images/file-share/scene4.svg){:class="centered-image" width="300px"}

Time continues flowing. Our remaining clients have now heard both announcements and can receive the file from either client. The **initial seed** could now leave without leaving clients to dry.

![Everyone now has the files, with sends being issued from each client.](/assets/images/file-share/scene5.svg){:class="centered-image" width="300px"}

With this application, we can now share our content peer-to-peer without needing to remain connected the entire time! Others can take up part of the sharing work.

### Big Files Slow Connections

The application is working well! Our organizers are slowly getting all of the generated media. Key word being *slowly*. You've gotten a few reports now that users will occasionally get stuck waiting ages for some of the larger files (RAW images are quite large after all). Not everyone in our swarm has fast internet connections, so the experience can be sub-optimal if you end up requesting a small file from a fast client and a large file from a slow client.

Your first guess here might be to always prioritize selecting the fastest client in the swarm. That sounds reasonable... until they get bombarded with requests. Their home connection slows to a crawl. Eventually, they figure out that Netflix is streaming at awful quality due to the application and decide to close it &mdash; right as someone had downloaded exactly 99% of the image, leaving them unhappy.

A light bulb goes off in your head: **why not split up our files into smaller pieces**? This kills two birds with one stone! It minimizes the impact of slow connections... downloading 100Kb from a slow connection is less painful than trying to download 100Mb. It also helps avoid re-downloading! If you are on a slow internet connection and already have 99% of a file downloaded and the connection drops all you need to do is download the missing piece instead of re-downloading the entire file.

So it's time to tweak our application &mdash; version 2.0.

Your application now conceptually splits up each file into **pieces**. Each piece has the same size. The protocol now uses the following operations:

1. We can tell the entire swarm which __pieces__ we have already downloaded. 
2. We can send/receive a __piece__ for a given connection.

Pickign the piece size actually has an important trade off. As piece size decreases, there are more announces and more file send/receive connections. If your piece size is too low (or too high!), the protocol loses efficiency.

### Growing the Swarm

Initially, we embedded a list of possible clients in our application. This isn't the most practical: IP addresses change, organizing teams grow, people want to download the file from multiple devices, etc.

We could develop a centralized **tracker** of peers. This service is responsible for tracking the IP addresses of swarm members, and for providing that list to clients. Our application would register with the tracker on startup and query for available peers. The tracker could periodically send out updates as the swarm changes to keep clients up to date.

This seems less than ideal given our initial goal of offloading as much work as possible and not needing to host anything. Feeling inspired from the peer-to-peer nature of the protocol up to this point, you might get curious and wonder if there's a way to exchange swarm information peer-to-peer as well. In the real world, [distributed hash tables](https://en.wikipedia.org/wiki/Distributed_hash_table) are generally used to accomplish this (eg. the BitTorrent protocol uses the [Kademlia](https://en.wikipedia.org/wiki/Kademlia) system). These approaches all generally require some clients which are always online to bootstrap the search for swarm members, but it provides a way to de-centralize and distribute this work.

### Do You Trust Me?

Up till this point, we've trusted all the members of our swarm to send the same files as the **initial seed**. But what if a malicious organizer decides to modify one of the images to make a meme and spreads it masquerading as the original? Alternatively, what if someone's connection is flaky and we're not quite sure we got the right bytes for the image?

We've discussed a similar problem in the past when [Verifying Data with Merkle Trees]({% post_url 2020-05-26-merkle_trees %}). The core of solving that problem involved using a [checksum function](https://en.wikipedia.org/wiki/Checksum) on our data and sharing the checksum so that others could verify they had the data we intended. 

So we decide to try **embedding a checksum with each file piece** in our application alongside our file piece information. Our application can then verify that any piece received from another client matches the checksum! We can now add untrusted members to the swarm without worrying that they might tamper our file data!

### What About Next Year?

We've now established a good conceptual protocol for sharing this type of content. But one part of it is still impractical: so much information is baked directly into the application! Organizers really like the application and would like to use it for other events.

The information we needed to coordinate this dance of file-sharing includes:

- The list of file pieces and their checksums.
- Any information we need to find other swarm members. As an example, this could be the URL for our tracker!

If we were to put this information in a file and modify our application to accept these files, we could now re-use the protocol arbitrarily! This means our application could download and share multiple sources at once.

It turns out that this is exactly what a `.torrent` file contains (along with other data). These files are encoded using the [Bencode](https://en.wikipedia.org/wiki/Bencode) format in order to allow flexibility and offer some size reduction. Everything you've read in this post is actually used in the BitTorrent protocol: the process itself, file pieces, checksumming, and trackers (or trackerless via distributed hash tables). Our application is the equivalent to a BitTorrent client! 

**Well done - we over-engineered a simple file-sharing problem and built BitTorrent from scratch.** 🎉
{: style="text-align: center;"}

### Aside: Encouraging Good Behavior

We didn't really cover a) how to select which clients we interact with and b) how to select which file pieces to get first, but these are both important decisions!

Some BitTorrent clients actually support preferential client selection to encourage good file-sharing behavior. As an example, they may prefer to send data to clients who have a track record of interacting with the swarm (eg. send files to a client that you know you need files from). This type of selection can help encourage users to stay connected to the swarm even once they've accumulated all files (ie. *seeding*). But implementing this approach as-is means that clients brand new to the swarm end up in a chicken and egg situation: they have no data to send back!

Prioritizing pieces also has trade-offs. While we could select pieces at random, this doesn't always result in the healthiest swarm. What if instead we tracked which file pieces had the least *availability* (ie. fewest peers announcing they have the piece)? These pieces are the most at risk of going "extinct": if only a few users have a piece, this is less resilient to clients going offline. By prioritizing these files, we can improve the experience for all clients.