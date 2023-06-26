# Chatting

Since mid-2019, I have been using Matrix as my primary chat protocol for basically everything. With the help of bridges to Telegram and other social media and instant messaging apps, I didn't have to throw a bunch of apps on my phone - just two, Discord and Element. The reason I didn't bridge Discord is quite simple, they don't allow it. Classic walled garden, I guess. That said, Discord has features that Element can not - and probably never will - replicate.

## Why Matrix?

Because it shares an awful lot of common points with how Nostr events work. In fact; they both call each thing that happens an event in the first place!

In Matrix, a typical event looks like this:

```json
{
  "content": {
    "body": "What is the recommended way to draft NIPs? To start in your own repo first and then upstream the changes via PR or which way should I go? I have a few ideas and I want to write up the documentation first for a while, then write a small example app and then later turn that into a proper NIP.",
    "msgtype": "m.text"
  },
  "event_id": "$EFSX7kXerUfCZFoQuMTt28NW93GYDjVVegtxeyIWXL4",
  "origin_server_ts": 1687755373909,
  "sender": "@ingwiephoenix:ingwie.me",
  "type": "m.room.message",
  "unsigned": {
    "transaction_id": "m1687755373779.0"
  },
  "room_id": "!pVsDKlXKMpoZFcukgu:matrix.org"
}
```

(This is a message I sent to the Telegram Nostr group through my Matrix bridge!)

And now, a Nostr note in comparison:

```json
{
  "content": "Neat. xD\n\nhttps://void.cat/d/DzUngTgs4oDtHc8DMmCALV.webp",
  "created_at": 1686954090,
  "id": "c76f190c2ae1f3a4307e0e49d808105ca82c0f255605cc8abc7d73ffdf4c92ce",
  "kind": 1,
  "pubkey": "5e336907a3dda5cd58f11d162d8a4c9388f9cfb2f8dc4b469c8151e379c63bc9",
  "sig": "4ec90b9ac11c5c998dd13096576e09cb38744559b09ef2c46e1e413286d231e27da022ed567e00cbb913d1d23ec15d38b6ce9075d26d3ca8ea59b0fa2e37c0fe",
  "tags": [],
  "relays": [
    "wss://nostr.ingwie.me/",
    "wss://nos.lol/",
    "wss://relay.damus.io/",
    "wss://relay.nostr.band/"
  ]
}
```

The following keys have effectively the exact same meaning across both protocols:

| Matrix             | Nostr        | Meaning
| -------------------|--------------|---------------------------------------
| `content`          | `content`    | The message/note content.
| `sender`           | `pubkey`     | Who sent it?
| `origin_server_ts` | `created_at` | When was this event published/created?
| `event_id`         | `id`         | Unique event identifier
| `type`             | `kind`       | The type of event

Though there are a few differences in how they are typed and used. In Nostr, event types are nummeric, whilst in Matrix they have a domain-style identifier. And, in Matrix, content can be of a different type, whilst in Nostr, it is plain text most of the time.

And this goes throughout the **entire** protocol. Hence, why I think it would lend itself well for being augmented into a Nostr specification.

## The goal

