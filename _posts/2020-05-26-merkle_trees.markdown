---
layout: post
title:  "Verifying Data with Merkle Trees"
date:   2020-05-26 22:00:00 -0400
tags: typescript concepts
description: Building up to using merkle trees to verify a list of data blocks.
---

Let's walk through a thought experiment together. As part of your latest sprint, you've been tasked with adding a feature to your product where users can download a set of files (or just data in general). This sounds easy enough. You set up some kind of hosted directory and instantly forget about it, since that implementation isn't relevant to this post.

### Did I get the right data?

After a while, a few bug reports start to come in - your application is misbehaving with the downloaded data. You spend a few hours/days/weeks debugging and end up figuring out that the files the users have don't match the files you are hosting. Uh oh.

This can happen for plenty of reasons. The internet is a wild place. Maybe the user has a flaky connection, causing some content to be dropped. It could be that the files got overwritten &mdash; we've all accidentally done that when saving a file from the browser. Possibly the user's machine is being tampered with in some way. 

Luckily, this is a problem we can fix! We can check that the user has the right data as part of our application. A simple way of accomplishing this is using a [checksumming function](https://en.wikipedia.org/wiki/Checksum). This takes an arbitrarily-sized piece of data and distills it into a small string. [Hash functions](https://en.wikipedia.org/wiki/Hash_function) are also related, providing some different properties for cryptographic purposes. Generally, these functions are designed such that similar input data will produce different checksums [^1]. A key aspect is that they should be deterministic: checksumming the same input will always produce the same checksum.

{% highlight typescript %}
function checksum(input: string): string {
  // Some implementation would go here. You 
  // could use CRC: https://en.wikipedia.org/wiki/Cyclic_redundancy_check.
  // 
  // This could also be a hashing function,
  // such as MD5 or the SHA family. 
}
{% endhighlight %}

Why does this fix our issue? It would be impractical to send over all the bytes the client has downloaded to verify (ie. a second download). Instead, we can now take a large set of data, obtain the checksum (generally < 1 kilobyte), and verify it against our expected checksum. Equipped with a `checksum` function of your choosing, this might look something like:


{% highlight typescript %}
function verifyFiles(
    fileContents: Map<string, string>, 
    expectedChecksum: string): boolean {
  let concatenatedFileContents = '';

  // We sort the file names in alphabetical order and then
  // concatenate their contents.
  const sortedFileNames = Array.from(fileContents.keys()).sort();
  for (let fileName of sortedFileNames) { 
    concatenatedFileContents += fileContents.get(fileName);
  }
  
  return checksum(concatenatedFileContents) == expectedChecksum;
}
{% endhighlight %}

You've solved this new problem statement! Some key aspects to note here:

- Checksumming functions *can* have collisions (ie. two different inputs produce the same checksum). This becomes a game of probabilities &mdash; you need to decide on a tolerance for collisions and adjust accordingly. This becomes more problematic as input size increases or output (checksum) size decreases [^2].
- The checksumming approach for verifying the local file contents must be the same one used to calculate the expected checksum. Things to be aware of here include using checksumming algorithms which are configured differently (maybe the output size is bigger on the local client?) or accumulating a string to checksum in a non-deterministic order (did you append all file contents based on a sorted set of file names?). Backwards compatibility can become *very* tricky here.

### Which file is wrong? 
 
The current of time keeps flowing and, by pure coincidence, you get two seemingly unrelated customer reports in the same day:

1. "The application warns me that my downloaded files don't match. How do I know what to download to fix this? My connection is limited... I don't want to redownload everything."
2. "My friend told me he had most of the files on a floppy disk / CD / USB / SomeCorpCloudService&trade; and gave them to me. Can I check that these are fine?"

We fixed the overall problem of verification with our previous patch, but the user experience isn't great here. We need more granularity! Right now we just provide a yes/no answer to the question "are all my files valid".

Let's take a look at the function we conjured together. Previously, our method was to concatenate all the files together into one giant string, produce a checksum, and then compare it. What if we were to instead keep a per-file checksum? This would let us answer the question on a per-file basis and also help us identify which files are missing. 

Taking this one step further - we can even keep concatenate all these file-specific checksums and produce a new checksum!  This lets us answer the overall question of "are all my files valid" quickly, and if the answer is no, we can then dig into the various checksums to find the root cause. Because of the properties of these functions, if one file is wrong, then its file-specific checksum will mismatch and consequently so will the concatenated checksum derived from it.

Turns out this concept already exists: [hash lists](https://en.wikipedia.org/wiki/Hash_list). You might implement something like this:

{% highlight typescript %}
class ChecksumList {
  /* A mapping of file name to checksum. */
  private fileChecksums: Map<string, string>;
  
  /* The checksum for all files. */
  private concatenatedChecksum: string;
  
  constructor(fileContents: Map<string, string>) {
   // First we sort our filenames.
    const sortedFileNames =
      Array.from(fileContents.keys()).sort();
    
    // Then we compute a file-specific checksum for each
    // file and aggregate all computed checksums in order.
    const allChecksums = '';
    for (let fileName of sortedFileNames) {
      const fileChecksum = checksum(fileContents.get(fileName));
      allChecksums += fileChecksum;
      this.fileChecksums.set(fileName, fileChecksum);
    }
       
    // Finally, we compute the overall checksum.
    this.concatenatedChecksum = checksum(allChecksums);
  }
}
{% endhighlight %}

This class is very powerful and easy to serialize! We can provide the expected checksum list to our application via any mechanism we want and then compare the computed concatenated checksum and file-specific checksum to figure out what is wrong. 

### Building blocks

At some point, you start to realize that our data structure can be more abstract than just for files. As is, if a user has even one byte off in a multi-gigabyte file, it's detected as invalid and they must redownload the entire file.

A better user experience is to segment our data into roughly equal-sized data blocks. This requires minimal modification to our above structure: chiefly, the indexing scheme for the fine-grained mapping of checksums. While we were previously using the filename, now you'll need to consider a different approach.

We do need to be careful here as the size of the data block and the size of the output checksums can affect storage size and performance characteristics. If you use blocks that are too big or a checksum size that is too small, you will increase the likelihood of collisions. Conversely, if you use blocks that are too small or a checksum size that is too big, you begin to lose the benefits of hashing as the metadata for your structure grows in size relative to data block size.

### Venturing into the (Merkle) trees

You've done it. Your company has hit it big and now your application has to scale. Other divisions have heard about your nifty verification code and they begin hosting data blocks there as well. The `ChecksumList` object grows with each file that is part of the system and becomes too expensive to send to your application. Clients also don't necessarily want to bundle all the unrelated files.

You begin to wonder: how can I determine that the data checksum `x` is actually in the overall data set without needing to get the whole `ChecksumList`. 

Enter the [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree), a generalization of the aforementioned hash list.

This tree works as follows:
- Each leaf node represents one data block, and stores its checksum/hash.
- Each parent node stores a checksum computed based on the checksums of its direct children.

As an example, here is a merkle tree for 4 data blocks (the leaf nodes). Observe that all the parent nodes are using a checksum based on its two children.

![Image describing a plain Merkle tree with 4 leaf nodes](/assets/images/merkle/plain.svg){:class="centered-image" width="300px"}

Why is this beneficial? On the surface, this looks like more data stored to represent the same thing. The win is subtle - consider what happens if you want to verify that a data block is correct. Previously, if you wanted to know that your checksummed block was actually correct and included in your top-level checksum, you had to concatenate all the checksums. Now you only need to compute the checksums along the path to the root [^3]!

So say you wanted to verify content for the third leaf (which should have checksum `C3`), you would need to know your computed checksum as well as the checksums for `C4` (necessary input to compute `C6`) and also `C5` (necessary input to compute `C7`). By storing the calculated values along the paths, you've effectively pruned `C1` and `C2` from your calculation.

![Image describing a Merkle tree with the nodes pruned along the paths which are not needed.](/assets/images/merkle/pruned.svg){:class="centered-image" width="300px"}

While this may seem like a small victory in this example, consider the benefits at scale. Since we had stored the result of computing `C5` when we first built the Merkle tree, we now can just use that checksum directly. We no longer need to consider the `C1` and `C2` nodes (which could actually be large subtrees) as part of our verification.

To summarize, Merkle trees allow you to download just a partial set of the files and still perform verification as if you had the entire data set by requiring only a small number of related checksums.

### Aside: There's no way this is useful...

I know what you're thinking. Do I really need to generalize into this tree structure? Are we prematurely optimizing? Is this actually used anywhere?

It turns out I'm not just showing you these structures because I find them really interesting! 

Merkle trees actually form the foundation for many large-scale projects. 

- If you've ever used the BitTorrent protocol (or a number of other file-sharing protocols such as Gnutella), you've used a form of these trees under the hood for data verification. 
- If you've ever used Git or Mercurial for your version control, these use Merkle trees for organizing your stored code/commits, providing those useful SHA-1 hashes you love typing.
- If you've ever dabbled in cryptocurrency or used a blockchain, these leverage Merkle trees to verify that you're up to date on the transactions which have been processed without having to download all the transaction content.

### Credits

Thank you to [Matt Wetmore](http://mattwetmore.me/) and [David Kua](http://kua.io/) for editing this post.

### Footnotes

[^1]: Checksumming and hashing functions can offer a wide variety of properties and configurable settings depending on your needs. Minimizing collisions is a good goal - generally this is achieved by having larger output checksums, but this begins to be a tradeoff for storage space.
[^2]: The [pigeonhole principle](https://en.wikipedia.org/wiki/Pigeonhole_principle) describes this well. The degree to which this impacts you also depends on the function of your choosing: some guarantee that each outputs are uniformly distributed (ie. each checksum has roughly the same number of inputs which produce it within a certain space), some don't.
[^3]: By using a tree structure (and assuming that your tree is balanced), the operation of verifying a checksum goes from being linear (relative to the number of tree nodes) to logarithmic (as you only need to check the path). 