ClearSkies Protocol v1 Draft
=========================

The ClearSkies protocol is a two-way (or multi-way) directory synchronization
protocol, inspired by BitTorrent Sync.  It is a friend-to-friend protocol as
opposed to a peer-to-peer protocol, meaning that files aren't shared with all
peers in the network.


Draft Status
------------

This is a draft of the version 1 protocol and is subject to changes as the need
arises.  The final version will replace this document.


License
-------

This protocol documentation is in the public domain.  Private implementations
of this protocol are not constrained by the terms of the GPL v3.


Shared secrets
---------------

When a directory is first shared, a 256-bit encryption key is generated.  This
should not be generated with a psuedo-random number generator (PRNG), but
should come from a source of cryptographically secure numbers, such as
"/dev/random" on Linux, CryptGenRandom() on Windows, or RAND_bytes() in
OpenSSL.

Instead of generating a random key, the user may enter a custom passphrase.
The SHA256 algorithm is applied twenty million times to generate the 256-bit
encryption key, meaning SHA256(SHA256(...(SHA256(SHA256("secret")))...)).  Due
to the nature of the secret sharing mechanism, it is not possible to use a salt
as is done with with PBKDF2, but someone possessing the key would already have
access to the files themselves, making the passphrase less valuable.

This key is known as the read-write key, and is an ECC private key, using
secp256k1.  The "public" key for the ECC private key is the read-only key.  Due
to the way it is used in clearskies, it should not be shared publicly, and will
be referred to hereafter as the "verification key" when used to verify
signatures.

The read-only key is used as if it were a new private key, to generate a new
public key, which is the known as the "untrusted key", which is used for
backups to untrusted hosts.

Finally, the process is repeated once more to create the share ID, which is not
secret.  In order to avoid confusion, this is never called a "key".

Any one of the three keys can be shared by the user to share the associated
files in different modes, which puts constraints on key length and salting.

