---
title: Implementing message retention policies in Matrix
description: As part of my work in the Matrix core team, I found myself implementing an actively requested feature, i.e. support for message retention policies, in Synapse (the reference Matrix homeserver implementation). In this post, I give you a peek at what the feature does, how it works, and how it's currently implemented in Synapse.
tags: [ 'matrix', 'synapse', 'free', 'software', 'decentralisation', 'federation', 'communications', 'retention', 'policy', 'message' ]
publishDate: 2021-01-11T00:00:00+02:00
draft: false
thumbnail: /matrix-retention-policies/msc1763.png
---

Hello there, long time no see!

As you may know, I'm currently working at [Element](https://element.io), as part of the backend team working on [Matrix](https://matrix.org)'s server-side implementations. The main project involved in this work, at least from my side of things, is [Synapse](https://github.com/matrix-org/synapse), the reference Matrix homeserver implementation. If you don't know what a homeserver is, you may want to check out my post [Enter the Matrix](https://brendan.abolivier.bzh/enter-the-matrix/), in which I give an extensive introduction to Matrix. Parts of this blog post need a basic comprehension of how Matrix works, so if you don't already have that, you'll probably want to give that post an eye before continuing this one.

In this context, in 2019, I got to [implement](https://github.com/matrix-org/synapse/pull/5815) in Synapse a feature that had been actively requested by the Matrix community for a while now: message retention policy support. It allows any server or room admin to define a period of time after which a message gets hidden from clients and eventually deleted.

This feature is fairly complex to implement and document, due to different moving parts needing to interact with one another. The [current documentation](https://github.com/matrix-org/synapse/blob/develop/docs/message_retention_policies.md) is a good place to start, especially if you're mainly interested in knowing how to configure a retention policy on your server or in your room. But I thought it might be interesting to get a bit deeper into its implementation and explain some design choices and shortcomings.

In other words, the goal of this post isn't to explain how to get set up with this feature, but to be a technical breakdown explaining the internal design of this implementation. For instance, I'm not going to dump and explain the necessary bits of configuration right away but rather try to explain is as much detail as I can what they do. If you're a complete stranger to this feature you might want to have a very quick skim through the documentation I've linked to in the previous paragraph, though I'm going to repeat a bunch of what's being said there.

So here I go.

# A quick look at the spec

Message retention policies are defined in Matrix in [MSC1763](https://github.com/matrix-org/matrix-doc/blob/matthew/msc1763/proposals/1763-configurable-retention-periods.md) (an MSC being a proposal to the Matrix specification, not unlike RFCs), which defines them as state events (of type `m.room.retention`) sent to the room the administrator (or moderator) wants to regulate. Therefore, a policy is scoped to a room.

The content for this state event looks like this:

```json
{
    "max_lifetime": 2419200000,
    "min_lifetime": 86400000
}
```

This event has two properties: `max_lifetime` and `min_lifetime`. Their values are time durations expressed in milliseconds. The combination of these properties define the lifetime of an non-state event after it's been sent to a room:

* `max_lifetime` defines how long after the message is sent the homeservers participating in the room can keep it around. In the example above (where its value is `2419200000`), it means a homeserver __must__ delete events sent to the room at the latest 28 days after they've been sent (though we'll see later that, in the current implementation in Synapse, it can vary a bit around that value).
* `min_lifetime` defines the minimum amount of time after the event is sent during which homeservers should store the event and not delete it. This is particularly helpful for e.g. governmental organisations that are required (through laws like the Freedom Of Information Act) to keep a record of messages sent, or for moderation purposes. In the example above (where its value is `86400000`), it means a homeserver __should__ store events at least 24h after they've been sent.

In other words, these parameters are limits to the total lifetime of an event. If a message retention policy has no `min_lifetime` then the homeserver is free to delete events as soon as it wants, and if it's got no `max_lifetime` then the homeserver is free to never delete any event. It's then up to the homeserver to decide when to delete events using these constraints.

# Processing policies

So let's have a look at how this is implemented in Synapse. Before going any further on the implementation side of things, I need to mention that this implementation is still currently considered experimental, and is disabled by default in any new install. We'll see a bit later in this post how to enable and tweak it using Synapse's configuration file. Currently, the MSC still needs some clarification and discussion, and so future iterations on it might cause the implementation to change. This is why we're not declaring it stable as is and enabling it by default yet. On top of that, its main goal is to perform bulk deletion of data, so we want to make extra sure it's done right before flicking the switch in order to prevent any irreversible breakage.

First, let's see how Synapse keeps track of retention policies for all the rooms it's in. That bit is rather simple: every time a state event is sent to a room with the type `m.room.retention`, Synapse will [insert](https://github.com/matrix-org/synapse/blob/70259d8c8c0be71d3588a16211ccb42af87235da/synapse/storage/databases/main/events.py#L1270-L1301) a row into its `room_retention` database table. This row will include some data about the policy, including the `min_lifetime` and `max_lifetime` properties. Note that both those properties are `NULL`able, allowing for either (or both) property to be omitted (we'll see later what Synapse does in this case). As far as Synapse is concerned, a room with a retention policy with an empty content (`{}`) is the same thing as a room with no retention policy.

Now that Synapse knows the retention policy for each room it's in, it can apply it to the events in the room. It's worth noting that a current point of discussion on the MSC, and somewhere the implementation differs from the spec, is that the MSC mentions events should be purged according to the retention policy of the room as it was when the event was sent. Synapse, on the other hand, will purge events based on the retention policy the room currently has, because it creates less technical complications, provides better performances and seems to better fit the expectations of users.

# Configuring message retention policy support

Let's take a quick break from the technical breakdown to clarify a thing or two. In the next few sections, I'll take a look at different parts of Synapse's implementation of message retention policy support. I'll also explain how they tie into the feature's configuration.

Message retention policy support can be enabled and tweaked in Synapse's YAML [configuration file](https://github.com/matrix-org/synapse/blob/70259d8c8c0be71d3588a16211ccb42af87235da/docs/sample_config.yaml). All of the configuration related to this feature can be found [in the `retention` section](https://github.com/matrix-org/synapse/blob/70259d8c8c0be71d3588a16211ccb42af87235da/docs/sample_config.yaml#L369-L436). I'm not going to get into too much detail about what the different sub-sections and settings mean and how they're used, as the rest of this post already covers this. One thing I will mention here, however, is that you can enable the feature by setting `enable` to `true` in this section.

Note that if this setting is missing or set to `false`, Synapse will still store new message retention policies. It will not, however, delete any event from the database.

Now let's see how Synapse deletes messages when this feature is enabled.

# Synapse and its many jobs

Because it would be too expensive and complex to track the lifetime of each event individually, and set a timer to purge them from the database, Synapse purges events by running regularly scheduled jobs. Doing so also allows merging code paths with another feature, which is the [purge history admin API](https://github.com/matrix-org/synapse/blob/master/docs/admin_api/purge_history_api.rst). The frequency and scope of these jobs are [defined in Synapse's configuration](https://github.com/matrix-org/synapse/blob/70259d8c8c0be71d3588a16211ccb42af87235da/docs/sample_config.yaml#L402-L436) as such:

```yaml
purge_jobs:
  - longest_max_lifetime: 3d
    interval: 12h
  - shortest_max_lifetime: 3d
    interval: 1d
```

This example describes two purge jobs. This definition includes a frequency, defined by the required `interval` setting, which defines the time between two instantiations of a job.

In the example above, Synapse will run a job every 12 hours purging expired events in rooms which retention policy feature a `max_lifetime` with a value of 3 days or less; as well as another job every day purging expired events in rooms which retention policy feature a `max_lifetime` with a value of more than 3 days (note that `longest_max_lifetime` is inclusive but `shortest_max_lifetime` isn't).

The reason Synapse allows multiple jobs to be defined in the same configuration is that all rooms don't have the same sensitivity with regards to their retention policy. Some might have their policy dictate that no event can live longer than a day, whereas others might only require events to be purged after a year.

Another thing to keep in mind is that running a purge job might be an expensive task to run as it can involve deleting a lot of data, so you don't want to run a job every minute purging all expired events in all rooms.

Defining multiple jobs allows making sure rooms get processed according to the sensitivity of their policy, as well as ensuring the best performance possible. You could see it as sharding (or partitioning) the load of purging history of rooms across all of the jobs based on a room's retention policy. This also allows sufficient flexibility in the configuration.

It's worth noting that both `shortest_max_lifetime` and `longest_max_lifetime` are optional here; and here as well lack of one limit simply means there's no limit applied in that direction. For instance, the following example defines a purge job without any limit on the interval of `max_lifetime` values it handles:

```yaml
purge_jobs:
  - interval: 12h
```

It is also possible to bind a job to a precise scope by specifying both settings:

```yaml
purge_jobs:
  - interval: 12h
    shortest_max_lifetime: 6h
    longest_max_lifetime: 1d
```

Heads up that it's highly recommended to configure a job with an open limit on each side of the range of `max_lifetime` values - this can be either a job with no limit (as shown above) or two jobs, each limiting in one direction.

But wait, you'll then say, isn't it bad if Synapse might delete expired messages hours, possibly days, after they've expired? To which I'd answer that yes, probably, however this is mitigated by another feature of this implementation: when a message expires, [Synapse will stop sending it to clients](https://github.com/matrix-org/synapse/blob/70259d8c8c0be71d3588a16211ccb42af87235da/synapse/visibility.py#L136-L143). This means that, even though Synapse might not purge events immediately when they expire, it will prevent clients from seeing it. Note that clients that have already downloaded and stored the event might continue to show it, unless they themselves implement support for message retention policies, and no homeserver can do anything about that.

The same mechanism applies if an expired event is sent to Synapse by another homeserver through federation, for example when [backfilling](https://matrix.org/docs/spec/server_server/latest#backfilling-and-retrieving-missing-events), if the remote server doesn't implement this feature (or doesn't have it enabled). In this case this feature, when enabled, will prevent this event from reaching clients, letting it sit in its database until the next run of a relevant purge job clears it up.

# Under the hood

Right, now we understand how to configure a purge job, let's see how it actually works. I'm not going to go into detail on the specific SQL deletion that happens, the main reason being this code was already there when implementing the feature, as part of the [purge history admin API](https://github.com/matrix-org/synapse/blob/master/docs/admin_api/purge_history_api.rst), and the purge jobs just hook into it.

Quick heads up, in this section I'll be moving from linking to Synapse's code to sharing snippets of the code directly here, because I believe it's nicer to understand what's going on. For reference, all of these snippets will come from [Synapse's pagination handler](https://github.com/matrix-org/synapse/blob/70259d8c8c0be71d3588a16211ccb42af87235da/synapse/handlers/pagination.py) and should be located in the top half of the file, if you want to contextualise them.

When Synapse starts, it will start a looping call for each purge job configured:

```python
# Run the purge jobs described in the configuration file.
for job in hs.config.retention_purge_jobs:
    logger.info("Setting up purge job with config: %s", job)

    self.clock.looping_call(
        run_as_background_process,
        job["interval"],
        "purge_history_for_rooms_in_range",
        self.purge_history_for_rooms_in_range,
        job["shortest_max_lifetime"],
        job["longest_max_lifetime"],
    )
```

If you're running Synapse in a worker setup that isn't configured to run background tasks on the main process, these purge jobs will be scheduled on whichever worker `run_background_tasks_on` is pointing to in your configuration file.

As expected, we can see that each job is run in a looping call. As its name might suggest, in Synapse, a looping call is a function that is called in an infinite loop (asynchronously) with a given interval between two calls to that function. In this instance, we can see that for each looping call we use the configured interval for the associated purge job configuration. We also provide the function with the purge job's range.

## Specifying a default retention policy

Now let's see what a purge job actually does. First it retrieves the rooms it will be purging, and their retention policies, from Synapse's database:

```python
# We want the storage layer to include rooms with no retention policy in its
# return value only if a default retention policy is defined in the server's
# configuration and that policy's 'max_lifetime' is either lower (or equal) than
# max_ms or higher than min_ms (or both).
if self._retention_default_max_lifetime is not None:
    include_null = True

    if min_ms is not None and min_ms >= self._retention_default_max_lifetime:
        # The default max_lifetime is lower than (or equal to) min_ms.
        include_null = False

    if max_ms is not None and max_ms < self._retention_default_max_lifetime:
        # The default max_lifetime is higher than max_ms.
        include_null = False
else:
    include_null = False

logger.info(
    "[purge] Running purge job for %s < max_lifetime <= %s (include NULLs = %s)",
    min_ms,
    max_ms,
    include_null,
)

rooms = await self.store.get_rooms_for_retention_period_in_range(
    min_ms, max_ms, include_null
)
```

The first part of this code doesn't actually do any retrieval from the database, but figures out what to retrieve. More specifically, it figures out whether this purge job needs to process rooms with no retention policy stored as well as rooms which retention policies are within the range of this job. A room with no retention policy will still be stored in the `room_retention` table, with a `NULL` retention policy, hence the name of the boolean variable indicating whether we need to retrieve these as well (`include_null`).

The reason we might want to process these rooms is because it is possible in Synapse to define a default policy for all rooms that don't have one in their state, using the following configuration in the `retention` section of Synapse's configuration file:

```yaml
default_policy:
  min_lifetime: 1d
  max_lifetime: 1y
```

This example is equivalent to adding this `m.room.retention` event into the state of any room that doesn't already specify a retention policy:

```json
{
    "min_lifetime": 86400000,
    "max_lifetime": 31557600000
}
```

If a room already specifies a retention policy, Synapse will use that policy and not the default one.

Note that there is one difference with actually inserting the policy into the room's state, it's that this default policy will only be applied on your homeserver, so if another homeserver is in the room they won't necessarily apply the same policy. However, as we've seen before, if another homeserver sends yours events that should be deleted according to your default policy, Synapse will hide it for clients and just wait for the relevant purge job to delete it.

This check is actually quite simple: we only need to process rooms without a retention policy if a default server-wide retention policy has been configured (because it then applies to any room without a policy). On top of that, we check whether this default policy specifies a value for `max_lifetime` that's within the job's range.

We then call `get_rooms_for_retention_period_in_range` on Synapse's storage layer, which returns a dictionary associating a room's ID with its retention policy, for example:

```python
{
    "!someroom:example.com": {
        "max_lifetime": 2419200000,
        "min_lifetime": 86400000
    }
}
```

Once we have these rooms, we iterate over them.

## Capping the policy

We first check if there isn't a purge in progress in that room, and if so skip it to prevent any damage due to a conflict:

```python
if room_id in self._purges_in_progress_by_room:
    logger.warning(
        "[purge] not purging room %s as there's an ongoing purge running"
        " for this room",
        room_id,
    )
    continue
```

We then proceed to cap the room's retention policy. This is done by another bit of configuration in the `retention` section of Synapse's configuration file:

```
allowed_lifetime_min: 1d
allowed_lifetime_max: 1y
```

The rationale on capping a room's policy is that your homeserver might run under different requirements with regards to data retention than the other homeservers in the room. You might want to make sure you keep messages long enough for e.g. audit or other legal purposes, or you might want to make sure you don't keep them too long so they don't take up too much space on your disk and/or for privacy-related reasons. Whatever your reason is for doing so, Synapse allows you to override a room's retention policy before purging it to ensure it doesn't purge what you want to keep around, or it purges what you don't want around anymore.

Both `allowed_lifetime_min` and `allowed_lifetime_max` are optional configuration parameters. They apply to both `min_lifetime` and `max_lifetime`, however when running a purge job, we only care about the policy's `max_lifetime` value, so that's the one Synapse will cap if necessary:

```python
# If max_lifetime is None, it means that the room has no retention policy.
# Given we only retrieve such rooms when there's a default retention policy
# defined in the server's configuration, we can safely assume that's the
# case and use it for this room.
max_lifetime = (
    retention_policy["max_lifetime"] or self._retention_default_max_lifetime
)

# Cap the effective max_lifetime to be within the range allowed in the
# config.
# We do this in two steps:
#   1. Make sure it's higher or equal to the minimum allowed value, and if
#      it's not replace it with that value. This is because the server
#      operator can be required to not delete information before a given
#      time, e.g. to comply with freedom of information laws.
#   2. Make sure the resulting value is lower or equal to the maximum allowed
#      value, and if it's not replace it with that value. This is because the
#      server operator can be required to delete any data after a specific
#      amount of time.
if self._retention_allowed_lifetime_min is not None:
    max_lifetime = max(self._retention_allowed_lifetime_min, max_lifetime)

if self._retention_allowed_lifetime_max is not None:
    max_lifetime = min(max_lifetime, self._retention_allowed_lifetime_max)

logger.debug("[purge] max_lifetime for room %s: %s", room_id, max_lifetime)
```

We first figure out what the effective value for `max_lifetime` is in the room; it's either the value from the room's policy, or from the homeserver's default policy if no specific policy is defined for this room.

Then we:

1. take the maximum value between `allowed_lifetime_min` and `max_lifetime`, so we use the effective value if it's within the allowed range, and the minimum allowed value if it's not.
2. take the minimum value between the result of step 1 and the maximum allowed value, so we use the value from step 1 if it's within the allowed range, and the maximum allowed value if it's not.

That way we ensure that, if the effective value of `max_lifetime` is within the allowed range, it stays the same, otherwise it's changed to the bound it goes over.

Note that a previous implementation of this configuration refused entirely to process any incoming event that was describing a policy that wasn't abiding to this range, this is no longer the case [as of a few months ago](https://github.com/matrix-org/synapse/pull/8104), when it was changed to the implementation I've just described.

## The purge

Now's come the time to get rid of these nasty old events. Let's look at the final preparation before we do that:

```python
# Figure out what token we should start purging at.
ts = self.clock.time_msec() - max_lifetime

stream_ordering = await self.store.find_first_stream_ordering_after_ts(ts)

r = await self.store.get_room_event_before_stream_ordering(
    room_id, stream_ordering,
)
if not r:
    logger.warning(
        "[purge] purging events not possible: No event found "
        "(ts %i => stream_ordering %i)",
        ts,
        stream_ordering,
    )
    continue

(stream, topo, _event_id) = r
token = "t%d-%d" % (topo, stream)

purge_id = random_string(16)

self._purges_by_id[purge_id] = PurgeStatus()

logger.info(
    "Starting purging events in room %s (purge_id %s)" % (room_id, purge_id)
)

# We want to purge everything, including local events, and to run the purge in
# the background so that it's not blocking any other operation apart from
# other purges in the same room.
run_as_background_process(
    "_purge_history", self._purge_history, purge_id, room_id, token, True,
)
```

First, we need to figure out the timestamp we need to start purging at, which is just now minus the room's policy's `max_lifetime`, and convert that into a stream ordering.

Matrix rooms are [DAGs](https://en.wikipedia.org/wiki/Directed_acyclic_graph), which means it's not always possible to have a straight line from one point of the history to another. To address that, Synapse orders events with their unique index in its streams of incoming events, which is what we call the stream ordering of that event. Retrieving a stream ordering allows us to translate the timestamp into a location in that stream we can then use.

However, here we're retrieving the first stream ordering Synapse can find after the timestamp, but the events stream isn't scoped to the room we want to purge. This means we need to get some data on the closest event _in that room_, and we do that by calling `get_room_event_before_stream_ordering`, which will return some metadata on the event sent to that room before the given stream ordering (so the most recent event to purge from that room). This will return, beside the event's ID, its topological and stream ordering.

Now, we already know what a stream ordering is, but what about a topological ordering? Well it's roughly the same thing, except that instead of being the index of the event in Synapse's events stream, it's its index in the room's topology. For example, the first event of the room will have a topological ordering of 1, the next one 2, etc.

The main difference with a stream ordering is that a topological ordering isn't always unique because a DAG can sometimes branch. This is why we're getting both the topological ordering of the event _and_ its stream ordering, so we can tell the purge code exactly what event we want the purge to start at.

From these two integers we create a token, using the format `t[topological ordering]-[stream ordering]` (starting with `t` to make it clear which ordering comes first), and we run the `_purge_history` function into a background process, which is another way of saying we're running that function in a non-blocking way, so we can start process the next room.

Now I'm not going to go any further, because as I've already said the rest of this code was initially introduced when implementing the purge history admin API; and I didn't work much on this code except for making sure it was doing what I would expect it to do.

Though what you'll probably want to know about the code that's actually clearing off events is that it takes some precautions to make sure it doesn't completely break the rooms it's purging, namely:

* it won't delete state events to prevent the room from getting into a broken state
* it won't delete the most recent event in the room; that's, again, because a room's history is a DAG and each event needs to reference previous events (with the exception of `m.room.create`, which creates the room) - therefore if you don't have any event in the room to reference, nobody will be able to send any new event in that room (or Synapse might try to reference an older state event but then the new event will probably appear out of order on other homeservers)

However, despite not being able to delete these events, Synapse will still hide them from clients, which should be enough of a mitigation in most cases.

# A note on media

As you might have noticed, this feature only manages the retention of messages, not state events or, a more requested variant, media. Media retention is an entirely different problem (tracked [here](https://github.com/matrix-org/synapse/issues/6832)) for a few reasons. For a brief point of context, the way uploading media into a room work in Matrix, is that you first upload your media to the homeserver, then send an event into the room with data on how to reach (and possibly decrypt) the media.

The first issue with this is that in end-to-end encrypted rooms the homeserver won't be able to read the event listing the media's URL and metadata (in fact it's not even capable of distinguishing it from a text message), so it's not always possible to map a media with the room it's been sent to. On top of that, some third-party media stores such as TravisR's [matrix-media-repo](https://github.com/turt2live/matrix-media-repo) implement some deduplication logic so the same file might be used in two different rooms, which complicates things even more.

This means a separate feature needs to be implemented for media. The details and design still need to be ironed out, but it's on the team's roadmap. You might notice, however, that while this feature isn't deleting media entirely, it removes references to them from the room, which at least would still prevent members of the room from accessing them easily.

# What a journey

Message retention policies can be a super useful feature, and some bits can be a bit tricky to understand, or a bit curious in terms of design. So I hope this deep dive into how that feature works and was implemented was helpful. If it's still a hazy and unclear feel free to reach out over [Matrix](https://matrix.to/#/@brendan:abolivier.bzh) or [Twitter](https://twitter.com/BrenAbolivier)! 🙂

Note that this isn't a technical documentation on how to use the feature, therefore I didn't specifically outlined the limitations, important bits of config, etc. related to this feature, but instead spread them through the post. If you just need to make it work and skim across its shortcomings then [the documentation](https://github.com/matrix-org/synapse/blob/develop/docs/message_retention_policies.md) is the right place to look.

I sure had fun writing this post, it was nice revisiting one of my first big features in Synapse, and it motivated me to look with fresh new eyes into all of the implementation's details (and even find a few bugs), which was welcome 😀

See you in the next post! 🙂
