---
title: "Content Stitching for Media over QUIC"
abbrev: "MoQ Content Stitching"
category: info

docname: draft-englishm-moq-content-stitching-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - MoQ
 - composition
 - caching
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "englishm/draft-englishm-moq-content-stitching"
  latest: "https://englishm.github.io/draft-englishm-moq-content-stitching/draft-englishm-moq-content-stitching.html"

author:
 -
    fullname: Mike English
    organization: Cloudflare
    email: ietf@englishm.net

normative:
  MoQT: I-D.ietf-moq-transport

informative:
  RFC9775:

...

--- abstract

Media over QUIC Transport (MOQT) requires that an Object is always the
same bytes, no matter who asks for it or where it is retrieved from.
That property is what makes MOQT content cacheable by relays. It also
means that when different subscribers need to receive different
content, the straightforward approach is to publish a separate track for
each of them, even when most of what those tracks carry is the same.
This document describes content stitching: building the tracks that
subscribers receive out of Objects taken from other tracks, so shared
Objects can be cached once while each audience still receives a track
of its own. It names the pieces involved, describes how they fit
together, works through some examples, and draws a line between a
stitcher and a relay. It does not define a format or mechanism for
stitching.


--- middle

# Introduction {#intro}

MOQT {{MoQT}} identifies every Object by its Track Namespace, Track
Name, Group ID, and Object ID, and requires that an Object "must be an
identical sequence of bytes regardless of how or where it is retrieved"
({{Section 2.1 of MoQT}}). It also requires that two tracks in the
same scope containing different data have different names
({{Section 2.5 of MoQT}}). This is what lets a relay answer any
subscriber from its cache without knowing anything about who that
subscriber is.

There are a number of applications where subscribers receiving what is
nominally the same content need to receive different bytes, for some or
all of the time. Within MOQT's rules, each distinct version has to be
its own track. If each version is published as a complete track in its
own right, caches end up holding every version in full. With N versions,
the cache footprint is roughly N times larger, even when only a small
fraction of the content actually differs between them. Ideally, cache
cost would scale with how much content is actually unique, not with the
number of versions.

HTTP-based adaptive streaming deployments often handle this in the
manifest, pointing different clients at different lists of the same
cacheable resources. MOQT has no manifest in the delivery path, and
unlike HTTP request headers, MOQT request parameters do not change what
a track contains. So the same problem has to be addressed at the level
of tracks and Objects.

This document describes one way to do that, which we call content
stitching. The goal is to describe building blocks: the entities
involved, what each is responsible for, and the properties that follow
from putting them together. It intentionally does not define how a
stitcher is configured, how signals are carried, or a format for
describing compositions. Some of that might be worth standardizing
later, if there turns out to be a real need for different
implementations to interoperate on it.


# Terminology {#terms}

This document uses the terms Application, Track, Group, Object, Full
Track Name, Original Publisher, End Subscriber, and Relay as defined in
{{MoQT}}.

Composed track:
: A track produced by a stitcher from Objects taken from one or more
  ingredient tracks. A composed track is an ordinary MOQT track with its
  own Full Track Name. Subscribers do not need to know that a track they
  receive is composed.

Ingredient track:
: A track that a stitcher reads from in order to produce composed
  tracks. An ingredient track might feed many composed tracks or only
  one. Ingredient tracks can also be generated dynamically. In most
  deployments, ingredient tracks are not made available to End
  Subscribers directly.

Recipe:
: The rule a stitcher follows to build a composed track from its
  ingredient tracks. Recipes are defined by the Application. A recipe
  might be a fixed pattern known in advance, or it might respond to
  signals.

Signal:
: Information that tells a stitcher to change how a composed track
  relates to its ingredient tracks. Signals might be carried in
  ingredient tracks (for example, as Object Properties), in a separate
  track, or outside of MOQT entirely. This document does not specify
  how.

Stitcher:
: An Application that applies recipes to ingredient tracks to produce
  composed tracks.

Audience:
: The set of End Subscribers that receive a given composed track. An
  audience can be large, or it can be a single End Subscriber.


# Stitchers and Relays {#not-relays}

A stitcher is the Original Publisher of the composed tracks it produces.
From the point of view of each ingredient track, it is an End
Subscriber: it does not forward that track to anyone. {{MoQT}} defines a
Relay as an entity that is neither the Original Publisher nor the End
Subscriber of the tracks it handles, so a stitcher is not a Relay. It is
an Application.

In practice, stitchers will often be deployed alongside relays,
possibly in the same process, because that is where the shared
ingredient Objects are already cached. That is a reasonable operational
choice, but it is only that. Nothing in this document is an expectation
of MOQT relays, and a relay that does no stitching at all is a
perfectly good relay.

Downstream of the stitcher, a composed track is just a track, and
relays handle it like any other.


# Composition Mechanics {#mechanics}

## Identity and Numbering {#numbering}