The human-sharable versions of the keys are written with a prefix of
'CLEARSKY', a letter that represents the key type, 'W' for the read-write key,
'R' for the read-only key, and 'U' for the untrusted key.  The key data itself
is be represented as base32, as defined in [RFC
4648](http://tools.ietf.org/html/rfc4648), with the equals-sign padding at the
end removed.  Finally, a LUN check digit is added, using the [LUN mod N
algorithm](http://en.wikipedia.org/wiki/Luhn_mod_N_algorithm).

The share ID is represented as hex to avoid confusion, and is not shown to end
users.

To share a directory with someone the read-only or read-write key is
shared by some other means (over telephone, in person, by QR code, etc.)

The passphrase can also be shared instead of the read-write key, if desired, or
can be committed to memory.

To create an encrypted backup, the "untrusted" key can be shared.  The remote
host will be given encrypted copies of the files.  The files are encrypted with
the read-only key.

Untrusted nodes still participate with other peers, including other untrusted
peers, to spread the data, in a manner similar to a cloud server.


Key rotation
------------

A key for a share can be regenerated at any time.  The new key must be manually
copied to other peers.


Peer discovery
--------------

The share ID is used to find peers.  Various sources of peers are supported:

 * A central tracker
 * Manual entry by the user
 * LAN broadcast
 * A distributed hash table (DHT) amongst all participants
 * Previously valid addresses for the share

Each of these methods can be disabled by the user on a per-share basis and
implementations can elect not to implement them at their digression.

Peer addresses are represented as ASCII, with the address and port number
separated by a colon.  IPv6 addresses should be surrounded by square brackets.


Tracker protocol
----------------

The tracker is an HTTP or HTTPS service.  The main tracker service runs at
tracker.example.com (to be determined).

Software should come with the main tracker service address built-in, and may
optionally support additional tracker addresses.  Finally, the user should be
allowed to customize the tracker list.

Note that both peers should register themselves immediately with the tracker,
and re-registration should happen if the local IP address changes, or after the
TTL period has expired of the registration.

A peer ID is an 128-bit random ID that should be generated when the share is
first created, and is unique to each peer.

The share ID and listening port are used to make a GET request to the tracker
(whitespace has been added for clarity):

    http://tracker.example.com/clearskies/track?myport=30020
         &peer=e139d99b48e6d6ca033195a39eb8d9a1
         &share=00df70a2ec5a8bfe4e68d00aba75792b839ea84aa70aa1dd4dfe0e7116e253cc

The response must have the content-type of application/json and will have a
JSON body like the following (whitespace has been added for clarity):

```json
{
   "your_ip": "192.169.0.1",
   "others": ["be8b773c227f44c5110945e8254e722c@128.1.2.3:40321"],
   "ttl": 3600
}
```

The TTL is a number of seconds until the client should register again.

The "others" key contains a list of all other clients that have registered for
this share ID, with the client's "peer ID", followed by an @ sign, and then
peer's IP address.  The IP address can be an IPV4 address, or an IPV6 address
surrounded in square brackets.


Fast tracker
------------

The fast tracker service is an extension to the tracker protocol that avoids
the need for polling.  The official tracker supports this extension.

An additional parameter, "fast_track" is set to "1" and the HTTP server will
not close the connection, instead sending new JSON messages whenever a new peer
is discovered.  This method is sometimes called HTTP push or HTTP streaming.

If the tracker does not support fast-track responses, it will just send a
normal response and close the connection.

The response will contain the first JSON message as normal, with an additional
key "timeout", with an integer value.  If the connection is idle for more than
"timeout" seconds, the client should consider the connection dead and open a
new one.

Some messages will be empty and used as a ping message to keep the connection
alive.

A complete response might look like:

```json
{"success":true,"your_ip":"192.169.0.1","others":["a958e1b202a3a432caeeb66616b1305f@128.1.2.3:40321"],"ttl":3600,"timeout":120}
{}
{}
{"others":["a958e1b202a3a432caeeb66616b1305f@128.1.2.3:40321","2a3728dca353324de4d6bfbebf2128d9@99.1.2.4:41234"]}
{}
{}
```


LAN Broadcast
-------------

Peers are discovered on the LAN by a UDP broadcast to port 60106.  The
broadcast contains the following JSON payload (whitespace has been added for
legibility):

```json
{
  "name": "ClearSkiesBroadcast",
  "version": 1,
  "share": "22596363b3de40b06f981fb85d82312e8c0ed511",
  "peer": "2a3728dca353324de4d6bfbebf2128d9",
  "myport": 40121
}
```

Broadcast should be on startup, when a new share is added, when a new network
connection is detected, and every minute or so afterwards.

A separate packet should be sent for each share.


Distributed Hash Table
----------------------

Once connected to a peer, a global DHT can be used to find more peers.  The DHT
contains a mapping from share ID to peer address.

Future updates to protocol version 1 will include the DHT mechanism.


Firewall transversal
--------------------

There is no standard port number on which to listen.

Software should use UPnP to make sure that its listening port is open to the
world.

Future updates to the protocol will include a method for communicating over
UDP.


Wire protocol
-------------

The wire protocol is composed of JSON messages.  Some message types include
binary data, which is sent as a binary payload after the JSON message, but
considered part of it.  In a similar manner, some JSON messages are signed, and
the signature is included after the JSON message.

A normal message is a JSON object on a single line, followed by a newline.  No
newlines are allowed within the JSON representation.  (Note that JSON encodes
newlines in strings as "\n", so there is no need to worry about cleaning out
newlines within the object.)

The object will have a "type" key, which will identify the type of message.

For example:

```json
{"type":"foo","arg1":"Basic example message","arg2":"For great justice"}
```

To simplify implementations, messages are asynchronous (no immediate response
is required).  The protocol is almost entirely stateless.  For forward
compatibility, unsupported message types or extra keys are silently ignored.

A message with a binary data payload is also encoded in JSON, but it is
prefixed an exclamation point, followed by the size (in bytes) of the binary
payload, followed by an exclamation point, and then the JSON message as usual,
including the termination newline.  After the newline, the entire binary
payload will be sent.

For example:

```
!44!{"type":"file_data","path":"test/file.txt",...}
This is just text, but could be binary data!
```

A signed message will be prefixed with a $.  The JSON message is then sent, and
then on the next line the ECDSA signature is given, encoded with base64, and
followed up with a newline.  The base64 data should not include any newlines.

```
${"type":"foo","arg":"bar"}
MC0CFGq+pt0m53OP9eZSndaUtWwKnoJ7AhUAy6ScPi8Kbwe4SJiIvsf9DUFHWKE=
```

If a message has both a binary payload and a signature, it will start with a
dollar sign and an exclamation mark, in that order.  The signature does not
cover binary data, just the JSON message.  Here is the previous example, but
with binary data added:

```
$!39!{"type":"foo","arg":"bar"}
MC0CFGq+pt0m53OP9eZSndaUtWwKnoJ7AhUAy6ScPi8Kbwe4SJiIvsf9DUFHWKE=
Another example of possibly binary data
```


As a rule, the receiver of file data should always be the one to request it.
It should never be pushed unrequested.  This allows streaming content and do
partial copies, as will be explained in later sections.

If a message does not begin with a, '$', '{' or an '!', it should be ignored,
for forwards compatibility.


Handshake
---------

While peers are equal, we will distinguish between client and server for the
purposes of the handshake.  The server is the computer that received the
connection, but isn't necessarily the computer where the share was originally
created.

The handshake negotiates a protocol version as well as optional features, such
as compression.  When a connection is opened, the server sends a "greeting"
that lists all the protocol versions it supports, as well as an optional
feature list.  The protocol version is an integer.  The features are strings.

The current protocol version is 1.  Future improvements to version 1 will be
done in a backwards compatible way.  Version 2 will not be compatible with
version 1.

Officially supported features will be documented here.  Unofficial features
should start with a period and then a unique prefix (similar to Java).
Unofficial messages should prefix the "type" key with its unique prefix.

What follows is an example greeting.  (Newlines have been added for legibility,
but they would not be legal to send over the wire.)

```json
{
  "type": "greeting",
  "software": "bitbox 0.1",
  "protocol": [1],
  "features": ["gzip", ".com.github.jewel.messaging"]
}
```

The client will examine the greeting and decide which protocol version and
features it has in common with the server.  It will then respond with a start
message, which asks for a particular share by the share's public ID.  (See the
encryption section for an explanation of public IDs.)  Here is an example
"start" message.

```json
{
  "type": "start",
  "software": "beetlebox 0.3.7",
  "protocol": 1,
  "features": [],
  "share": "22596363b3de40b06f981fb85d82312e8c0ed511",
  "peer": "6f5902ac237024bdd0c176cb93063dc4",
  "access": "read_write"
}
```

The "access" mode is one of "read_only", "untrusted", and "read_write", and is
the highest key that the client supports for this share.

The "peer" field is the node ID explained in the tracker section.  This is used
to avoid accidental loopback.

If the server does not recognize this share, it will send back an
"no_such_share" message, and close the connection:

```json
{
  "type": "no_such_share"
}
```

Otherwise it will send back a "starttls" message:

```json
{
  "type": "starttls",
  "peer": "77b8065588ec32f95f598e93db6672ac",
  "access": "read_only"
}
```

The "access" in the starttls response is the greatest common denominator access
level between the client and server.  The corresponding key is then used by
both peers as a pre-shared key (PSK) for TLS.

The connection is encrypted with with TLS_DHE_PSK_WITH_AES_128_CBC_SHA
from [RFC 4279](http://tools.ietf.org/html/rfc4279).  Protocol version 1 only
supports this mode, not any other modes.

Once the server sends the "starttls" message, it upgrades the plain-text
connection to a TLS connection.  Likewise, when the client receives the
"starttls" message from the server, it upgrades its socket connection.

Both peers send a message through the connection divulging more information
about themselves for diagnostic purposes:

```json
{
  "type": "identity",
  "name": "Jaren's Laptop",
  "time": 1379225084
}
```

The "name" field is a human-friendly identifier for the computer.

The "time" is a unix timestamp of the current time.  This is sent because the
conflict resolution relies on an accurate time.  If the difference between the
times is too great, software may notify the user, and/or attempt to account for
the difference in conflict resolution algorithm, at the software's discretion.
The software may also refuse to participate.


File tree representation
------------------------

The entire shared directory should be scanned and a database of the tree should
be created, with the following elements for each file:

 * relative path from inside the share, with no leading slash
 * file size in bytes
 * mtime
 * unix mode bits
 * SHA256 of file contents
 * a 128-bit file ID, chosen at random
 * a 128-bit encryption key

The "mtime" is the number of seconds since the unix epoch since the file
contents were last modified.

The unix mode bits represent the user, group, and other access modes for the
file.  This is represented as an octal number, for example "0755".

The SHA256 of the file contents should be cached in a local database for
performance reasons, and should be updated with the file size or mtime changes.

The file ID is used for untrusted nodes.  It should be updated when the SHA256
changes.

The encryption key is only used when sending the file to untrusted nodes.  It
is predetermined so that all nodes agree on how to encrypt the file.  It should
be changed when the SHA256 changes.

Software running on an operating system that doesn't support all the characters
that unix supports in a filename, such as Microsoft Windows, must ensure
filenames with special characters are handled properly.  One option is to use a
reversible encoding for these characters, such as '\', '/', ':', '*', '?', '"',
'<', '>', '|'.  The suggested encoding is to use URL encoding with the percent
character, followed by two hex digits.  This should only be done for these
characters and should only be reversed for these characters.

An alternative for Windows software to path munging is to keep a record of the
original path in a database and use that path as the name when communicating
with other systems.

In a similar manner, Windows software should preserve unix mode bits.  A
read-only file in unix can be mapped to the read-only attribute in Windows.
Files that originate on Windows should be mapped to mode '0600' by default.


File tree synchronization
-------------------------

Once an encrypted connection is established, the peers usually ask for each
other's file tree listing.

A read-write peer has a definitive view of its own files.  A full listing of
its tree is called a manifest.  A read-write peer will generate a new manifest
whenever requested by a peer.  The manifest has a version, stored as a 64-bit
integer, which starts with one and should increase whenever a file changes[1].
The peer_id is also included, and this is then signed.

The contents of the shared directory can diverge between two read-write peers,
and stay diverged for a long time.  (Most notably, this happens when a peer
opts not to sync some files, as in "subtree copies", explained later.)  For
this reason, each read-write peer has its own manifest.  They do not need to
store any other manifest, but should store the latest version number that they
have successfully synced from each peer.

A read-only peer cannot change files, and may only have some of the files in
the share.  In order to prove to other read-only peers that the files it has are
genuine, it saves the read-write manifests to disk, as well as their signature.
It builds its manifest from the read-write manifest, called a read-only
manifest, which is versioned the same way as the read-write manifest.  When it
does not have all the files mentioned 



```json
{
  "type": "listing_version",
  "sha1": "e90f88f8053f4a2c0134f5fd71907fb9c12127b0",
  "last_sync": 1379220847
}
```

Note that if a peer has just started it may not have a complete picture of its
directory contents.  It will not send a listing_hash until it has completely
indexed the files it already has.

Also peers that don't want to update their copy of the listing may elect to
never send a copy of its hash.

If the hashes from both messages match, then the trees are synced and no further
synchronization is necessary.

If the hash does not match, then the behavior depends on the sync mode.  If it
is unidirectional (one peer is read-only and the other read-write), only the
server sends the "listing" message.  Otherwise both client and server send the
message simultaneously.

The "listing" message contains the full directory tree as well as list of
deleted files and the time they were deleted.  The full rationale for the need
for tracking deleted files is explained in a later section.

```json
{
  "type": "listing",
  "files": [
    {
      "path": "photos/img1.jpg",
      "sha1": "602aba74d093e7893e87c4ba4295021937087bc4",
      "mtime": 1379220393,
      "size": 2387629
    },
    {
      "path": "photos/img2.jpg",
      "sha1": "dbe2e1f6f295102b0b93d991ab4508979aa9433e",
      "mtime": 1379100421,
      "size": 6293123
    },
    {
      "path": "photos/img3.jpg",
      "dtime": 1383030498,
    }
  ]
}
```

In unidirectional mode, the file tree is now synchronized and the client is
fully informed as to which files it needs to request.

The rest of this section deals with merging the trees in bidirectional mode.

If a conflict arises, the file with the newest time wins.  If both have the
same time, the largest file wins.  If both have the same time and size, an
ASCII string comparison (strcmp) of the file hashes should be done and the file
with the lesser hash wins.

Deleted files win if the dtime (deletion time) is newer than the mtime of the
file in question.  If both times match, the deletion loses.

Note: Too increase efficiency, software may cache the correct file listing for
each known peer so that it does not need to be redetermined on subsequent
connections.

[1]: If, somehow, the 64-bit manifest version number overflows, a new peer_id
must be generated.  Since the version number only need to be incremented by one
when a file changes, this is not a terribly likely scenario.


Retrieving files
----------------

Files should be received in a random order so that if many peers are involved
with the share the files spread as quickly as possible.

If either peer wishes to retrieve the contents of a file, it sends the
following message:

```json
{
  "type": "get",
  "path": "photos/img1.jpg",
  "range": [0, 100000]
}
```

The "range" parameter is optional and allows the peer to request only certain
bytes from the file.

The other peer responds with the file data.  This will have a binary payload,
as explained earlier:

```
!100000!{"type": "file_data","path":"photos/img1.jpg", ... }
JFIF.123l;jkasaSDFasdfs...
```

A better look at the JSON payload:

```json
{
  "type": "file_data",
  "path": "photos/img1.jpg",
  "mtime": 1379223577,
  "ctime": 1379223570,
  "mode": "0600",
  "range": [0, 100000]
  "sha1": "fd5b138f7e42bd28834fb7bf35aa531fbee15d7c"
}
```

The message also contains additional metadata so that the file can be recreated
as closely as possible.  Mode bits can be translated between operating systems
at the software's discretion.

Software may choose to create read-only directories, and read-only files, in
read-only mode, so that a user doesn't make changes that will be immediately be
overwritten.

The sender should verify that the file's mtime hasn't changed since it was last
added to its database, so that the SHA1 can be updated.  For small files, the
entire file should be read into memory before being sent, so that the SHA1 can
be verified.

The receiver should write to a temporary file, perhaps with a ".!clearsky"
extension until, it has been fully received.  The SHA1 hash should be verified
before replacing the original file.  On unix systems, rename() should be used
to overwrite the original file so that it is done atomically.

A check should be done on the destination file before replacing it to see if it
has changed.  If so, the usual conflict resolution rules should be followed as
explained earlier.

Remember that the protocol is asynchronous, so software may issue multiple
"get" requests in order to receive pipelined responses.  Pipelining will cause
a large speedup when small files are involved and latency is high.

If the client wants to receive multiple files at once, 

Software may choose to respond to multiple "get" requests out of order.


File change notification
------------------------

Files should be monitored for changes on read-write shares.  This can be done
with OS hooks, or if that is not possible, the directory can be rescanned
periodically.

The hash of the file should be regenerated.  If it doesn't match, the mtime
should be checked one last time to make sure that the file hasn't been written
to again while generating the hash of the file before sending notification to
other peers.

Notification of a new or changed file looks like this:

```json
{
  "type": "replace",
  "path": "photos/img1.jpg",
  "sha1": "602aba74d093e7893e87c4ba4295021937087bc4",
  "mtime": 1379220393,
  "size": 2387629
}
```

Notification of a deleted file looks like this:

```json
{
  "type": "delete",
  "path": "photos/img3.jpg",
  "dtime": 1379224548
}
```

In addition to deleting the file, the receiving peer should add the file to its
delete list.  This is explained in greater detail in a later section.

Notification of a moved file looks like this:

```json
{
  "type": "move",
  "old_path": "photos/img5.jpg",
  "path": "photos/img4.jpg",
  "sha1": "49ef4c1f9273718b2421b2c076f09786ede5982c",
  "mtime": 1379732734,
  "size": 2259148
}
```

Moves should internally be treated as a "delete" and a "replace".  That is to
say, delete tracking should happen as usual.

It is the job of the detector to notice moved files (by SHA1 hash).  In order
to accomplish this, a rescan should look at the entire batch of changes before
sending them to the other peer.  If file change notification support by the OS
is present, the software may want to delay outgoing changes for a few seconds
to ensure that a delete wasn't really a move.

File changes notifications should be relayed to other peers once the file has
been successfully retrieved, assuming the other peers haven't already sent
notification that they have the file.


Untrusted nodes
---------------

Absent from the above description is how to communicate with an untrusted node.
Untrusted nodes are given encrypted files, which they will can then send to
other nodes with the untrusted key, read-only key, or read-write key.

A random 128-bit AES key is generated for each file, and that key is used to
encrypt the file.  The key is XOR'd with the first 128-bits of the read-only
key and stored as the first 8 bytes of the file, followed by the 16-byte IV,
followed by the encrypted data, using AES in CTR mode.

Files with the same contents should use the same encryption key, they do not
need to be encrypted twice.

Filenames are encrypted using the read-only key (FIXME how?).  The sha1 given
is the sha1 of the encrypted data, including the encryption header.  The size
given is the size of the file including the header.

"file_data" responses will always include the encryption header as the first 32
bytes of the binary payload, even for ranged requests.

FIXME: It is too much burden to calculate the sha1 for the listing, so we could
xor the sha1 with the read-only key.  However, this means that the encrypted
files cannot be verified locally, nor can a listing for an encrypted directory
be built from scratch.  I guess we could add the unencrypted sha1 (with xor) to
the encryption header and add the sha1 of the encrypted data to its footer so
that the listing could be regenerated.

A random amount of padding may be added to the end of each file before the
footer (up to 3% of its size) to avoid fingerprinting a share.

Instead, why not have the untrusted source store the manifest and then store
all the files by their SHA sum.  That deduplicates without having to store the



Deleted files
-------------

Special care is needed with deleted files to ensure that the user always gets
expected behavior.

If the care is not taken, "ghost" copies of deleted files will reappear
unexpectedly if a peer that hasn't connected in a while returns.

A list of deleted files and the time they were first noticed to be missing must
be tracked until the file stops being reported by all known peers.

An alternative method that is easier to implement is to track deleted files for
a limited amount of time, which should be at minimum 90 days.

If the exact deletion time is unknown, the oldest possible time it could have
been deleted is used.  For example, if the software is not running when the
file is deleted, the time that the last successful scan was started is used.
Additionally, if the software notices that a file is missing during a scan, it
should use the start time of the previous scan as the deletion time for that
file.


Consider the following scenarios:

There are only two peers, operating in read-write mode, peer A and B, which
both have a file.  Peer A deletes it.

Case 1: Both peers are running at that time

Peer A notices the file is missing, and notifies B.  B deletes the file.  The
next time A and B sync reconnect, A notices that B no longer has the file and
stops tracking it.

Case 2: Only A is running

Peer A notices the file is missing and saves the file in its deleted files
list.  The next time B starts, it is notified of the deleted file during file
tree synchronization and file is deleted.  The next reconnect the deleted file
can stop being tracked.

Case 3: Only B is running

When peer A is started it notices that the file was deleted.  It sets the
deletion time to the last time it had seen the file in a scan, which is
nevertheless newer than the mtime of the file.  It connects to B and the
deleted file wins in tree sync.  Things then proceed like case 1 and 2.

Case 4: Neither is running

This is identical to case 3.

+++ More peers

If there are three peers, then the delete must be tracked by all peers until
they all know about the deletion.  This avoids the following scenario:

1. Peers A, B, and C know about a file
2. Only peers A and B are running.
3. The file is deleted on A.
4. B also deletes its file.
5. A disconnects
6. C connects to B.  Since C has the file and B does not, the file reappears on B.
7. A reconnects to B.  The file reappears on A.


First sync and ghost files
--------------------------

When a key is first added to the software from a peer in read-write mode, the
software should warn the user that any directory contents may be overridden.

The existing contents of the directory should not be merged.  However, if a
file appears with an mtime that is newer than the time the share was added to
the software, it can be merged.  This allows the user to start work before the
directory has completely synced.


Keepalive
---------

A message of type "ping" should be sent every minute or so to keep connections
from being dropped:

```json
{"type":"ping"}
```


Known issues
------------

If a file is reverted by copying in an old copy from another source, the mtime
will be older on peers and so it will not be reverted.

A read-only peer has no way of verifying that a peer that claims to be
read-write is actually read-write, so a malicious peer can change the contents
of other read-only peers.  This perhaps could be solved by making the
read-write key an ECC private key, and the read-only key is the public key for
that read-write key.  This would allow for the file listing and notifications
to be signed.


Checking for missing shares
---------------------------

Each directory should have a hidden file, perhaps named ".ClearSkiesID", which
is not synced but is used to check if a drive is not mounted or a removable
drive is not present.  The file should contain the Share ID.  If this file is
not present, the software should not attempt to do any synchronization, instead
showing an error state for the share.


Archival
--------

When files are changed or deleted on one peer, the other peer may opt to save
copies in an archival directory.  If an archive is kept, it is recommended that
the SHA1 of these files still be tracked so that they can be used for
deduplication in the future.

Software may opt to limit the archive to a certain size, or offer a friendly
way to navigate through the archive.


Deduplication
-------------

The SHA1 hash should be used to avoid requesting duplicate files when already
present in the local share.  Instead, a copy of the local file should be used.


Ignoring files
--------------

Software may choose to allow the user to ignore files with certain extensions
or matching a pattern.  These files won't be sent to peers.


Base32
------

Base32 is used to encode keys for ease of manual keying.  Only uppercase A-Z
and the digits 2-9 are used.  Strings are taken five bits at a time, with
00000 being an 'A', 11001 being 'Z', 11010 being '2', and '11111' being '9'.

Human input should allow for lowercase letters, and should automatically
translate 0 as O.

FIXME: Can we refer to the relevant RFC instead of documenting this ourselves?
Also is there a documented format that includes a checksum?



Subtree copy
------------

Software may support the ability to only checkout a single subdirectory of a
share.  This does not require peer cooperation or knowledge.

In order to make this efficient, the client should keep a cached copy of the
entire tree so that the server doesn't need to send a complete copy of the tree
at connection time.


Partial copy
------------

Software may opt to implement the ability to not sync some folders or files
from the peer.

The software may let the user specify extensions not to sync, give them the
ability to match patterns, or give them a GUI to pick files or folders to
avoid.

As with partial copies, the client should keep a cached copy of the metadata
for the entire tree for efficiency reasons.


Streaming
---------

Software may optionally support not keeping a local copy of the files at all,
and instead stream the file contents live, perhaps as a FUSE filesystem,
directly integrated into a music player as a plugin, or on a mobile device.
The client can keep a small local cache of commonly used files.

It should also be possible to stream writes back to the server.  The client
would need to keep a buffer of outgoing files on local storage while waiting
for the server.

As with subtree copies and  the client should keep a cached copy of the metadata
for the entire tree for efficiency reasons.


Computer resources
------------------

This section is a set of recommendations for implementors and are not part of
the protocol.

The period between directory scans should be a multiple of the time it takes to
do rescans, for example, scans may be done every ten minutes, unless it takes
more than a minute to run a scan, in which case the scan won't be run until ten
times the time it took to run the scan.  This guarantees that scanning overhead
will be less than 10% of system load.

The software should run with low priority.  It should let the user pause sync
activity.

The software should and give battery users the option to not sync while on
battery.

Software should implement rate limiting, as sync is intended as something that
will run in the background without interfering with normal usage.

Software should also consider that many ISPs limit the amount of bandwidth that
can be consumed in a month, and support for limits can be used to ensure that
the cap isn't exceeded.

The software should debounce files changes so that it can stop syncing a file
that is changing too frequently.

The software should not lock files for reading while syncing them so that the
user can continue normal operation.

The software should give the users a rough estimate of the amount of time
remaining to sync a share so that the user can manually transfer files through
sneakernet if necessary.

Users may be relying on your software to back up important files.  You may want
to alert the user (on both computers) if the share has not synced with its peer
if has been longer than certain threshold (perhaps defaulting to a week).

While it is not the designed use case of this protocol, some shares may have
hundreds or thousands of peers.  In this case it is recommended that
connections only be made to a few dozen of them, chosen at random.