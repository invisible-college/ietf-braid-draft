# The Braid Protocol

**Braid** is a proposal for a new version of HTTP that transforms it from a state *transfer* protocol into a state *synchronization* protocol.

## Introduction

HTTP was initially designed to transfer static pages. If a page changes, it is the client's responsibility to issue another GET request. This made sense when pages were static and written by hand. However, today's websites are generated from databases, and continuously mutate as their state changes. Now we need state synchronization, not just state transfer.

But there is no standard way to synchronize. Instead, programmers write code *around* the standards, wiring together custom protocols over WebSockets and long-polling XMLHTTPrequests with stacks of Javascript frameworks. The task of connecting a UI with data is one that every dynamic website has to do, but there is no standard way to do it.

<br><img src="https://invisible.college/braid/images/the-whole-stack.png" width=600><br><br>

This non-standard code must *synchronize* changing data with a UI, across clients and servers.

### Synchronization

*Synchronization* is a general problem that occurs whenever two or more computers or threads access the same state. Synchronization code is tricky to write. It can result in clobbers, corruptions, and race conditions.

Luckily, a set of maturing synchronization technologies (such as Operational Transform and CRDTs) can now automate and encapsulate synchronization within a library. They can synchronize arbitrary JSON data structures across an arbitrary set of computers that make arbitrary mutations, and consistently merge their edits into a valid result, without a central server, in the face of arbitrary network delays and dropouts. In other words, it is now possible to interact with state stored anywhere on a network as if it is a local variable, and program as if it is already downloaded and always up-to-date.

#### HTTP + Synchronization

Braid puts these technologies into HTTP, to standardize them. HTTP's URLs and requests provide a standard vocabulary for reading and writing state. Braid extends them with the power of Operational Transform and CRDTs.

#### New features for the web

By building support for synchronization technologies into HTTP, we solve a number of the Web's outstanding problems in caching, networking, and programming:

- Caches update automatically and instantly, because servers promise to push changes to their subscribers. This obsoletes the `cache-control` and `refresh` headers, and the `max-age` heuristic. Users never need to force-clear their cache.
- Updates go over the network as diffs, which can be much smaller than the resources they modify, making network usage much more efficient.
- *Reload* buttons in browsers become unnecessary, and can be removed for braid sites. Browsers automatically discover and display the most recent version on their own.
- Web apps require roughly 70% less code to build (in our experiments), because programmers do not need web frameworks or custom logic to wire together a stack of server-state and client-state. This work is automated by the protocol.
- Every `<textarea>` becomes a collaborative editor (like Google Docs) by default.
- Web apps get an offline mode for free. Edits from multiple clients merge automatically once they come online. Network failures recover transparently.
- Servers become optional. Many web apps can function without a server, because peers can synchronize with one another directly over the protocol.

#### Standardizing the insides of websites opens them up

This makes the internal state of websites open, distributed, and shareable. Whereas HTTP gives each *page* a URL, braid gives each *internal datum* a URL, and makes it as easy to synchronize with (and thus reuse) another site's internal data as linking to a webpage is today. You can program with another site's internal data as if it were a local variable in memory, already downloaded and always up-to-date. This enables a web of connected, linked, synchronized data to develop as an alternative to the centralized websites we see today, just as a web of pages has grown to replace the centralized networks (AOL, Compuserve, Prodigy) of the 1990s.

<br><img src="https://invisible.college/braid/images/aol-to-braid.png" width=600><br><br>

We have a working prototype of the Braid protocol, and have deployed it with production websites. This document describes the new protocol, how it differs from prior versions of HTTP, and a plan to deploy it in a backwards-compatible way, where web developers can opt into the new synchronization features without breaking the rest of the web.

## The Braid Model

The Braid Protocol supports many synchronizers. Any OT or CRDT system can translate its network traffic into Braid messages, and back again. This allows any set of OT or CRDT systems to interoperate, if they resolve ambiguities in the same way for conflicting edits to the same region of space.

Thus, the Braid protocol allows any peer to implement a different OT or CRDT algorithm, with a different set of tradeoffs in performance, as long as they agree on a model of how they *resolve* conflicting changes.

