---
date: 2020-07-01
url: /2020-07-01-rust-ipfs-unixfs/
header_image: rust-ipfs-crab.png
title: Rust IPFS UnixFS Support Lands
author: Joonas Koivunen and Mark Robert Henderson
tags: Rust, IPFS, UnixFS
---

UnixFS exporting has landed in [Rust IPFS]!

This post provides a review of what UnixFS is, and then for Rust developers we provide a detailed
overfiew of the [`ipfs-unixfs`] crate and its usage. If you're not a Rust developer, or if you'd
simply like to learn more about the HTTP APIs, we conclude with an overview of the new `/cat` and
`/get` endpoints.

[Rust IPFS]: https://github.com/rs-ipfs/rust-ipfs
[ipfs-unixfs]: https://crates.io/crates/ipfs-unixfs
[devgrant]: https://github.com/ipfs/devgrants/tree/master/open-grants/ipfs-rust/phase-2
[`ipfs-unixfs`]: https://crates.io/crates/ipfs-unixfs

## A UnixFS Refresher

UnixFS is a [Protocol Buffers] encoding used to represent Unix-alike files IPFS
MerkleDAG. UnixFS is also the functionality that's used behind the `ipfs add`, `ipfs get`,
`ipfs cat` commands, as well as their corresponding HTTP API endpoints.

Here's how it works - your raw bytes are encoded with specific fields. It can easily encode your
raw data behind a CID, but also it handles directories like so:

```
Data {
  Type: Directory
}
...
Links {
  Hash: "cid5"
  Name: "lib.rs"
  Tsize: 39334
}
..
Links {
  Hash: "cid7"
  Name: "repo"
  Tsize: 32018
}
```

From there, the bytes produced from the protbuf encoding are then encoded into the `dag-pb` format,
giving you the final CID that `ipfs add` produces. Our [previous post] on this topic used a
Matryoshka doll analogy:

