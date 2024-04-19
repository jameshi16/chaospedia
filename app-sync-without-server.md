---
description: This page describes an idea for synchronization without a server.
---

# 🔀 App Sync without Server

Reminder: This page is part of [.](./ "mention"), which means that it is not researched / scientific in nature.

## Impetus

I'm trying to build an app that requires me to synchronize data files across different machines, ideally without a central application server. This is ideal for a number of reasons, including ease of setup for the end-user, and the ability to use P2P file synchronization applications, like [https://syncthing.net/](https://syncthing.net/).

## Prior Research

It seems like this is not something that is normally done in the industry, as such protocols are _bound_ to have synchronization issues. Someone else might find implementing central databases a more suitable option.

I've decided to use a leader-follower star-like network as inspiration. Ideally, there would be one leader at all times, and only the leader can write to the data files. If any of the followers need to write new data, they must request to become a leader.

## Requirements

These are just some requirements I have at the top of my head:

1. The protocol should try to obtain leadership in a fair manner
2. If it is unclear who can acquire leadership, the user should be prompted. Something like: "It looks like you are the only device online! If this is true, click 'Confirm'".
3. The protocol needs to be file-sync-service-agnostic.
   1. The file-sync must guarantee that if a file is in the filesystem, it represents some complete file; i.e. if file sync is chunked, it should be written somewhere else first before being moved into the target directory.

## Assumptions

All devices in the system follows the protocol; no misuse is projected.

`THRESHOLD` >> `GRACE_SYNC_PERIOD`

## Protocol

Let's assume that there are three devices in the system. They will be nominally named A, B, and C.

Suppose all three networks come online at the same time. All three devices create their individual lock files, such as `A.lock`, `B.lock`, and `C.lock`. Each file will contain a heartbeat timestamp. It is important that each device only touches their own `.lock` file, especially if they have not been elected the **leader**.

A device is considered online if their heartbeat timestamp is within `THRESHOLD` from the current system time.

All three devices check (read-only) a common lock file, say `leader.lock` to determine who the leader is, and when the leader has last checked in. If this is more than `THRESHOLD` (in minutes), then the device will attempt to gain leadership after `GRACE_SYNC_PERIOD`. **This process includes the last leader; they cannot simply regain leadership if they were appointed as a leader in a previous session.**

At this point, there are three cases:

1. All 3 device lock files have synced; hence, all devices know the existence and online status of all 3 devices.
2. Some of the 3 device lock files have synced; there is an inaccurate view of the online status of all 3 devices, but each device knows that there is a minimum of 2 devices on the system.
3. One of the devices is effectively offline; the remaining devices does not know the existence of this offline device, and the offline device assumes it is the only one in the network.

There is no simple way to fix case 3. Hence, there must be a tool to merge conflicting files built into the protocol implementer.

In case 1 and case 2, only the device that determines it has the earlier timestamp will assume leadership. If there is an equal timestamp, then the first device with the lowest lexicographical name will be the leader. In case 1, there is no doubt who will be the leader. Assume case 2, and the timestamp order is `A < B < C`. Then any combination of these shall be possible:

* A knows A and either B or C. It concludes that A is the leader.
* B knows B and either A or C. If it knows A, then it concludes that A is the leader and does not do anything. If it knows C, then it concludes that B is the leader.
* C knows C and either A or B. In both cases, it does not do anything.

When a conclusion has been reached ballot files are created: in this case, `A.ballot` and `B.ballot`.

For `GRACE_SYNC_PERIOD`, the above protocol is repeated. If any of the device realizes that there is an older timestamp that is newer than `THRESHOLD` and has balloted, it will withdraw its ballot.

There are now two cases by the time `GRACE_SYNC_PERIOD` has ended:

1. There is only one ballot file left
2. There are still multiple ballot files

In case 2, the user will be prompted who will become the leader. An improper response from the user will lead to conflicted files.

Otherwise, the winning device will take leadership.

### Leadership

The moment a leader is appointed, it will write its identifier to `leader.lock` and update the timestamp. It will also delete any `*.ballot` files.

Only the leader can write to the data files and `leader.lock`.&#x20;

The leader must respond to requests for other devices to become a leader by reading all `.request` files. It can do this in any fashion, as long as it updates `leader.lock` with the new leader, since it is the sole decision maker.

### Followers

Followers will only read files. It cannot write to them.

If it wants to write to files, it must request for leadership by creating `.request` files. Each `.request` file should be unique to the device and contain enough information for the leader to appoint the device as the new leader.

Followers should monitor the `leader.lock` file at least once every `GRACE_SYNC_PERIOD` to update its internal state about the leader.

If the leader is dead (via heartbeat in `leader.lock`), the balloting process stated above will be reused.

## Pitfalls

* An offline device suddenly going online might break things
* At each user-facing conflict resolution step, the user has a chance of blowing up the whole thing

## Addendum 1

Instead of doing the above, it might be better to employ a CRDT system. Since file-sync is involved, we cannot rely on a singular database; ideally, the appending should create multiple files.

Allow the user to compress the size of the database by eventually writing it to an SQLite store or something. Requires the user to confirm that _the_ device is the only one on the network.
