# Verified Video Progress Specification v0.1

**Status:** Draft  
**Baseline date:** 2026-10-07

## 1. Goal

This document defines a common baseline so Moodle, Rhymix, WordPress, and GnuBoard5 bridges produce equivalent verified watch-progress semantics for equivalent input.

It does not standardize platform storage APIs. It standardizes the meaning of the result.

## 2. Default Values

- heartbeat: 5 seconds
- seek grace/tolerance: 3 seconds
- completion threshold: 80%
- per-user progress is normally stored only for authenticated users

Platforms may make these configurable, but tests MUST declare the values used.

## 3. Server Authority

A server MUST NOT trust client current_time as accumulated watch time.

At minimum the server maintains the semantic equivalents of:

- watched_ranges
- watched_seconds
- frontier
- last_position
- last_heartbeat
- completed

## 4. Heartbeat Validation

If no prior progress record exists, the baseline behavior is to establish state without advancing verified watch time on that first request.

If a record exists and playing=true:

~~~text
max_forward = last_position + max(1, elapsed_seconds) + seek_grace
allowed_position = min(client_current_time, max_forward)
~~~

A new watched range is added only when allowed_position is greater than last_position.

If playing=false, a paused heartbeat MUST NOT advance verified watch time.

~~~text
allowed_position = min(client_current_time, frontier + seek_grace)
~~~

A paused request may move the stored position back into an already verified area, but it does not add a new watched range.

## 5. Range Merge

Every range is clamped to the media duration.

Malformed ranges and ranges with end <= start are discarded.

After sorting, overlapping or adjacent ranges are merged. The current bridge baseline uses:

~~~text
next.start <= current.end + 1
~~~

watched_seconds is the sum of merged range lengths, so replayed ranges are not double-counted.

## 6. Frontier

frontier is the farthest position continuously verified from media time zero.

Continuity stops when the next range starts beyond frontier + 1.

A client may use frontier to immediately restore an excessive forward seek, but the server response remains authoritative.

## 7. Completion

~~~text
percentage = watched_seconds / duration * 100
completed = percentage >= completion_threshold
~~~

A zero or invalid duration MUST NOT produce completion.

## 8. Seek Handling

The client SHOULD restore a seek that moves beyond frontier + tolerance.

Client-side seek blocking is not a security boundary. Implementations MUST assume developer tools or direct API calls and MUST apply the forward-progress limit on the server.

## 9. Playback Events

Recommended client behavior:

- play: synchronize state and start heartbeat
- pause: stop heartbeat and synchronize
- ended: stop heartbeat and perform final synchronization
- pagehide/beforeunload: use keepalive or a best-effort final send where supported

The server must remain safe even when client events are lost.

## 10. Media Replacement

When the media content for an activity changes, old progress must not silently carry over to unrelated content. An implementation needs an explicit reset or migration policy.

The Moodle Simple Video Tracker reference implementation resets progress when video content changes and may recalculate when only duration changes.

Bridge implementations SHOULD preserve the same meaning.

## 11. Concurrency

Multiple heartbeats for the same learner and lecture may overlap.

Implementations SHOULD use transactions, row locks, application locks, or an equivalent serialization mechanism to avoid lost updates.

Concurrency between media replacement and learner heartbeats must also be considered.

## 12. Shared Tests

Reuse [progress-v0.1.json](../test-vectors/progress-v0.1.json) in each bridge test suite.

Storage representation may differ by platform, but watched_seconds, frontier, position, completed, and percentage semantics should match.
