---
layout: post
title:  "Collaborating on Data Without a Server"
date:   2020-05-23 19:53:38 +0000
tags: javascript concepts
---

Recently I was working on a password management application. I had the following goals in mind:

- Write the code once and be accessible on all devices. This led me to building it as a [progressive web app](https://web.dev/progressive-web-apps/) which could be installed on mobile devices as well as accessible online. 
- Avoid storing the passwords on a server, and eliminate the use of a server if at all possible. This minimizes the chances of potentially sensitive data being leaked.
- Allow offline password creation and support syncing in a way that no data would be lost. A key workflow is creating a password while on the go and wanting it to be visible on another device.

As a learning exercise, I wanted to try and make this application peer-to-peer to satisfy the server-less goal. [WebRTC](https://webrtc.org/) offers a number of constructs which can achieve this. A mechanism is still needed for finding peers, but that is out of scope for this post. 

This post will be focusing on the sync protocol aspect &mdash; specifically, how to collaborate on a data set while in a peer-to-peer context.

As a very simple example, a password manager's vault may contain a collection of entries:

{% highlight typescript %}
interface Entry {
  domain: string;
  username: string;
  password: string;
}
{% endhighlight %}  

We want to be able to synchronize vaults across devices such that whatever order clients come back online/message each other, every user eventually ends up with the same vault.

A number of techniques can help us achieve this, and the one I'll be covering today is [conflict-free replicated data types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs). These are data structures which can exist on multiple clients and follow defined rules such that:

1. each client can manipulate its own version (ie. replica) without needing a central server or communication with any other client.
2. a client is able to synchronize with another client at any point and this will never result in an inconsistenc (eg. client A has a different result than client B) or, worse, a merge conflict (the dreaded `>>>>>`).

To begin, we'll make a small tweak to our model which will greatly simplify this: make entries immutable and assign them an index based on a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier).

{% highlight typescript %}
interface Entry {
  readonly uuid: string;
  readonly domain: string;
  readonly username: string;
  readonly password: string;
}
{% endhighlight %}  

A sample implementation of a vault may look something like:

{% highlight typescript %}
class Vault {
  // A mapping of UUID its corresponding entry.
  entries: Map<string, Entry>
  
  constructor() {
    this.entries = new Map<string, Entry>();
  }

  addEntry(entry: Entry) {
    if (this.entries.has(entry.uuid)) {
      // ... error
    }
    this.entries.set(entry.uuid, entry);
  }
}
{% endhighlight %}

This implementation already satisfies our first rule! Adding an entry requires no communication to any other client. If we can then solve the following function in accordance with rule 2, we'll have a CRDT and can sync our data across clients:

{% highlight typescript %}
/**
 * Implementations should satisfy:
 *   merge(local, remote) == merge(remote, local)
 *
 * @param local The vault stored locally.
 * @param remote The vault stored on the peer we are connected to.
 * @returns A new vault which has merged the local and remote vault.
 */ 
function merge(local: Vault, remote: Vault): Vault {
  // ???
}
{% endhighlight %}

This would be used in the following scenario:

![Image describing two replicas being updated locally and then their clients running merge to synchronize](/assets/images/crdts/merge.png){:class="centered-image"}