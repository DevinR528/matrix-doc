# MSC2753: Peeking via Sync (Take 2)

## Problem

Peeking into rooms without joining them currently relies on the deprecated v1 /initialSync and /events APIs.

This poses the following issues:

 * Servers and clients must implement two separate sets of event-syncing logic, doubling complexity.
 * Peeking a room involves setting a stream of long-lived /events requests
   going. Having multiple streams is racey, competes for resources with the
   /sync stream, and doesn't scale given each room requires a new /events
   stream.
 * v1 APIs are deprecated and not implemented on new servers.

This MSC likely obsoletes [MSC1776](https://github.com/matrix-org/matrix-doc/pull/1776).

## Proposal

We add an CS API called `/peek/{roomIdOrAlias}`, very similar to `/join/{roomIdOrAlias}`.

Calling `/peek`:
 * Resolves the given room alias to a room ID, if needed.
 * Adds the room (if permissions allow) to a new section of the /sync response
   called `peek` - but only for the device which called `/peek`.
 * The `peek` section is identical to `join`, but shows the live activity of
   rooms for which that device is peeking.

The API returns 404 on an unrecognised room ID or alias, or 403 if the room
does not allow peeking.  Rooms allow peeking if they have `history_visibility`
of `world_readable`.  N.B. `join_rules` do not affect peekability - it's
possible to have an invite-only room which joe public can still peek into, if
`history_visibility` has been set to `world_readable`.

If the `history_visibility` of a room changes to not be `world_readable`, any
peeks on the room are cancelled.

In order to clear up stale peeks if a client restarts without cleaning up
nicely, the act of calling `/sync` without a `since` token must cancel any peeks
created by that device.

Similar to `/join`, `/peek` lets you specify `server_name` querystring
parameters to specify which server(s) to try to peek into the room via (when
coupled with [MSC2444](https://github.com/matrix-org/matrix-doc/pull/2444)).

If a user subsequently `/join`s the room they're peeking, we atomically move
the room to the `join` block of the `/sync` response, allowing the client to
build on the state and history it has already received without re-sending it
down `/sync`.

To stop peeking, the user calls `/unpeek` on the room, similar to `/leave` or
`/forget`.  This returns 200 on success, 404 on unrecognised ID, or 400 if the
room was not being peeked in the first place.  Having stopped peeking, the
empty room block appears in the `leave` block of the next sync response to tell
the client that the user is no longer peeking.

The act of joining a peeked room automatically cancels any ongoing `/peeks` by
that user.

Clients should check for any irrelevant peeked rooms on launch (left over from
previous instances of the app) and explicitly `/unpeek` them to conserve
resources.

## Potential issues

It could be seen as controversial to add another new block to the `/sync`
response.  We could use the existing `join` block, but:

 * it's a misnomer (given the user hasn't joined the rooms)
 * `join` refers to rooms which the *user* is in, rather than that they are
   peeking into using a given *device*
 * we risk breaking clients who aren't aware of the new style of peeking.
 * there's already a precedent for per-device blocks in the sync response (for
   to-device messages)

It could be seen as controversial to make peeking a per-device rather than
per-user feature.  When thinking through use cases for peeking, however:

 1. Peeking a chatroom before joining it is the common case, and is definitely
    per-device - you would not expect peeked rooms to randomly pop up on other
    devices, or consume their bandwidth.
 2. [MSC1769](https://github.com/matrix-org/matrix-doc/pull/1769) (Profiles as
    rooms) is also per device: if a given client wants to look at the Member
    Info for a given user in a room, it shouldn't pollute the others with that
    data.
 3. [MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772) (Groups as
    rooms) uses room joins to indicate your own membership, and peeks to query
    the group membership of other users.  Similarly to profiles, it's not clear
    that this should be per-user rather than per-device (and worse case, it's a
    matter of effectively opting in rather than trying to filter out peeks you
    don't care about).

## Alternatives

[MSC1776](https://github.com/matrix-org/matrix-doc/pull/1776) defined an
alternative approach, where you could use filters to add peeked rooms into a
given `/sync` response as needed.  This however had some issues:

 * You can't specify what servers to peek remote rooms via.
 * You can't identify rooms via alias, only ID
 * It feels hacky to bodge peeked rooms into the `join` block of a given
   `/sync` response
 * Fiddling around with custom filters feels clunky relative to just calling a
   `/peek` endpoint similar to `/join`.

While experimenting with implementing MSC1776, I came up with this as an
alternative that empirically feels much simpler and tidier.

## Security considerations

Servers should ratelimit calls to `/peek` to stop someone DoSing the server.

Servers may stop maintaining a `/peek` if its device has not `/sync`ed
recently, and thus reclaim resources.  At the next `/sync` the server would
need to restore the `peek` and provide a gappy update.  This is most relevant
in the context of peeking into a remote room via
[MSC2444](https://github.com/matrix-org/matrix-doc/pull/2444) however.

## Unstable prefix

TBD