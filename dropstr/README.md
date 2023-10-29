# Dropstr

A concept for file sharing, via Nostr, next to Nostr.

## Features

- One-Shot uploader
    * Upload files with a single PUT or POST request.
    * Return a share-able URL
- Partially and fully anonymous files
    * Encrypted or Encrypted + Scrambled files
- Fixed chunk sizes
    * No more guessing; one size, that has to work.

## Inspiration and idea

I wanted to make this tool specifically and especially after having read a lot about Usenet's binary "articles" and their respective specification, plus having tinkered a good bit with IPFS. Both solutions have their pros and cons - but I think that it could be done better, much better.

I mainly used IPFS as a file uploader in combination with ShareX to quickly upload screenshots while playing games to circumvent Discord's file upload size limit for non-Nitro users - why should I pay for instant messaging, after all? It has been a mainstay on the internet in forever; Discord jsut happens to have captured a big enough audience that it has become a certain neccessity for communicating with others. Matrix had been my high hope for quite a while, but I feel rather disenchanted about it too - still much better than Discord. That said, something that all of them lack is properly sending and receiving files from other users.

I would like to change that with a file hosting and sharing format that can help both in sharing files around the internet and just to a friend of yours

## The three types

Dropstr is to implement three kinds of "files":

* Ones that can easily be searched with a basic filter, such as `{kind: n, tags: ["#file.name": "Foo.txt"]}`
* Ones that can be shared, but not be found, by providing unencrypted files via their event ID
* Ones that can be shared, but only read by those who *should*, by encrypting AND scambling them, to make it possibly hard to figure out what algorythmn was used, and where the de-scramble file is - this is mainly an idea taken directly from the Sony PlayStation 3's `.ird` files, that can be used to unscramble dumped games. A similar concept exists for their PlayStation 1 games as well.

This should allow you
* ... to share a file to everyone, publicy.
* ... to share a file in a post, but not neccessarily making it searchable.
* ... to share a file to someone else, so that only they can open it.

## The "protocol"

In layman's terms, the flow of uploading and publishing a file is effectively this:

- Alice sends the file `smiley.png` via a POST request to a dropstr instance. Because it is her own instance, only she has access to that API, by only exposing it to herself through a VPN or other form of encapsulation.
- She can now share a link akin to `https://dropstr.alice/deadbeef` into the world. But since the file was not encrypted, scrambled, or otherwise marked as being "private", it had also been published as a Nostr event.
- Anyone can connect to her dropstr instance and look for all the public files and download them at will by constructing an URL of `https://${hostname}/${event.id}`

If Alice wanted to share a file to be attached to a post however, she could do that by possibly appending a query parameter to her initial POST request, like `?public=false`, to cause her dropstr instance to not generate a filename tag in the resulting root event. And if she intended to only send the file to Bob, she could first encrypt and scramble it, generating a "table of contents" file akin to a `.cue` sheet often found with multi-purpose CD dumps (Audio + Data), where the sheet clearly describes which part of the dump is what.

## The events

Each file starts out as a root event, which is actually a list as per NIP-51:

```json
{
  "kind": 30330,
  "tags": [
    ["d", "smiley.png"],
    ["m", "image/png"],
    ["x", "..."],
    ["size", 123],
    ["f", "<chunk id>", "1"],
    ["f", "<chunk id>", "2"],
  ],
  "content": "[\"na]",
  // ...other fields
}
```

Whilst this follows the typical lists schema very closely (which is in fact intentional), it uses a different kind - but that is the only difference.

- Each `f`ilechunk tag has a numeric index set as the description (third array entry). This denotes the order, in case a JSON implementation confuses the actual order of entries.
- The `d`escription tag denotes the filename.
- Additional tags akin to NIP-94 may also be specified to offer further information about the file.
    * `url` may be used to offer a direct link to this dropstr instance, known to have the file available.
    * `m` may denote a MIME-Type.
    * `size` may contain the size, in bytes, of the final file.
    * `x` may contain a sha256 hash.
- The `content` may contain a full text description of the file, if wanted.

When such a root event is aquired, a client may immediately contact other dropstr instances to allow downloading from more than one host and query them for the various hashes referenced in the `e`-tags. Those should return actual NIP-94 files but only regarding this particular chunk.

```json
{
  "id": "<event ID referencing just one chunk>",
  "pubkey": ">pubkey of the chunk publisher>",
  "created_at": "<unix timestamp in seconds>",
  "kind": 1063,
  "tags": [
    ["url", "https://dropstr.instance/${event.tags.x}"],
    ["x", "<Hash SHA-256>"],
    ["size", "<size of file in bytes>"]
  ],
  "content": "",
  "sig": "<signature>"
}
```

This means that a client may query a dropstr instance with `{kind: 1063, tags: ["#x", $hash]}`, where `$hash` is the original 2nd array entry in the root's `f` tag. In addition, they may even look for events specifically published by the original publisher, or opt to ignore from whom they find the fitting hash. The latter option might come in handy for deduplication purposes.