Right now, [NIP-04](https://github.com/nostr-protocol/nips/blob/master/04.md) is solid and serviceable - but it shares the same problem as with Matrix. It is only Message-Level Encryption. That means all the other metadata - from who to who, when, how long, ... - is still present. And, public. Like, fully public. You can totally query just about any Nostr relay for `kind:4` events with ease.

And, [NIP-28](https://github.com/nostr-protocol/nips/blob/master/28.md) is not encrypted and fully public entirely. Encrypted group chats do not exist, period.

My goal is to use the vast development that Matrix has seen in order to dramatically improve Nostr's chatting capabilities. 

Basically, my goal is to implement these Matrix features:

* Encrypted group chats
* Group chat moderation
* Threading in chats
* Spaces

In addition, ontop of "encrypted" chats, I want to also integrate a higher level of encryption, "Trusted Chat".

### Trusted Chats

Encryption, right now, is negotiated entirely in public. And as mentioned previously, it still has it's metadata fully attached. Just the message itself is encrypted. But, what if two people knew each other in person? They could make up a passphrase that would then be used to encrypt the **entire** object. Idealy, this would look something like this:

```json
{
  "id": "some ID",
  "created_at": 1686954090,
  "kind": 44, // Placeholder
  "content": "encrypted object",
  "tags": [
    ["m", "2nd passphrase"]
  ],
  "relays": [
    "wss://nostr.ingwie.me/",
    "wss://nos.lol/",
    "wss://relay.damus.io/",
    "wss://relay.nostr.band/"
  ]
}
```

That encrypted object would result in one that lacks an ID and `created_at` field, but would instead contain tags, signature and everything else. These events may be found with a supplementary passphrase, because otherwise there is literally nothing to identify the users.

However, since it is entirely identifier-less, it is hard to suggest just about every relay to allow these kinds of messages to be stored. Be it with a relay accepting `AUTH` or another method. There might even be relays specializing in only doing this kind of event, and only this, deleting any event that is older than a certain amount of time to keep their storage free in return.

Now, I mentioned two passphrases: One is to encrypt the object itself, allowing only the parties that have that passphrase to decrypt it, whilst the other is effectively a glorified search term. It could also just be a random ID, really. This, however, also means that the entire conversation can be tracked by it. However, no data is leaked, so one couldn't tell who said what to whom.

These passphrases should only be exchanged in person, or via burner accounts online on a platform that is not going to store them for long or ever. i.e.: expiring and password protected pastebins.

This *should* allow uninterrupted and uncensorable communication between peers - even in a group - that can not be read by anyone. Or, technically, at least.

Disclaimer: I suck at cryptography. (:

## Matrix: Message types

Before we can adapt anything into Nostr, it is important to list out all the potential types of messages, and see if there is equivalents within Nostr already.

(To see it in the Matrix spec, see [7.1. Types of room events](https://spec.matrix.org/v1.7/client-server-api/#types-of-room-events))

| Matrix `type`                  | Nostr `kind` | Is...
|--------------------------------|--------------|--------------------------------------------------------------------------------------------------------------------------------------
| `m.room.create`                | 40 (-ish)    | the root event of a room, it's creation.
| `m.room.name`                  | 41           | an event to change the room name. Think updated `kind:0`.
| `m.room.avatar`                | 41           | the same as `.name`, but for it's avatar/icon.
| `m.room.topic`                 | 41           | again the same as above, but for it's description.
| `m.room.join_rules`            | None         | a definition who can and can not join. Irrelevant for Nostr, probably.
| `m.room.canonical_alias`       | 41?          | kinda like NIP-05 - a proper name associated to gibberish.
| `m.room.history_visibility`    | 41?          | the definition of what can be seen of the backlog.
| `m.room.encryption`            | None         | complicated... It denotes encryption information.
| `m.room.member`                | None?        | basically an invite, and also the current status of membership.
| `m.room.power_level`           | None         | an object describing permissions. I.e.: When a user can post etc.
| `m.room.message`               | 42           | an actual message being sent. [And there's many types of that.](https://spec.matrix.org/v1.7/client-server-api/#mroommessage-msgtypes)
| `m.room.pinned_events`         | None         | pinned messages; stuff that is important or alike.
| `m.room.tombstone`             | None?         | Room has moved to a new location or is otherwise deleted.
| `m.direct`                     | None?        | sent by the Matrix server to say which rooms are actually DMs. Irrelevant; we have kinds for that :)
| `m.call.answer`                | None         | an event indicating someone picked up a call.
| `m.call.candidates`            | None         | basically WebRTC ICE stuff.
| `m.call.hangup`                | None         | just a call party member ending the call.
| `m.call.invite`                | None         | someone inviting someone else into a call.
| `m.call.negotiate`             | None         | how clients negotiate capabilities of a call. Probably preceeds `.candidates`.
| `m.call.reject`                | None         | like `.hangup`, but the user never even answered.
| `m.call.select_answer`         | None         | a way to tell everyone who you are now calling with.
| `m.typing`                     | None         | a typing indicator. Should DEFINITIVELY be an ephemeral event. Sent by server.
| `m.receipt`                    | None         | an indicator that a user read a message.
| `m.read`                       | None         | is sent along `m.receipt` as a "who read your message" kinda thing.
| `m.fully_read`                 | None         | sent by server as part of the state, indicating where they stopped reading. Possibly not important for Nostr.
| `m.presence`                   | None         | is an Offline/Online etc. indicator.
| `m.policy.rule.user`           | None         | an elaborate description of a moderation. Kicks, bans, etc.
| `m.policy.rule.room`           | None         | the same as above, but for a room. Likely useless for Nostr.
| `m.policy.rule.server`         | None         | definitively not applicable to Nostr.
| `m.space`                      | 41-ish       | an event to explicitly mark this as a Space, rather than a simple room. (Almost everything in Matrix is a room.)
| `m.space.child`/`.parent`      | None         | used to create a room list associated to a space.
| `m.relates_to`                 | `tag:e`      | Establishes event relations. Often used for replies or edits.
| `m.reaction`                   | 25           | a reaction to an event.

In addition, There are also a few different types of `m.room.message`. Whilst those aren't important for a server, they should be for a client, normally. In Nostr, this probably doesn't do a whole lot. However, they could be respected and shown appropreately as a separate tag.

| `m.room.message.msgType` | Is...
|--------------------------|------------------------------------------------------------------------------------------
| `m.text`                 | a plain text message.
| `m.emote`                | an action, like `/me sips coffee.`. Just a fancy `m.text`, really.
| `m.notice`               | usually a bot message, announcement or something minor.
| `m.image`                | a post containing just an image.
| `m.file`                 | a file that is being shared. Could be useful to just specify an IPFS CID instead, perhaps.
| `m.audio`                | a sound file; often used for voice messages.
| `m.location`             | a set of geo-coordinates.
| `m.video`                | like image, audio and file - but a video.
| `m.sticker`              | one big sticker. Likely part of a set, too.

And these are some events specific for moderation:

| Type    | Is...
|---------|----------------------------
| `m.ban` | a recommendation for a ban.


I omitted most of the E2E stuff for now, but it's in [11.12](https://spec.matrix.org/v1.7/client-server-api/#end-to-end-encryption). The used encryptions are noted here: [11.12.4. Message Algorythmns](https://spec.matrix.org/v1.7/client-server-api/#molmv1curve25519-aes-sha2). That last section in particular goes a little over my head, I will admit... The long-story-short is, however, that OLM's spec is used for most of it.

In addition, I also ignored a few state events that indicate the current situation of the user. In Nostr, there is no such thing as a server; it ALL happens on the client side, and thus this stuff should also be resolved there. However, since not everything in Nostr is just a `m.room.*`, it's also far easier to tell certain things apart.

### Conclusion

You may have noticed the many `None` declarations in the table above - but, this isn't entirely true, as most of those are events that Nostr never needed or rather, needed to have. In addition, there a lot of events that just don't make sense in Nostr in the first place; such as banning federated rooms. That simply doesn't exist within the Nostr protocol - everything is, technically, "global".

But we can summise that...

* `m.room.*`: All events responsible for whatever happens inside a room.
* `m.call.*`: Responsible for calling. Matrix leans heavily on WebRTC. Now, since [Nostr Nets](https://nostrnests.com) are a thing, I won't bother with that, for now.
* `m.policy.*`: These are all moderation events.

The rest are basically state events that have some significance though; maybe you do want to share a read-receipt or a typing notification.

In addition, all the `m.room.message.msgType` specifiers are effectively useless. In Nostr, you will be sending links most of the time anyway and can fetch the metadata as you fetch the resource; it all happens on the client anyway. It could, however, be interesting to add details about attached files into a combination of an `f` tag and a small serialized string.

For instance, imagine `kind:660` was a message in a public chat:

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "kind": 660,
    "created_at": 1234,
    "content": "This is a neat icon!\n#[0]",
    "tags": [
        ["f", "https://ipfs.io/ipfs/someCIDhere", "img:256x256:jpeg"]
    ]
}
```

From here, the attached file via it's URL can have a small description of what the client can expect. However, this would constitute yet another NIP just for the `f` tag. It's a thought, and for now, I will leave it at that.

## The nitty-gritty

### TL;DR: Kinds

| `kind` | Purpose
|--------|----------------------------
| 14400  | Channel create
| 14401  | Channel metadata update
| 14402  | Channel message
| 104405 | Owner: Set permission state
| 14410  | Mod: Delete message
| 14411  | Mod: Kick user
| 14412  | Mod: Ban user
| 14420  | Zone creation
| 14421  | Zone metadata update
| 104422 | Zone channel list

### DMs, Rooms, Spaces.

Telegram groups are super popular and Discord servers are the norm. Matrix Spaces have only recently been a relatively stable feature as well.

So, let's start with the easiest: An unencrypted, public room.

First off, we have to create it. I will use placeholder kind numbers for now - expect these to change as I hammer in the various event types. 

To create a public room, we create it first similiar to a `kind:40`. Whilst this could be used here, I will stick to custom kinds for the time being. However, technically, `kind:40` to `kind:44` are probably fully compatible. They just lack moderation and editing those would be a bad idea as it could totally break compatibility with older clients and relays.

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 1234,
    "kind": 14400,
    "content": "",
    "tags": [
        ["relays", "wss://relay.example.com", "wss://other-relay.example.com"],
        ["title", "Channel title"],
        ["description", "Topic"],
        ["image", "URL to channel icon"]
    ]
}
```

Since these tags (relays, title, description, image) are already standarized, it should make the message object far cleaner compared to stringifying JSON. In this case, the channel metadata is defined within tags. However, the most special one here is `relays`. This denotes the prefered relays for a channel and is completely optional. Though it can be used to help discoverability. Moreover, I am not sure if it is even needed - this might literally be my "Matrix-brain" speaking...

The ID of the channel is now it's main identifier. An appropriate `kind:14401` event may be sent to update that information, similiar to `kind:41`, but using the tags instead

From this point on, the pubkey which created the channel is automatically treated as "Owner" and "Admin". We'll get to this in a moment.

Next, a user might want to send a message to that channel. Well first, they might want to join the channel and see all of it's backlog. This is fulyl client-side; the client just has to keep track of that channel and store it approprietly. Probably `kind:30078`?

To send a message, they send a `kind:42`-alike message. In fact, it is the same layout.

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "kind": 14402,
    "created_at": 1234,
    "content": "Hello!",
    "tags": [
        ["e", "kind:14400 ID", "root/reply"],
        ["p", "pubkey of reply, if applicable."]
    ]
}
```

Just as `kind:42`, a message should be marked as a root or reply message respectively - and this in fact fully implements threading immediately. The only iteration to do threading a little better would be to create a small event indicating the start of a dedicated thread; like Discord's, that may have an expiration date unless something happened in it for a while. However, this seems overkill. For now.

What *could* be added is an `m` tag to denote the very specific kind of message - it's sub-kind, if you will - like the `m.text` Matrix specifiers. This allows voice messages and other such specific ones:

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "kind": 14402,
    "created_at": 1234,
    "content": "#[0]",
    "tags": [
        ["e", "kind:14400 ID", "root"],
        ["m", "audio"],
        ["f", "https://files.example.com/voicemessage.ogg", "audio:180:ogg"]
    ]
}
```

Unlike `kind:42`, this enables entirely new kinds of messages. And, to be blunt, it's easy to rip them from Matrix. ;) In addition, the `f` tag can be used to help the client set proper expectations. Alternatively, `kind:1063` should be sufficient and can be linked with a `e` tag.

A user may now delete their message as per `kind:5` and react as per `kind:7`.

This all was just for public, unencrypted rooms. But we still haven't discussed DMs and Spaces. Honestly, I neither like the term "Space" nor "Server" - and "Community" is already about to be used... So, I will use "Zone". I don't like it a whole lot - reinventing the wheel for the sake of differenciation and all - but for the purposes of this draft, it should do.

A zone is effectively like a channel - but, more. It contains a full structured list of all the channels, ordered in categories. Yes, it's a Discord Server, I know.

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 1234,
    "kind": 14420,
    "content": "",
    "tags": [
        ["relays", "wss://relay.example.com", "wss://other-relay.example.com"],
        ["title", "Zone title"],
        ["description", "Description of the zone"],
        ["image", "URL to zone icon"],
        ["banner", "URL to a banner"]
    ]
}
```

A zone's description is expected to be long and possibly formatted in Markdown. This is supposed to be shown to users browsing known zones in a relay.

Subsequent changes can be done via `kind:14421`.

However, compared to the creation of a channel, a zone has several categories and channels under it. [Pinstr](https://pinstr.app/) already does lists and it is a noteable shoutout at this point. Since these channel lists change, this is a replaceable event.

The basic structure is very simple:

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 1234,
    "kind": 104422,
    "content": "$obj",
    "tags": [
        ["e", "kind:14420 event"]
    ]
}
```

However, the contents of `$obj` are the actual interesting part. This is a stringified JSON object containing a full category and channel map, literally.

```json
{
    "_": [
        "kind:14400 id",
        "kind:14400 id",
        "..."
    ],
    "Announcements": [
        "kind:14400 id"
    ],
    "Another category": [
        "kind:14400 id"
    ]
}
```

Each key in the object represents a category, whilst each entry in the array represents a channel - in this specific order. The special value `_` is inspired by NIP-05, and means "root" or "category less" and should ideally appear at the top of the list right underneath the zone's branding (icon, banner).

And with that, only DMs are left. But before I talk more about these, I need a lot more research into Matrix' OLM and friends. For now, this is a TODO.

### Administration and Moderation

This is where things diverge a lot from the NIP-40 line. A lot.

New event types cover:

- Announcing moderators (as in, giving them their status)
- Doing moderation (Deleting messages, kicking or banning users)


First, let's start with actually making someone a moderator. Since channels are ment to be basic and not filled to the brim with permissions, but also be forward compatible, a "Grant" event should be used. Since it can also be revoked however, this is more like a publicy known state - and hence, it fits nicely into the category "Replaceable Events".

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 12345,
    "kind": 104405,
    "content": "[\"kick\",\"ban\",\"delete\"]",
    "tags": [
        ["d", "target pubkey"],
        ["e", "channel's 14400 ID"]
    ]
}
```

IF the owner signed this event, THEN the specified pubkey can do what the array in "content" listed. In this case, they can kick, ban (users) and delete (messages). However, this is the first instance, in which a relay needs to do a little bit of background work: It has to verify that this event is actually legal and counter-check.

1. Query the `e` target.
2. Link the channel and permission grant together to figure out who is a mod.
3. If approved, perform the action.

To illustrate:

* Alice creates a channel (`kind:14400`)
* Alice makes Bob a moderator (`kind:14405`)
* James starts to behave like a douche, so Bob wants to kick him for a while.
* Bob sends the kick event (`kind:14411`) to his relay with an explicit expiration date.
* James can still send messages to the relay, but they will be rejected as he has been kicked.
* Once the kick expires, James can post again.

The ban event (`kind:14412`) behaves similarily.

```json
{
    "pubkey": "Bob's key in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 12345,
    "kind": 14411,
    "content": "Reason for kick, optional",
    "tags": [
        ["p", "James' pubkey"],
        ["expiration", 12345]
    ]
}
```

A ban and message delete don't have an expiration.

This is a message delete:

```json
{
    "pubkey": "Bob's key in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 12345,
    "kind": 14410,
    "content": "Reason for deleting the message, optional",
    "tags": [
        ["e", "James' pubkey"]
    ]
}
```

Instead of a `p`erson, an `e`vent is tagged instead.

Once the channel creator feels like their mod should no longer have their position, they can revoke their status.

```json
{
    "pubkey": "pubkey in hex",
    "sig": "...",
    "id": "some id",
    "created_at": 12345,
    "kind": 104405,
    "content": "[]",
    "tags": [
        ["d", "target pubkey"],
        ["e", "channel's 14400 ID"]
    ]
}
```

Setting it to an empty array means they have no special permissions anymore.

And finally, the channel owner is fed up with their channel and just wants to delete it entirely. Just use `kind:5` on the channel ID - done.

### Relays

You might have noticed that by now, but there are a lot of backend checks that need to be performed. This is specifically why I would personally recommend chat-specific relays. There will be more processing work involved. That said, if a relay hoster is feeling adventureous, they might just support it all at once.

### Missing features

#### Roles

Roles, especially on Discord, are like badges that can serve administrative purposes - as in, giving permissions and such - and can also be purely symbolic; a membership status, a special name colour, a certain trait, ... Since Nostr already has badges, there is no point in implementing a new system for it, so a user should just use that. However, it should be considered to implement this as part of the channel's meta, possibly an additional meta type, that only deals with "extended attributes", per-se.

#### Calls

I also have not talked at all about the `m.call.*` parts - I have honestly not done my WebRTC homework entirely just yet to put out a proper description for that. Also, Nostr Nests is a thing and possibly a much better integration here.

#### Human readable names / Verifications

For now, IDs were used as the ultimate identifiers. But, we have NIP-05; so it should have some use here. But, I haven't fully decided on what kind of URI I would like to use that can be verified against a domain.

Current ideas:

- `!name@domain.tld`: This would closely resemble what Matrix does internally, where rooms with a proper name are prefixed with a hash, but fully internally, it uses an exclamation point (`!abc:server` and `#nice_name:server` respectively). Also, hashtags are already in great use on Nostr. Also, Lemmy/KBin use this. For this reason alone, it might also not be a good idea to use it, since there is a full Reddit-alike NIP in the works. I don't want to interfere there.
- `$name@domain.tld`: Could be possible.
- `domain.tld/$name`: Instead of putting the name before the domain, one could use the domain itself as a differenciator. For instance: `tech.bros/$arm_cpus` could be a usable name.

Then there is also the fact that there must be a differenciation between channels and zones.

In an ideal situation, you would find these human readable names on Nostr, standarized (disclaimer, this is pure speculation and examples!):

- `user@domain`: NIP-05 for an individual profile. As in: "User" at "this domain".
- `domain/community`: Reddit-like community, using the domain as the main curation method. As in: "waifu.fun/toplists" would be "Toplists" in the "waifu.fun" namespace.
- `$zone@domain`: A zone. As in, "$nerds@car.freaks" would be the "nerds" zone at the "car.freaks" namespace.
    * If it was *just* `$domain`, the entire domain would be treated as the zone, indirectly translating to `$_@domain`.
- `domain/!channel`: A channel. As in, "big.world/!music" would be the "music" channel in the "big.world" namespace.

However, I am very much not satisfied with this yet. The cool thing about NIP-05 is that it is a proper internet identifier, meaning if you punch it into an E-Mail client, you can send an email. XMPP also uses the same notation! If the `domain/community` naming was to be used, it could easily lead to a whole homepage full of additional resources or even a wiki and related things, because it is quite natural. It could also just outright be a HTTP redirect to a `nostr:` URL that points to that very community, too. 

But, which formats do make sense for zones and their channels? Should zones and channels be intrinsicaly linked - or not? If they should, then `$zone@domain/!channel` would look weird - you can't string a nice HTTP URL from this. So this is clearly something that needs work.

#### Public vs. Invite-only

There might be public rooms for which you need an invitation to be allowed to speak in. This also needs more thought, allowing a more fine-grained control of who is allowed to access a channel - or zone - and who is not.

#### Trust (passphrase based) encryption in invite-only channels aka. encrypted groups

You might have more than one friend with whom you would love to chat entirely privately. For that, all parties would just need the same passphrase. But for this to work, invite-only channels would have to exist, first.

#### Typing and read notification

Typing is clearly an ephemeral event, but I am not sure on read receipts just yet. They should also be optional.