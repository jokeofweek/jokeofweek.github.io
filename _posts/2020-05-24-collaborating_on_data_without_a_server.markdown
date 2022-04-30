---
layout: post
title:  "Collaborating on Data Without a Server"
date:   2020-05-24 01:45:00 -0400
tags: typescript concepts
description: Using conflict-free replicated data types to collaborate on data in a peer-to-peer application.
---

Recently I was working on a password management application. I had the following goals in mind:

- Write the code once and be accessible on all devices. This led me to building it as a [progressive web app](https://web.dev/progressive-web-apps/) which could be installed on mobile devices as well as accessible online. 
- Avoid storing the passwords on a server, and eliminate the use of a server if at all possible. This minimizes the chances of potentially sensitive data being leaked.
- Allow offline password creation and support syncing in a way that no data would be lost. A key workflow is creating a password while on the go and wanting it to be visible on another device.

As a learning exercise, I wanted to try and make this application peer-to-peer to satisfy the server-less goal. [WebRTC](https://webrtc.org/) offers a number of constructs which can achieve this. A mechanism is still needed for finding peers, but that is out of scope for this post. For the remainder of this article, assume we have scaffolding for sending JS objects between peers. I'll also be ignoring the security aspect of a password manager entirely.

The focus of this post will be on the sync protocol aspect &mdash; specifically, how to collaborate on a data set while in a peer-to-peer context.

As a very simple example, a password manager's vault may contain a collection of entries:

{% highlight typescript %}
interface Entry {
  domain: string;
  username: string;
  password: string;
}
{% endhighlight %}  

We want to be able to synchronize vaults across devices such that whatever order clients come back online/message each other, every user eventually ends up with the same vault.

### Stop the conflicts!

A number of techniques can help us achieve this, and the one I'll be covering today is [conflict-free replicated data types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs). These are data structures which can exist on multiple clients and follow defined rules such that:

1. each client can manipulate its own version (ie. replica) without needing a central server or communication with any other client.
2. a client is able to synchronize with another client at any point and this will never result in an inconsistency (eg. client A has a different result than client B) or, worse, a merge conflict (the dreaded `>>>>>`).

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
  private entries: Map<string, Entry>;  
  
  constructor(entries = new Map<string, Entry>()) {
    this.entries = entries;
  }

  addEntry(entry: Entry) {
    if (this.entries.has(entry.uuid)) {
      // ... error
    }
    this.entries.set(entry.uuid, entry);
  }

  getEntries(): Map<string, Entry> {
    return this.entries;
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

![Image describing two replicas being updated locally and then their clients running merge to synchronize](/assets/images/crdts/merge.svg){:class="centered-image" width="300px"}

The tweaks we made to our model earlier (immutable entries and uniquely ID'd) are critical for implementing this. We know that if the maps both contain an entry with the same index[^1], they must have been previously synchronized (due to the uniqueness of the ID) and the content will not have changed (due to the immutability). The merge operation is actually just a merging of the two maps, with deduplication in place for the entry ID.

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
  // Make a copy of the local vault set of entries.
  const mergedMap = 
      new Map<string, Entry>(local.getEntries());

  for (let [id, entry] of remote.getEntries()) {
    // If the id is not already in our map, then this is a 
    // new entry that has not yet been synced.
    if (!mergedMap.has(id)) {
      mergedMap.set(id, entry);
    }
  }

  return new Vault(mergedMap);
}
{% endhighlight %}

The `Vault` type combined with the `merge` function is now a fully fledged CRDT. Clients can update their own local vault and periodically merge with other peers as they come online in order to acquire any new entries. As we never locally remove entries, this is an example of the [**Grow-only Set CRDT**](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#G-Set_(Grow-only_Set)).

The way we do our merge here is also important - there are two types of CRDTS: state-based CRDT and operation-based CRDT.

- **State-based CRDTs** work by receiving a replica's entire state in the `merge` function. This is the case for our above implementation, where we receive the client's entire vault.
- **Operation-based CRDTs** focus on the updates instead. You can represent a vault as a sequence of `AddEntry(entry)` update operations. In these CRDTs, clients send out the list of update operations instead of their entire state. The `merge` operation must now apply the operations locally. In our vault case, this is easy enough - we can just apply the operation directly due to the limitations we put in place! However this can get *much* more complicated [^2]. 

### One step further

This is nice and functional, however there's a user experience issue which can become obvious after using the application for a while. There's no way to remove entries! Over time your view becomes cluttered with stale passwords, expired accounts, leaked credentials... all things that you no longer want to see.

Ideally, we want to be able to clean up our view! Users should be able to delete/archive/remove an entry. 

A key constraint on our above implementation was that our set of entries only ever **grew**. Why is this critical? Consider what happens if a client removes an entry and then synchronizes with another replica. The other replica (`remote`) still has the entry. As we perform our `merge`, we encounter the entry... detect that it isn't in our own set of entries... and suddenly the pesky password has come back from the grave. Sp00ky.

Let's pretend for a minute that our application is *very* simple. There's no undo. Once you delete, the entry is gone.

How do we solve this? We need to remember what's been deleted! This is usually done through a second set containing **tombstones** &mdash; this is all the entry IDs which have been removed. With this, we have the power to propagate the deletion! No more being haunted.

{% highlight typescript %}
class Vault {
  // ...
  // The set of deleted entry UUIDs.
  private tombstones: Set<string>;
  
  constructor(
      entries = new Map<string, Entry>(),
      tombstones = new Set()) {
    this.entries = entries;
    this.tombstones = tombstones;
  }

  // ...

  isEntryActive(entry: Entry) {
    return !this.tombstones.has(entry.uuid);
  }

  removeEntry(entry: Entry) {
    this.tombstones.add(entry.uuid);
  }

  getTombstones(): Set<string> {
    return this.tombstones
  }
}
{% endhighlight %}

Now we can add and remove entries from our `Vault` and use the helper `isEntryActive` to determine whether we should actually present an item!

The merging logic only has a small change to make - we now need to account for the tombstones. Since we cannot remove from our tombstone set (our simplification above), and our entry IDs are unique, we can actually just merge the tombstone sets from `local` and `remote` to bundle in any deletions that either client has made!

{% highlight typescript %}

function merge(local: Vault, remote: Vault): Vault {
  // ... 

  // Merge the two set of unique IDs.
  const mergedTombstones = 
      new Set([
         ...local.getTombstones(),
         ...remote.getTombstones()
      ]);

  return new Vault(mergedMap, mergedTombstones);
}
{% endhighlight %}

Taking a step back, this is actually two grow-only sets working in tandem! This is called a [**Two-Phase Set**](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#2P-Set_(Two-Phase_Set)).

This gives us the power to reclaim our UI space! 

### Extras

#### Compacting

Our current implementation is quite wasteful - while deleted entries don't actually clutter our UI, they still clutter our app memory! This isn't really scalable and can easily be fixed. When we add an ID to the tombstone, we can actually just remove it from our entry set altogether. The tombstone is enough to be able to synchronize the deletion with all clients, and will generally be much smaller (in our case we just use the string `uuid` as the tombstone instead of all the entry data).

#### Oops I deleted a password a.k.a. undo

We added the "feature" that deletion was permanent. In practice, we often want to be able to undo operations. The first instinct may be to simply remove the ID from the tombstone set when we undo... but we have to keep in mind these are **grow-only sets**. Recall when implementing deletion that we couldn't simply delete from the entry set. The entry would get re-added upon the next merge with anothe replica that didn't have the deletion! We face a similar problem here. Deleting a tombstone and then syncing with anothe replica that has the tombstone would undo the undo. Ouch.

Instead, a simple way of solving this is to create a *new* version of the entry (eg. it has its own new ID). 

- If we didn't implement compacting, this is easy enough: look up the deleted item in the entry set and make a new entry with the same values but a new UUID.
- If we implemented compacting, this is trickier. We need to keep around enough data about the entries to reconstruct them. This starts to hit the limitation of a state-based CRDT. However, an operation-based CRDT can work nicely here! Imagine your two operations are `AddEntry(entry)` and `RemoveEntry(entry)` (these can be made commutative by updating the tombstone set internally even if the entry isn't there yet). We kept the entry data in our `RemoveEntry(entry)` operation and so could undo it by generating an `AddEntry` based on the data attached with the `RemoveEntry`.

### Footnotes

[^1]: UUIDs *can* collide though it is exceedingly rare. This could be prevented by prefixing the ID with a unique client name.
[^2]: For operation-based CRDTs to work, special constraints are placed on the operations. They **must** be commutative (eg. applying operation A followed by B is the same as B followed by A). We can make no guarantees on what order a client will receive its updates. This is the only constraint, so clients need to enforce deduplication of operations somehow since they may receive updates from various sources.