Each composed track has its own Full Track Name, so MOQT's immutability
rule applies to it exactly as it does to any other track. As the
Original Publisher, the stitcher assigns the composed track's Group IDs,
Subgroup IDs, Object IDs, Publisher Priority, and Object Properties.
None of these need to match the ingredient Objects the composed Objects
were taken from. A stitcher might keep an ingredient's Group IDs when
its ingredients are aligned by Group ID, or it might number the composed
track's Groups independently. Either way, the composed track's Group
IDs are expected to increase with time, the same as any other track's
({{Section 2.3.1 of MoQT}}).

Immutability has a consequence for how recipes run. Once a stitcher has
published an Object in a composed track, that Object is fixed. A recipe
can change which ingredient future Objects come from, but it can never
revise what has already been published, so every decision about a given
composed Object has to be made before that Object is published.

## Where the Uniqueness Lives {#uniqueness}

It can help to ask where the uniqueness of a composed track actually
lives. In some cases it lives in the Objects: some ingredient tracks are
used by only one audience, and their Objects appear in no other composed
track. In other cases every ingredient track is shared, and the
uniqueness lives entirely in the recipe, in which shared Objects each
audience gets and in what order.

The cache cost of stitching follows from that answer. Unique Objects
cost storage. Unique recipes mostly do not.

## Transitions Between Ingredients {#transitions}

Taking an Object out of one track and putting it in another does not
make it decodable. Some media has dependencies between Objects. A frame
that depends on a reference frame the composed track never carried is
useless, and so is a frame without the initialization data its decoder
needs. Codec configuration and frame rate also have to line up across
the ingredients that a composed track moves between.

The simplest approach is for a composed track to move between
ingredient tracks only at Group boundaries, since MOQT Groups are
already meant to be independently useful join points
({{Section 2.3 of MoQT}}). This works best when ingredients are prepared
so that their Group boundaries line up with each other. Where Objects
within a Group do not depend on each other, a recipe can work at a finer
granularity and select individual Objects.

Timing is a separate problem. An ingredient carries its own timeline,
which does not necessarily match the composed track's. If timing
information lives outside the payload (in Object Properties, for
example), the stitcher can set it for the composed track as part of
being its Original Publisher. If timing lives inside the payload, then
either the stitcher has to rewrite payloads, which costs processing and
breaks any end-to-end authentication of those payloads, or the
ingredients have to be prepared so that their timelines already agree.

## Object Properties {#properties}

Because a composed Object belongs to a different track than the
ingredient Object it came from, the stitcher decides which Object
Properties it carries. Carrying everything over is rarely what is
wanted. Properties that only make sense in the context of an ingredient
generally should not appear in composed tracks, and that includes
signals. In some applications it matters that End Subscribers cannot
tell from the composed track where transitions happen.

## Signals and Lead Time {#lead-time}

When a recipe responds to signals, how much warning a signal gives
matters. A signal that arrives well before the affected Objects need to
be published leaves time for the stitcher to choose ingredients,
subscribe to them, and receive their Objects. A signal that arrives too
late cannot change anything, since the affected composed Objects may
already be published. Recipes that respond to signals generally need a
fallback for that case, such as continuing with the current ingredient.

## Placement and Cost {#placement}

Ingredient Objects shared across many composed tracks only need to be
cached once where the stitcher runs. Downstream of the stitcher, though,
each composed track is a separate track, and a relay that caches a
composed track caches all of it. Most of the benefit of stitching
therefore comes from placing stitchers as close to End Subscribers as is
practical. The larger an audience, the more a downstream cache of its
composed track is worth. For an audience of one, it is worth very
little.

## Creating Composed Tracks {#creation}

A stitcher does not have to create every composed track in advance.
When the set of possible audiences is large, it can be more practical to
create a composed track when it is first subscribed to. How a subscriber
learns which composed track to subscribe to, and how that is authorized,
is up to the Application.


# Examples {#examples}

These examples are illustrative only. In them, a Full Track Name is
written as its Track Namespace Fields separated by slashes, then a
colon, then the Track Name; for example, `example/in:main` is the track
named `main` in the namespace with fields `example` and `in`. A
Location is written as {Group ID, Object ID}. This notation is specific
to this document.

## Substituting a Span of Groups {#ex-span}

There are two audiences, A and B. The ingredient track `example/in:main`
is shared by both. The ingredient track `example/in:alt-1` is used only
by audience A, and has three Groups prepared to fit in place of Groups
104 through 106 of `main`. Every Group in both ingredients has four
Objects.

Audience A receives `example/out/7f3a:v` and audience B receives
`example/out/c21e:v`. The stitcher numbers composed Groups starting
from 0.

| Composed Group | example/out/7f3a:v (A) | example/out/c21e:v (B) |
|:--:|:--|:--|
| 0 | `main` Group 100 | `main` Group 100 |
| 1 | `main` Group 101 | `main` Group 101 |
| 2 | `main` Group 102 | `main` Group 102 |
| 3 | `main` Group 103 | `main` Group 103 |
| 4 | `alt-1` Group 0 | `main` Group 104 |
| 5 | `alt-1` Group 1 | `main` Group 105 |
| 6 | `alt-1` Group 2 | `main` Group 106 |
| 7 | `main` Group 107 | `main` Group 107 |
{: title="Composed Groups and the ingredient Groups they carry"}