### Interactive demo

On the left are shared state as seen by four computers. You can edit each computer's state, and click the arrows to transfer messages amongst them. Hover over the braid to see how mergers are resolved.

<iframe src="/play" width="100%" height="500px" frameborder="0"></iframe>
[Try it fullscreen here.](/play)

The Braid represents spacetime of state:
 - **Versions** define points in time, irrespective of space
 - **Locations** define points of space, irrespective of time
 - **Patches** replace regions of space, across spans of time

These three aspects of spacetime are shown in three visualizations:
 - The **Braid** shows patches
 - The **Time DAG** shows the partial ordering of versions
 - The **Space DAG** shows the partial ordering of locations

#### The Braid represents all change

Each *patch* is a colored region in the braid, that replaces a region of state with something new:

<img width=180 src="https://invisible.college/braid/images/dogbirds-braid.png">

Time goes downward. Over time, you can follow locations in space, like threads of hair, as they branch, re-order, and merge back together again, just like a [braid](https://en.wikipedia.org/wiki/Braid_group).
Each patch replaces a region of space. Each peer's patches have a different color. Grey regions are mergers.

[Click here to see complex braids in action.](/demo)

#### The Time DAG represents forks and merges over time

If we ignore space, we see just the Time DAG:

<img width=100 src="https://invisible.college/braid/images/dogbirds-time.png">

When two peers edit the same version of state, time forks into a DAG. Time on a network is ambiguous—it is impossible to say which version came first. Thus, the two versions exist in parallel in the DAG.

When peers communicate their edits, time merges.

<!-- Note: every change includes mergers AND an edit patch. This way if you get N merges, you put them all together, rather than having to depend on an associative rule for choosing which go together first and second ... you only choose which N constitute a single merge on each edit, cause that's what goes across the network, cause that's all the info that HAS to go across the network. -->

#### The *Space DAG* represents conflicts as forks in space
If two peers edit the same region of space:

<img width=180 src="https://invisible.college/braid/images/dogbirds-fork.png">

then *space forks* into two:

<img width=250 src="https://invisible.college/braid/images/dogbirds-space.png">

When reading from left-to-right, space forks into a bubble of two options. It is ambiguous if one comes first. One says "dog", and in the other, "bird", but the string unambiguously ends with "s".

The space DAG represents ambiguity *without* saying how to resolve it.

Different application want to resolve conflicts in different ways. For instance, strings in a collaborative text editor will want to merge clobbering edits by inserting everything typed, and deleting everything deleted, and breaking ordering ties arbitraliy; but if two debits to a bank account balance occur in parallel, we will want to merge the debits by *adding* the differences together.

#### A *Resolver* defines how to flatten bubbles in space

For all peers to synchronize, they just need to resolve these bubbles in space in the same way. Thus, we make sure each peer uses the same *resolver* function, for the same region of space. In this case, the resolver has arbitrarily (but consistently) chosen "dog" to come before "bird":

<img width=180 src="https://invisible.college/braid/images/dogbirds-resolve.png">

In this example, the resolver simply placed both versions into the resulting string, with a particular order. But the output of a general resolver is arbitrary—a resolver could theoretically read two edited strings in english with NLP, and merge their intent, adjusting syntax and grammar along the way, as long as it did so consistently and deterministically.

In the protocol, each piece of state can specify its resolver function, like a *type*. Each field on an object can have a different resolver type. As long as all peers obey the specified resolvers, they will converge to a consistent result.

Note that in practice, a synchronizer's resolver might not actually be implemented as modular function. Real-world OT systems bake resolution into their transformation steps, and CRDTs bake it into their data structures and read functions. However, no matter the structure of their internal code, any system can *specify* its resolution strategy abstractly as a resolver, as a standard way to describe how to interoperate with it. Resolvers specify the protocol; not the implementation.

#### Braid generalizes the OT and CRDT paradigms

The Braid allows us to map Space, Time, and Patches amongst one another. From this general view, we can see the OT and CRDT paradigms as special cases:

- Operational Transform systems make *Patches* (aka *Operations*) first-class objects, and focuses on how they *Transform* when rebasing them from one branch to another.
   - Note: The transformation happens in rebasing, will be discussed later.
- CRDTs make *Locations* first-class objects, usually giving each character in a string a global, unique ID, and typically represent spatial data (like strings) using Space DAGs. Examples: WOOT, RGA, Logoot, Automerge.
- Version Control (like git) makes the *Time DAG* a first-class object. With git, instead of storing each *patch* in the braid, it stores a full version, and then computes diffs after the fact to infer patches when necessary. DARCs is more like OT; storing patches and allowing them to re-order.

In practice, an OT system satisfying the TP2 property looks a lot like a CRDT; and CRDTs with historical pruning and rebased edits look like OT systems. (In future work, we will map both systems to the standard Braid model to show where they are implementing the same algorithms.)

Because existing synchronization algorithms can be expressed in the braid model, it turns out that we can also express their messages within the braid protocol. The messages for OT and CRDT systems can be transformed into braid messages, and vice-versa. This makes the braid protocol a good candidate for a standard, like HTTP.

## The Braid Extension to HTTP

Braid is an opt-in upgrade that any developer can add to their website, without breaking the existing web. A programmer upgrades their site by including a Javascript library on their webpages and web servers, which implements the Braid protocol, and runs it over a websocket alongside existing HTTP requests.

 1. The first version of the protocol is encoded to run within a WebSocket. This lets us use it in existing browsers. A Javascript library provides a new set of APIs, giving programmers a synchronous programming environment that can be included in web apps today. These APIs provide the full functionality of the protocol, and are easy to experiment with.
 2. Once we have standardized the functionality, we can integrate at a lower level with the HTTP/3 protocol (using the QUIC transport on UDP), to get better performance.

Braid is made backwards-compatible in 3 ways:
 1. It can be inserted into existing websites with polyfill, with existing browsers and servers
 2. It gracefully degrades to serve legacy HTTP clients, just with fewer features
 3. The semantics are an extension of HTTP methods, which makes it easy for developers to learn, and to adapt their existing sites

The rest of this document explains the WebSocket protocol—as implemented in existing browsers—in more detail.


### Architectural Changes to HTTP
<!--
 [request-response](https://en.wikipedia.org/wiki/Request%E2%80%93response)
 [stateless](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Persistent_connections)
-->

Braid makes these basic changes to HTTP to add support for synchronization:

1. **Request-Response → Persistent Subscriptions.** Rather than being limited to a single request-response cycle, it maintains persistent subscriptions to changing data by default. When you GET a resource, you also subscribe to future updates, until you issue FORGET.
2. **Client/server → Symmetric.** These persistent connections allow messages in both directions. Thus, instead of being restricted to client-server relationships, relationships can be peer-to-peer. Either party can initiate a request, rather than just the "client."
3. **Opaque Resources** → **Linked JSON.** Rather than each resource consisting of a set of headers and an opaque body, we have added first-class support for structured JSON, along with a structured JSON diff/patch language, and a link format. This allows for synchronizing mutations to structured data, and also allows resources to link to other structured resources elsewhere on the network, enabling a web of data. <!-- Whereas HTTP (mostly) doesn't need to know about HTML—the resource is opaque—braid knows the structure of JSON, and how to splice patches into spatial regions. -->
4. (Optional) **Stateless → Versioned Patch History.** Whereas each message in HTTP is stateless, and can be interpreted without remembering prior messages, synchronization becomes much more powerful if each peer maintains a version history of past mutations created by it and its peers. This allows a minimal patch to be interpreted within a larger data structure. Although these features are optional, peers can implement them to gain performance enhancements and consistency guarantees. There are five such optional features:
  - Versions
  - Patches
  - Conflict Resolvers
  - Acknowledgements
  - Hints


### Persistent Subscriptions

Instead of just *getting* state at a snapshot of time, a Braid client also *subscribes* to future changes:

| HTTP method  | Braid method  | What's new |
| --------- | ------- | -----| 
| Get | Get | Also subscribes to future updates |
|  | Forget | Ends a "Get" subscription |
| Put/Post/Patch | Set | Also updates all subscribers |
| Delete | Delete | Also updates all subscribers |

This requires only minimal changes to HTTP's semantics. Programmers can thus reuse most of their knowledge from HTTP when building Braid applications. There is one new method, `forget`, which is usually issued by automatically by Braid libraries rather than programmers. Finally, Braid unifies the `put`, `post`, and `patch` methods in to a single `set` method, which is able to both create state and change state, and can optionally do so with a patch, as is explained below.

<!--
#### Persistent, Stateful Connections

HTTP methods were designed to be [stateless](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Persistent_connections), and constrained to a [single request and response](https://en.wikipedia.org/wiki/Request%E2%80%93response) initiated by a client, to a server. This 

Subscriptions require holding network connections open, and remembering the state of each connection. This foregoes HTTP's design goal of being a [stateless](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#HTTP_session_state) [request-response](https://en.wikipedia.org/wiki/Request%E2%80%93response) protocol. However, over time, HTTP itself has already added [persistent connections](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Persistent_connections) for performance and web push. 
-->

### Client-server Request-response to Peer-to-Peer Messaging
Whereas HTTP messages are either a *request* or *response* in a client-initiated connection, Braid messages can be initiated by either peer.


| HTTP 1, 2, 3 | Braid | Meaning |
| ------------- |    ------------ | ------ |
|   Get Request |      Get message | "I want this" |
|   Get Response |      Set message | "This is the current version" |
|   Put Request |       Set message | "This is the current version" |
|   Put Response |      Ack message | "I accept this version" |

<!--
|   Delete Request |       Delete message | "Delete this state" |
|   Delete Response |      Ack message | "I accept this deleted version" |
-->

Note that a Braid peer responds to a GET with a SET, whereas HTTP has an explicit *response* block.
It turns out that a GET response message has the same effect on a peer as a SET request message— both set the state on the recipient.

## Protocol Messages

Messages are sent over the websocket encoded as JSON. There are four basic messages:

```
{get: "path"}
{set: "path", val: {something: 3}}
{forget: "path"}
{delete: "path"}
```

#### Basic example
Here is an example session from a client's perspective, where it requests a resource, changes a field on it, and closes the connection:

```
1. Send: {get: "user/fred"}
2. Recv: {set: "user/fred", val: {name: "Fred", friend: {key: "user/michael"}}}
3. Send: {set: "user/fred", val: {name: "Fred Wilson", friend: {key: "user/michael"}}
4. Send: {forget: "user/fred"}

[1] The client requests to GET "/user/fred".
[2] The server responds to the GET, by issuing a SET.
[3] The client updates the current user's name to "Fred Wilson".
[4] The client is done with current_user, and unsubscribes.
```

### Versioning features

These features are all optional. If you want perfect synchronization with multiple simultaneous editors, then you want to implement them all. But you can implement just versioning, for instance, to get some nice history features, without bothering with conflict resolution. And you can implement patches to get network optimizations, without bothering with conflict resolution. There are more examples in the "graceful fallback" section of "Design Rationale".

Here is the form of messages with optional versioning features:

GET messages can specify a specific version to retrieve from history, and/or the parent versions that new versions should be specified with respect to:
```
{get: "path", version: "v2"}                   # Retrieve a specific version
{get: "path", parents: ["v1"]}                 # Subscribe to updates since v1
{get: "path", version: "v2", parents: ["v1"]}  # Retrieve v2 as patch from v1

# If `version` is not specified, get will send the latest version, and promise
# to send updates until `forget`. If `parents` is not specified, get will
# presume this version extends the latest set of versions.

```

SET messages can (1) define the IDs of new versions, (2) define which parent versions they descend from in the time DAG, and (3) express changes using a *patch*:
```
{set: "path", val: 3, version: "v2"}                     # Name this version v2
{set: "path", val: 3, version: "v2", parents: ["v1]}     # It came from v1
{set: "path", patches: ["[3] = 9"], version: "v2", parents: ["v1"]}
{set: "path", patches: ["[3] = 9"], version: "v2", parents: ["v1"], hint: [3,5,2]}

# If `version` is not specified, the recipient can create a new ID to define it.
```

Likewise, DELETE messages can specify at which version a state was deleted, and which parent versions came before it in the time DAG:
```
{delete: "path", version: "v3"}
{delete: "path", version: "v3", parents: ["v2"]}
```

Finally, ACK messages can be introduced to tell the sender when their version has been received. This information allows the sender to prune some of their history.
```
{ack: "path", version: "ax937"}
```

> Todo: specify resolver in network messages

#### Versioning example

Here is an example using the optional version, patch, and acknowledgement features:

```
1. Send: {get: "user/fred", parents: ['62347']}
2. Recv: {set: "user/fred",
          patches: ['.name[0] = "F"'],
          version: '2h38a',
          parents: ['62347']}
3. Send: {ack: "user/fred", version: "2h38a"}
4. Send: {set: "user/fred",
          patches: [".name[4:4] = \" Wilson\""],
          version: "36x02",
          parents: ["2h38a"]}
5. Recv: {ack: "user/fred", version: "36x02"}
6. Send: {forget: "user/fred"}

[1] The client requests the current version of "/current_user", 
    as a patch based on version 62347, which it has in cache.
[2] The server responds with a SET containing a patch.
[3] The client acknowledges receipt of the new version.
    This enables the server to prune its history of old versions.
[4] The client updates the current user's name to "Fred Wilson".
[5] The server acknowledges receipt of the new version.
    This enables the client to prune its history.
[6] The client is done with current_user, and unsubscribes.
```

Now let's look at how each of these features are specified:
 - Versions
 - Patches
 - Resolvers
 - Acks
 - Hints

#### Versions

Versions are represented as opaque (but unique) strings. You can represent anything in them (numbers, version vectors, hashes) by just serializing them as a string. All that matters for the braid protocol is that each version's string is unique.

```
"1"            # Version 1 (a global number)
"3_93          # Version 93 from peer 3
"x823h234"     # A hash, or random string
"[2,5,0,5]"    # A version vector
```

#### Patches

All patches are *replace* operations. Insertions are implemented as replacing a zero-length region with a non-zero-length region, and deletes replace a non-zero length region with a zero-length one.

```
'.foo[0].bar = null'       # Replace obj.foo[0].bar = null
'[3:3] = "asdf"'           # Insert the string 'asdf' at index 3 of a string. Illegal if array.
'[3] = "a"'                # Set char 3 of string to 'a'; or element 3 of array to 'a'
'[3] = "asdf"'             # Illegal if string. If array, set element 3 to 'asdf'
'[3:4] = [1, 3, 5]'        # Splice [1,3,5] into array, replacing element 3. Illegal if string.
'[4:4] = [{msg: "hi"}]'    # Insert a new message at the end of a chat
'[3:10] = ""'              # Delete characters 3-10 in string
'= false'                  # Make the whole thing false
'.foo[0].bar = undefined'  # Delete obj.foo[0].bar
```

See the section on *Portals* below to see how future work can implement *moves*, *copies*, and *nested splices* by referring to regions from past versions.

#### Conflict Resolvers

This is a function that produces a Space DAG given a set of versions to merge. It can look at any information transmitted in the braid messages, including the hint field (below).

Resolvers are specified in the protocol as arbitrary uniquely-identifiable strings. When state is served, it specifies its resolver as a string. The receiver then knows to use that resolver.

A general-purpose synchronizer could implement a large library of resolvers.

Example resolvers:
```
LWW(vid)           # Last-write-wins, sorted by version ID
insertable(vid)    # A resolver for sequences
linear-number      # Merges numerical diffs by adding them
```

Imagine that each fragment of JSON has a resolver type. Maybe you specify it in a schema:

```
Message:
{
	id: <string>            : LWW(vid)
	body: <string>          : insertable(vid)
	authors: <array>        : insertable(vid) [
		author_id: <string> : LWW(vid)
	]
	likes: <int>            : linear-number
}
```

#### Acknowledgements

These are the information needed to prune old versions.

#### Hints
Systems can put extra information in here if they want, such as something needed by the resolver.

Examples:
 - Some CRDTs natively think in terms of character IDs (aka "locations"), rather than versions + offsets. A hint can include a character ID of an edit, instead of an offset+version

The hint is not allowed to change the result, but it can be used for information that makes it easier to 

Three reasons for hints:
 - Info for arbitrary disambiguation
 - Info for a specific implementation programming complexity or efficiency
 - Semantic "command" or tag type of info to express an operation (maybe this is subset of point #2)

### Network Transports

 - WebSocket transport
   - Binary files are encoded in base64 strings.
 - Backwards-compatibility over legacy HTTP
    - If you are unable to run over a websocket, for instance because you are connecting to a legacy Rails server, you can use a reduced 
 - Future work: using QUIC

#### Compatibility

Braid is made backwards-compatible in 3 ways:
 1. It can be inserted into existing websites with polyfill, with existing browsers and servers
 2. It gracefully degrades to serve legacy HTTP clients, just with fewer features
 3. The semantics are an extension of HTTP methods, which makes it easy for developers to learn, and to adapt their existing sites
 4. The new semantics can be added gradually (versions, patches, acks, resolvers, hints)

## Design Rationale

### Time
  - We wanna make time explicit:
    - Explicit versions let you send diffs (between versions)
    - Explicit versions let you do perfect caching, and invalidate, automatically
    - Explicit versions let you catch up when you've been offline
    - Explicit versions let you disambiguate mutations that would otherwise be clobbers
    - These issues crop up as problems in SQL, Filesystem Watch, Pubsub, ...
  - Time is a DAG

### Space
  - Data structures have sequences
  - We wanna be able to insert, delete, replace.
    - Everything can be represented as replace
  - When two things edit the same region, space is a DAG

So time and space are both relative.  But any CPU or human operates with a thread.  Whenever we have multiple threads, there's no objective ordering of time and space, so to synchronize, we can use DAGs for both.

### URLs
  - URLs are arbitrary string paths
    - Can represent queries inside here, like SQL or GraphQL.  Agnostic.

### Structured JSON
  - Protocol understands structured data
    - Then it can (1) do diffs, and (2) link a reference from one state to other state, on other computer
  - We use JSON
    - Other data formats, e.g. relational tables, can be represented in JSON views
    - We can extend with more types if needed later (like dates)
    - Anything else (SQL, etc.) can be represented as JSON
    - Using a different query key can give a different view of it
  - JSON can be linked.
    - You can combine data across services
    - You can handle non-tree data
    - You can scope listeners this way

### Synchronization: Changes over Time
  - We Represent Time (versions)
    - Versions are strings.  You can capture anything that way.
  - We think in "State", rather than "Actions"
    - This is the same difference as REST vs. RPC
    - We can capture an action with a diff
    - That way there's less coupling -- even middleware can host state without knowing the semantics of the app
    - We can also attach a name to a diff, optionally, to augment it with any specific semantics necessary to an app that needs it.
  - We Represent changes with diffs
    - This both makes network messages smaller
    - And allows for merging changes
    - If two things change at the same time, we need a resolver
  - We handle conflicts with a Resolver
    - This is instead of a totally different datatype
    - Or instead of different actions, with custom merge semantics
    - This abstracts away the core intrinsic difference between synchronizers: how they resolve conflicts
      - All other resolutions are unambiguous.
  - We coordinate pruning history with acknowledgements
    - Some systems say when to delete, or infer that someone has received something
    - But in any system, the underlying need for state is to sync when someone reconnects -- the "fork point"
      - And if everyone always adds to the newest versions, everyone can delete old ones
    - So we just ack those, and know that we will only add history to the end.

### Graceful fallback examples
The full synchronization algorithms can be difficult to implement. However, Braid is still useful even if you don't want to implement that. In simple cases, you can just read and write the messages by hand, in like 4 lines of code.

Users only need the full synchronization features if they want:
 - Multiple simultaneous writers
 - To all achieve a consistent result after arbitrary edits

In practice, a lot of applications don't need this:

#### A logging system
The code for this is super simple, because:
   - Write-only doesn't need sync code
   - Read-only doesn't need sync code
   - Replicas don't care if lines get interleaved

#### A chat system
The code for this is super simple, because:
   - Replicas don't care if lines get interleaved
   - Or they can trust each other's timestamps
   - You can implement the patches by hand

#### A read-only replica
   - Doesn't need any sync code.

#### A simple, low-volume webapp
   - Most web apps fail with clobbers already, and are fine without sync code
   - But add versions and you can at least get consistency and history
   - Or add patches and get better performance for a couple places where it's necessary

## Braid Programming Model
 - Read about it at https://stateb.us/tutorial
 - UI can be reactive functions generating state
 - We can get async stuff without callbacks
 - Because we collapse time and space
   - Program as if state is already downloaded and always up-to-date
   - Because the actual protocol collapses time and space with the braid.
   - This lets you map locations in space across time, and locations in time across space
   - And your thread of execution doesn't have to care

## Reference implementation

 - Greg: once your sync9 demo is online we'll link to it here

## Extended features
### Rebasing
### Pruning intermediate details before sending
 - Can result in overlapping patches

### Storing big long history and rolling back, undo

### Branching multiple parallel histories

### Portals
 - Gives you moves, ancestral splices, and clones
 - Useful for rich text editing, without even needing a custom resolver

Imagine your rich text editor bolds a word. You might get this state transition:
```
"Hello there, dude!" ➜ ["Hello there, ", {tag: "b", _: "dude"}, "!"]
```
With this patch:

```
 = ["Hello there, ", {tag: "b", _: "dude"}, "!"]
```

But that clobbers the whole string! We'll get a conflict if we just edit "dude" to "John".

But not if we use portals. Portals are patches that reference regions from the *last* version:

```
 = [old[0:13], {tag: "b", _: old[13:17]}, old[17:18]]
```
Note the reference `old[13:17]`. This is a portal to a previous version! So then when someone replaces "dude" with "john" in that version:

```
[13:17] = "John"
```

We are able to splice it right in!

Ok, how would we delete an ancestor?

```
.grandparent = old.children[0]
```

Or insert one?

```
.grandparent = {children: old}
```

These look like move/clone/projection/wormhole/portal operations!

#### Portals!

The normal behavior for a version is that it *creates* some regions, and then just *portals* other regions from the previous version.

However, these *portals* are implicit so far... in a subsequent extension, we can make these portals explicit, and remap them amongst regions.

Like, instead of:

```
[3:5] = 'hello'
```

We could explicitly write the copies with portals:
```
[:3] = old[:3]
[3:5] = 'hello'
[5:] = old[5:]
```

Portals let us support:
 - Clones
 - Moves
  - A move is a portal + a delete
 - Ancestral splices
 - Re-orderings (perhaps useful to capture a merge linearization)

Portals also seem to introduce an ambiguity:

```
     xyz
      |
    xyzxy   # The last xy is from a portal repeating the first xy

# Now, imagine that we learn that someone edited xyz ➜ xy•z.
# What's the result?
#  Option 1:  xy•zxy•
#  Option 2:  xy•zxy
```

Another example:
```
            click here
            /        \
   click here!    click [here]  <-- wrap "here" with link
            \        /
           click [here!]    <-- Option 1
           click [here]!    <-- Option 2
```

### Flexible Network Configurations
  - Client/server
  - P2P
  - Tree (allows rebasing)

## Exciting Applications
### [Mailbus](https://wiki.invisible.college/mailbus)
   - Supports private email and blogs and SMTP and IMAP/POP all with a single protocol
   - Secure
   - Simpler
   - Spam prevention
   - Like JMAP but simpler and synchronized and easier to put on web (and probably better in other ways too)

### Make a new UI for every site — Subjective Web
   - No more censorship
   - While also filtering out stuff you don't like
   - Spam/Hate/Terrorism is subjective, which makes it dangerous for central
     org to control, because they can use it as an excuse to control you.
     - And since it benefits users, it's user's incentive to define it for
       themselves, and subjective web returns control to them.
   - Add a new feature to any site you want
   - Merge together data from multiple sites you want
   - It's as easy to program with state on some other website as a local variable

### Faster CDNs, with standard change push API
   - Can even distribute their content via open protocol
   - Thus the whole thing could potentially be open
   - You just pay people to host content nearby with high SLA

### Reputation system
   - By sharing lots of data... web of trust... can react immediately