Once the client has established a few dropstr instances from which it may download the chunks, it will effectively do the following - just, a lot of it:

1. Download the first chunk
2. Verify the first chunk based on the root event's `f` tag.
3. Download the second chunk
4. Verify the 2nd chunk
5. Concatenate the first and second chunk
6. Verify the concatenation based on the root event's `x` tag.

This way, each chunk is verified and the combination of all of them is verified again. At this point, the file has been reassembled, potentially across several relays and hosts.

In addition: A root event may list additional `relay` tags to hint at other known dropstr instances that either happen to have this event or might be good query targets. Those might be spread across Tor and i2p respectively. However, individual chunks should NOT use Torrents, magnets, or infohashes. Dropstr is ment to operate over basic protocols like HTTP, preferably with TLS enabled (HTTPS).

A file can easily be encrypted - but scrambling a file and creating a table of contents might be a bit new in this regard. A ToC file is downloaded separately and applied after all chunks have been verified and assembled. Only if the resulting file THEN matches the root event's `x` tag should it be considered "valid". Thus, in the numeric list above, insert the de-scrambling between step 5 and 6.

A potential library/program for this would be this: https://github.com/SKGleba/blobpak
Ideally a library were to be created that completely shuffles and scrambles a file, producing a `.toc` file to explain where what sections of the files had gone to, whilst the encryption method can be picked by the user to be as flexible as possible. The important thing is that a fully downloaded scrambled file may NOT immediately match the root event's `x` value - only after having been descrambled.

### Folders

A more complex setup would be to create an actual regular list event acting as a folder and listing either root events for file chunks, or additional folder events within itself. However, this goes way beyond the initial idea of "one shot uploader".

## The other half - Storage

Nostr is ONLY to be used to exchange metadata information about the files and their chunks. The actual storage should happen elsewhere - which could be anything from a local disk drive, to cloud storage. Since each hash is unique, a database could technically be used, taking advantage of the unique nature of the hashes to treat them as primary keys and encoding their binary data within a row. This offers a lot of flexibility in terms of implementation as to where data should be stored.

However, I did mention chunks before. Unlike IPFS, where various chunking methods can be used, each chunk in dropstr is rather strictly defined:

- A chunk may never exceed 100MB. This should allow most common files to "just fit", or at worst consume 2 to 3 chunks.
- A chunk that does not fill the limit, must specify a size.
- If a chunk is bigger than 100MB, the entire event and it's referenced chunks should be treated as "illegal", or at least "invalid".
- A chunk mustn't neccessarily belong to the same pubkey as the root event.

## Endpoints

There are two basic endpoints: One is a public gateway with the following route spec:

`GET /f/:nostr-event-id`
- `:nostr-event-id` is a raw event ID referencing a root event.
`GET /c/:sha256`
- `:sha256` is a reference to a chunk and should immediately return a `Content-Type: text/octed-stream` and apropriate filesize header

An additional HTTP server may be started as the pure control-server to post and delete data to and from the instance:

`DELETE /f/:nostr-event-id[?all=true|false]`
- `:nostr-event-id` refers to the root event to be deleted
- `all` is a parameter that denotes if just the root event, or also subsequent chunks should be deleted.
`DELETE /c/:sha256`
- `:sha256` denotes a chunk to be deleted by it's sha256 hash.
`POST /[?public=true|false[&filename=:filename]]`
- `public` denotes if any public information should be included in the resulting root object or not.
- If the former is true, then `filename` should set the filename that is to be specified in the root event.
- Should the `Content-Type` be `text/plain`, then the body's content is to be placed in the root event's `content` field to act as a description.

## The decentralization factor

Inherently, everything in Nostr is public. So it would definitively be possible for other relays to download entire files or just chunks from other instances and then offer a mirrored copy on their end. However, dropstr *itself* may be used to also share those events around as a proactive aware-making by acting as a Nostr client itself and attempting to publish to other dropstr instances. This would offer the following advantages:

- Since dropstr is a relay itself, it may reject or accept events from anyone.
- Each user can broadcast root events further to spread knowledge of a file.
- Each dropstr instance owner can also decide to download entire files and publish them on their own relay - partially or entirely.
- The second half of the control API may be made public and use NIP-98 as an authorization method to either be granted access or not. This could even be combined with Lightning-paid hosting, where a user with a certain npub may be allowed to write files to the host, which would also give them the opportunity to link uploaded files to that particular npub, and thus entirely prune them, should there be a need for it.
- Since encrypted, non-public files are not found by default, they can also not be proactively replicated; unless the user is willing to take the risk to replicate something they can not even open themself.

# Conclusion

Where IPFS uses a DHT and Usenet uses sort-of distributed news hosts, nostr lives and dies by the network of relays that can be freely or commercially operated, giving everyone the choice in how they want to engage with this, if at all, and aproprietly lock their relays from receiving such events, should they wish to.