Within each Group, composed Objects 0 through 3 carry the payloads of
Objects 0 through 3 of the corresponding ingredient Group. So {4, 2} in A's composed track
has the same payload as {0, 2} in `alt-1`, and {4, 2} in B's composed
track has the same payload as {104, 2} in `main`.

The stitcher's cache holds 8 Groups of `main` and 3 Groups of `alt-1`,
no matter how many End Subscribers are in each audience. A relay
downstream that caches both composed tracks holds 16 Groups.

B's composed track carries exactly what `main` carries, under a
different name and numbering. Giving B `main` directly would avoid that,
but it would expose an ingredient track to End Subscribers, and it would
make visible which End Subscribers are getting something different.

## Responding to a Signal {#ex-signal}

Object {204, 0} of `example/in:main` carries an Application-defined
Object Property indicating that Groups 206 and 207 are to be replaced.
The composed track `example/out/9b04:v` numbers its Groups as the
`main` Group ID minus 100. When the signal arrives, the stitcher has
published composed Groups up through 104. Its recipe selects
`example/in:alt-9` for this audience, and the stitcher subscribes to it
and receives its Groups 0 and 1.

| Composed Group | Taken from | Notes |
|:--:|:--|:--|
| 104 | `main` Group 204 | signal Property on {204, 0} not carried |
| 105 | `main` Group 205 | |
| 106 | `alt-9` Group 0 | |
| 107 | `alt-9` Group 1 | |
| 108 | `main` Group 208 | |
{: title="Composition in response to a signal"}

Composed Object {104, 0} does not carry the signal Property, so End
Subscribers cannot see the transition coming.

Suppose instead that Group 0 of `alt-9` is not available by the time
composed Group 106 has to be published. A recipe with a fallback might
publish `main` Group 206 as composed Group 106. Once it has, composed
Group 106 is fixed. If Group 0 of `alt-9` arrives a moment later, it
cannot replace what has already been published. Whether it makes sense
to use Group 1 of `alt-9` on its own for composed Group 107 is up to the
recipe.

## Composing Individual Objects {#ex-objects}

Where Objects within a Group do not depend on each other, a recipe can
select individual Objects rather than whole Groups. Group 12 of
`example/in:shared` has Objects 0 through 3. The ingredient track
`example/in:u-91f0` is used for a single End Subscriber, and its Object
{3, 0} is to appear after Object 1 of shared Group 12.

| Composed Location | Taken from |
|:--:|:--|
| {12, 0} | `shared` {12, 0} |
| {12, 1} | `shared` {12, 1} |
| {12, 2} | `u-91f0` {3, 0} |
| {12, 3} | `shared` {12, 2} |
| {12, 4} | `shared` {12, 3} |
{: title="Object-level composition"}

The stitcher assigns the composed Object IDs, so Object {12, 2} of
`shared` appears in the composed track as {12, 3}. The audience is a
single End Subscriber, so the composed track as a whole is unique to
them, but only one of its Objects is.


# Security Considerations {#security}

If End Subscribers can subscribe to ingredient tracks directly, they can
receive content their recipe would not have given them, or bypass the
composition entirely. Where that matters, deployments need access
control on ingredient tracks, not just on composed tracks.

Signals change what End Subscribers receive. A forged or replayed signal
could cause a stitcher to publish content that was never intended.
Stitchers need some way to trust the signals they act on.

{{Section 2.1 of MoQT}} makes the Original Publisher solely responsible
for an Object's payload, including any end-to-end encryption or
authentication. A stitcher is the Original Publisher of its composed
tracks, but the payloads it publishes were produced by the publishers of
its ingredients. Where payloads are encrypted or authenticated end to
end, keys and authentication need to cover every ingredient a composed
track may draw from, and End Subscribers need to accept payloads from
all of them. A stitcher that cannot read payloads can still compose
tracks, as long as its recipes only depend on information it can see.

Composed tracks are state on the stitcher. A stitcher that creates
composed tracks in response to subscriptions lets subscribers cause it
to create state, so it needs to authorize a subscription before creating
anything for it.


# Privacy Considerations {#privacy}

A composed track exists because some set of End Subscribers was treated
differently from others, so its Full Track Name can reveal something
about its audience. If a name encodes audience membership in a readable
way, anything that can see the name, including relays and logs, learns
it too. Opaque composed track names avoid this.

A composed track for a single End Subscriber is linkable to that End
Subscriber. In some applications that linkability is the point. In
others it is a side effect worth minimizing, for example by using such
tracks only when the recipe actually calls for them.


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Use of Generative AI
{:numbered="false"}

Anthropic's Claude
was used to assist with organizing source materials,
structuring the document, and drafting text.
All AI-generated content was reviewed and approved by the author.
This disclosure follows the guidance in {{RFC9775}},
applied voluntarily to this IETF document.