![Nested dolls of bytes -> UnixFS -> dag-pb](https://miro.medium.com/max/1400/1*DLsR9Q8hMsDv0G98DFeMww.png)

Basically, it's encodings all the way down to bytes and all the way back up to your CID. You can
read more, and even try it yourself in our [previous post] on the topic. From here, we'll take
both a theoretical and practical look into the `ipfs-unixfs` crate.

[Protocol Buffers]: https://developers.google.com/protocol-buffers
[previous post]: https://medium.com/equilibriumco/the-road-to-unixfs-f3cf5222b2ef

## Unboxing the [ipfs-unixfs] crate

If you're a Rust developer, you now have this functionality available to you, both in
[Rust IPFS], as well as the standalone [`ipfs-unixfs`] crate. It helps to understand the theory
behind UnixFS, because it informs the decisions we made in the implementation.

Any UnixFS exporter implementation should provide:

1. An interface to the data contained in the blocks. (This is fairly straightforward)
2. An ability to "walk" across multiple blocks via the MerkleDAG. (This is much less
straightforward.)

Most of our work was finding a suitable abstraction for #2. The essential operation needed is something like:

{{<highlight rust>}}
async fn get_block(..., cid: Cid) -> Result<impl AsRef<[u8]>, _>
{{< / highlight >}}Translation: Given a CID, eventually return something that can be represented as
a pointer to some bytes, or an error.

On its face it seems simple, but we wanted to find a way to keep the IO of blocks separate from the
implementation itself. This way, any store integration - be it async or blocking - should be
possible, and you'll able to compose the higher level operations out of the lower level
pieces.

So, instead of having something like a `GetBlock` or `BlockStore` trait, the
`ipfs_unixfs::walk::Walker` currently has an API like this:

{{<highlight rust>}}
impl Walker {
  fn new(start: Cid, root_name: &str) -> Walker;

  fn pending_links(&self) -> (&Cid, impl Iterator<Item = &Cid> + '_) { ... }

  fn continue_walk(self, next_block: &[u8]) -> Result<ContinuedWalk, _> { ... }
}
{{< / highlight >}}

Let's step through this implementation one function at a time.

`Walker::new`

A new `Walker` is created by passing in a start `Cid` and an optional `root_name`, or the path of
the root document. Once created, the `Walker will start traversing the path inside the graph for
entries.

`Walker::pending_links`

This function may have a gnarly-looking signature, at the same time it communicates an important
invariant of the API: whenever there exists a `Walker` value, there must be some pending links to
load. This also means that the calling code does not have to deal with unwrapping an `Option<Cid>`
from an `Iterator::next`, but can just get the next `Cid` as follows:

{{<highlight rust>}}
let (next, prefetchable_links) = walker.pending_links();
{{</highlight>}}

The iterator (`prefetchable_links`) in the tuple can be used to start pre-fetching the `Cid`s from
the network. When walking a directory, `prefetchable_links` will contain unvisited links to entries
under the directory. However, following an opening of a multi-block file, `prefetchable_links`
would now contain the next, most important, links for the file. Since files can expand on every
block before hitting the leaf level, it can be a long time before any of the previously seen
"prefetchable" blocks are ready to be processed.

`Walker::continue_walk`

Continuing the walk returns a `ContinuedWalk` on success, which is an enum of three possibilities:

1. file (new or continuation)
2. directory (new or continuation)
3. symlink

For files and symlinks, there is an additional [`FileSegment`] or a slice of bytes
provided. This represents a segment of the file, which could possibly be empty.
For symlinks the slice is the linked path of the `Symlink`.

From a `ContinuedWalk` value the walk can be continued by first accessing the
inner `Item` value either by pattern matching or by using
`ContinuedWalk::into_inner`. The [`Item::into_inner`] will return an `Option`
of the next `Walker`. One will exist if there are still links to walk.

### What you can use it for?

TODO

[MerkleDAG is the outer protocol buffers description]: https://github.com/ipfs/go-merkledag/blob/master/pb/merkledag.proto
[UnixFS messages]: https://github.com/ipfs/specs/blob/master/UNIXFS.md
[`FileSegment`]: https://docs.rs/ipfs-unixfs/0.0.1/ipfs_unixfs/walk/struct.FileSegment.html
[issue #200]: https://github.com/rs-ipfs/rust-ipfs/issues/200
[`Item::into_inner`]: https://docs.rs/ipfs-unixfs/0.0.1/ipfs_unixfs/walk/struct.Item.html#method.into_inner

## Not a Rust developer? Don't fret, you have `/cat` and `/get`!

In short, these two operations **retrieve** data from MerkleDAGs and, in the
process, fetch the required blocks from the network. [`/cat`] can only process
UnixFS blocks of type `File` or `Raw`, while [`/get`] can start from a
block of type `Directory`, `File` or `Symlink`.

Depending on the type of the block, the walk will only consist of a single
`File` or `Symlink` block, a single `File` tree, or some combination,
including sub-directories in a directory hierarchy. The `/cat` HTTP API returns
the raw file whereas `/get` ([HTTP impl](https://github.com/rs-ipfs/rust-ipfs/blob/a1614011a330d32842352bf1095219e6b068b92a/http/src/v0/root_files.rs#L86-L195))
returns a tar file of the tree.

Exposing the `/cat` operation ([HTTP impl](https://github.com/rs-ipfs/rust-ipfs/blob/a1614011a330d32842352bf1095219e6b068b92a/http/src/v0/root_files.rs#L38-L69))
as a Rust API was the easier part, as it is a pretty good fit for a
`futures::stream::Stream<Item = Result<Vec<u8>, _>>` or
`futures::stream::TryStream<Ok = Vec<u8>, Error = _>`. As we had already used
`async_stream` over at `ipfs-http` for the initial `refs` implementation, it
again seemed like the easiest way to get this implemented.

The implementation ended up using `Vec<u8>` for bytes instead of `bytes::Bytes`
simply because we currently didn't depend on `bytes` even indirectly in `ipfs`.
It might be that the type in the signature should be changed to
`impl AsRef<[u8]>` to allow later migration to `Bytes`.

[`/cat`]: https://docs.ipfs.io/reference/http/api/#api-v0-cat
[`/get`]: https://docs.ipfs.io/reference/http/api/#api-v0-get

File metadata could potentially be handled by first returning a
value which would contain the eventual stream of bytes. This would be a welcome
addition to the `ipfs::Ipfs::cat_unixfs` API, which currently returns an
`Future` of the `Stream` of bytes -- instead it should return a `Future` of a
`File` description, which would in turn contain the stream of bytes.

## How you can get involved

- Try out [Rust IPFS] and provide feedback via GitHub issues
- Check out the [good first issue] and [help wanted] labels in the Rust IPFS repo
- Sponsor our work on [OpenCollective]
- Learn what the Equilibrium team has [next in store for Rust IPFS]

[Rust IPFS]: https://github.com/rs-ipfs/rust-ipfs
[help wanted]: https://github.com/rs-ipfs/rust-ipfs/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22
[good first issue]: https://github.com/rs-ipfs/rust-ipfs/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22u
[OpenCollective]: https://opencollective.com/rs-ipfs
[next in store for Rust IPFS]: https://medium.com/equilibriumco/unixfs-exporting-has-landed-what-comes-next-4775cc568838

Special thanks to Protocol Labs for their [devgrant] support.
