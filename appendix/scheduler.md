[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 10. SCHEDULER-1 in OpenVoiceOS

This page maps the scheduled-events specification to the OpenVoiceOS
implementation, lists where the implementation diverges, and gives the
roadmap for closing the gap. Nothing here is normative.

## Where the pieces live

| spec concept | OpenVoiceOS location |
|---|---|
| scheduler service | `ovos_bus_client.util.scheduler.EventScheduler`, started by `ovos-core`'s skill manager as a daemon thread (`--disable-event-scheduler` turns it off) |
| owner-facing client | `ovos_bus_client.apis.events.EventSchedulerInterface`; `ovos_utils.events.EventSchedulerInterface` is a deprecated duplicate of the same class |
| skill API | `OVOSSkill.schedule_event`, `schedule_repeating_event`, `update_scheduled_event`, `cancel_scheduled_event`, `cancel_all_repeating_events`, `get_scheduled_event_status` in `ovos-workshop` |
| main consumer | `ovos-skill-alerts` (`util/alert_manager.py`), which keeps alert state itself and uses the scheduler as a one-shot clock |
| other consumers | `ovos-skill-homescreen`, `ovos-skill-laugh`, `ovos-skill-ggwave`, `ovos-skill-date-time`, `ovos-skill-audio-recording`, `ovos-skill-easter-eggs`, `ovos-ocp-audio-plugin`, `ovos-PHAL-plugin-wallpaper-manager` |
| store | `schedule.json` in the XDG **config** directory |

## Legacy wire protocol

The implementation predates the specification and uses a different
topic family and field names. A conforming scheduler for OpenVoiceOS
keeps these as aliases for one stable cycle.

| legacy topic | spec topic | notes |
|---|---|---|
| `mycroft.scheduler.schedule_event` | `ovos.scheduler.schedule` | fields `event`, `time` (epoch float), `repeat` (seconds), `data`; no response |
| `mycroft.scheduler.remove_event` | `ovos.scheduler.cancel` | field `event`; no response; no owner check |
| `mycroft.scheduler.update_event` | `ovos.scheduler.schedule` (replace) | replaces `data` only, cannot change the time; no response |
| `mycroft.scheduler.get_event` | `ovos.scheduler.get` | replies on `mycroft.event_status.callback.<name>` with a list in `data` |
| `mycroft.scheduler.list_events` | `ovos.scheduler.list` | replies with `scheduled_events`; no client wrapper |
| `system.clock.synced` | §7.2 | only updates a timestamp; dropped requests are not recovered |

Legacy field mapping: `event` ↔ `event`; `time` ↔ `at` (epoch float ↔
RFC 3339 instant); `repeat` ↔ `every.seconds`; `data` ↔ `data`. The
legacy protocol has no `id` (the event name doubles as identity), no
`owner` (the `<skill_id>:` prefix of the event name is the only
namespace, and it is applied by the client, not checked by the
service), no `local` recurrence, no `until`/`count`, no misfire
policy, and no `ephemeral` flag.

## Divergences from SCHEDULER-1

| spec clause | implementation today |
|---|---|
| §4 every request answered | `schedule_event`, `remove_event`, `update_event` are fire-and-forget; validation failures are logged and lost |
| §5.1 persist before answering | the store is written only in `shutdown()`, on a daemon thread; a kill or power loss loses every pending schedule |
| §5.4 recurring schedules restored | `shutdown()` deletes recurring entries before saving; only one-shots survive a clean restart |
| §4.3 misfires reported | one-shots that were due during downtime are dropped on load with no message |
| §5.2 replace on identity | one-shots with the same name are appended; a skill that re-creates its schedules on boot doubles them |
| §3.2 instants with offset | wire time is a bare epoch float; a naive datetime gets the configured zone in the client, then loses it |
| §3.4.2 wall-clock recurrence | not available; skills compute the next occurrence themselves and re-schedule after each fire |
| §3.4.1 anchored periods | after a missed period the next occurrence is re-anchored on the current time; the phase drifts |
| §6.2 owner scoping | `remove_event` cancels any name from any caller; the client refuses to send a cancel unless it holds a local handler, so restored schedules cannot be cancelled |
| §7.1 clock steps | wall clock only; suspend, NTP steps and manual changes misfire or storm |
| §7.2 unsynchronized clock | requests made while the clock reads before a fixed date are refused and dropped, not deferred |
| §8 client library | workshop's `schedule_event`/`schedule_repeating_event` drop an explicitly passed `context` when no message is on the stack (operator precedence); `get_scheduled_event_status` reads a list-shaped response and raises a bare `Exception` on timeout; no `list` wrapper |
| registry | `ovos-pydantic-models` declares `name`/`when`/`repeat_interval` where the wire carries `event`/`time`/`repeat` |

Consequences seen in the field: alarms fire at the old offset after a
daylight-saving change, alarms set before a restart fire twice or not
at all, and repeating housekeeping events silently stop after a core
restart until the skill re-registers.

## Context replay and the satellite alarm

Storing the request's whole `context` and replaying it at fire time is
what the implementation already does, and §4.2 prescribes it, so this
is one behaviour the adapter does not have to change: the legacy
service and a conforming one agree on the context of a fire, and the
legacy store already round-trips it.

It is also what makes a hub serving satellites work. A satellite asks
for an alarm; the layer-2 substrate stamps the satellite's identifier
into the request's routing keys and the satellite's session into its
session carrier; the hub's scheduler stores that context with the
record. When the alarm comes due, the fire carries the same context,
the router reads the routing keys, and the alarm rings on the
satellite that asked for it. Without the replay it would ring on the
hub.

Records written before the store carried a context restore without
one and fire with the `scheduler` key alone, which is the local
behaviour — the alarm rings on the hub. Owners that re-create their
schedules on start (all of them do today) replace those records with
context-bearing ones on the first boot after the change.

## Roadmap

The work splits into what can be fixed in place without a wire change
and what needs the new protocol.

### Stage 1 — stop the bleeding (current implementation, no wire change)

1. Persist on every mutation with an atomic replace, and stop deleting
   recurring entries at shutdown; load recurring entries on start.
2. Make one-shot registration replace on name instead of append, so a
   skill re-creating its schedules on boot does not double them.
3. Report missed one-shots on load instead of dropping them (emit the
   event late when within a fixed grace period, otherwise log at
   warning level with the name and due time).
4. Defer requests made while the clock is unsynchronized instead of
   refusing them; re-evaluate on `system.clock.synced`.
5. Scope `remove_event` and `update_event` to the caller's `skill_id`
   prefix; let the client send a cancel even without a local handler.
6. Fix the workshop `context` precedence bug; add a `list` wrapper;
   move the fired-event log line to debug with the name only.
7. Delete the `ovos-utils` duplicate class in the next major of
   `ovos-utils`; correct the `ovos-pydantic-models` field names; close
   the stale bus-client issue about the update topic name.
8. In `ovos-skill-alerts`: stop re-scheduling from the cache when the
   scheduler already holds the event (rely on replace-on-name after
   item 2).

Every item is a bug fix under the current contract and ships as an
alpha of the repository it touches.

### Stage 2 — implement SCHEDULER-1

1. A new scheduler service implementing §3–§7: RFC 3339 instants,
   `every` and `local` recurrences, write-ahead store in the state
   directory, replay with misfire policy, owner scoping, clock-step
   detection with a monotonic reference, `ovos.scheduler.ready` and
   `ovos.scheduler.missed`.
2. Legacy adapter: the new service also answers the
   `mycroft.scheduler.*` topics for one stable cycle, mapping fields
   as in the table above and synthesizing `id` from the event name.
3. New client API in the bus client (`schedule`, `cancel`, `get`,
   `list`, all awaiting the response), and thin skill methods in the
   workshop that take a time-zone-aware datetime or a `local` rule and
   return the schedule `id`. The old skill methods delegate to it and
   carry a deprecation warning with the removal version.
4. Conformance suite in the test harness: one test per MUST of §9,
   including replay after kill, daylight-saving gap and overlap,
   clock step forward and backward, owner isolation, and idempotent
   re-creation.

### Stage 3 — adopt

1. `ovos-skill-alerts` moves timing to the scheduler: one schedule per
   alert with a stable `id`, `local` recurrence for weekday alarms,
   `until`/`count` for bounded repeats, `ovos.scheduler.missed` as the
   source of the missed-alerts list. The skill keeps the alert domain
   (names, sounds, DAV sync) and drops its own timing arithmetic and
   the second persistence layer for pending alerts. Alarm-like owners
   set `misfire: "skip"` so a stale occurrence is reported instead of
   ringing arbitrarily late; `late` suits reminders.
2. The remaining consumers switch to the new skill methods (all of
   them re-register on load today, so the change is mechanical).
3. Remove the legacy topics and the old skill methods in the major
   release after the one that shipped Stage 2.

### Sequencing

Stage 1 lands first because it fixes user-visible loss without
waiting for the protocol. Stage 2 can start in parallel in a new
module; it becomes the default once the conformance suite passes on
the reference deployment. Stage 3 follows one alpha after Stage 2.
