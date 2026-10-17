# Incident Postmortem — Duplicate Sends from a Warm-Standby Host

**What it is:** A real incident postmortem — duplicate sends to ~16,000 real recipients from a warm-standby host.
**Why it existed:** Written to document a real multi-night incident and the three layered root causes found while chasing it.
**What I owned:** The full investigation, every fix applied, and this writeup.
**Stack:** PostgreSQL, Docker, a workflow-automation engine.
**Production status:** N/A — this documents a real, resolved incident. Host/service names are generalized to avoid detailing real infrastructure; every technical fact and number is accurate.

---

**Summary:** A workflow automation engine running on a primary host and a
warm-standby host both ended up actively executing the same scheduled job,
causing a real messaging system to send the same content to the same
~16,000 real recipients two to three times across two separate nights.
Three independent root causes stacked across the investigation; each was
found through direct evidence, not assumption, and the final fix closed
the category of failure rather than just the one mechanism found first.

This is a real incident from a self-hosted multi-host automation setup —
details below are described generically (host roles rather than names) but
every technical fact, number, and fix is accurate to what actually
happened.

## Architecture context

Two hosts ran copies of the same workflow-automation engine: a **primary**
(source of truth, actively serving production) and a **standby** (kept
warm via a nightly one-way database sync, intended to be promotable during
a primary-host outage, not to run anything live on its own). Both hosts
could reach the same shared downstream delivery service over the network.

## What happened, in the order it was actually found

### 1. Discovery

Debugging an unrelated alerting issue (stale/repeating pause notifications)
turned up two Slack posts with **no corresponding execution on the
primary host at all**. Tracing this led to the real finding: the standby
host had its own, independently-active copy of the exact same scheduled
workflow, running on its own local (nightly-synced, by-then-stale)
database state — completely untouched by same-day fixes made on the
primary.

### 2. Scope check — broadened deliberately, not assumed isolated

Rather than treating this as a one-workflow problem, every workflow's
active/inactive state was compared directly between the two hosts.
**47 of 48 real production workflows were active on both simultaneously** —
essentially the entire automation fleet, not just the one that happened to
surface the bug.

Each was triaged by trigger type, since the real risk differs enormously:
- **Schedule/poll-triggered workflows** (~23 of them): genuine
  independent-firing risk — each host's own internal clock fires them
  regardless of network routing.
- **Webhook-triggered workflows** (~15 of them, including payment/donation
  intake): confirmed structurally safe — the standby host had no public
  ingress tunnel running at all, so external providers had no network path
  to reach it, verified directly (checked the real production database for
  duplicate records from the highest-stakes integration: zero found).
- **Error-triggered and sub-workflow-only nodes**: no cross-host risk by
  design.

### 3. First fix — deactivated, then discovered "deactivated" wasn't reliable

All 47 duplicate workflows were deactivated on the standby host, and the
nightly sync script was patched to re-deactivate them every night going
forward (the sync overwrites the standby's entire database from the
primary — without this, the very next sync would have silently
re-activated everything, undoing the fix).

**Checking in the next morning found the "fix" had a real gap**: one
workflow was still executing on its 2-hour schedule for hours *after* the
sync had run and the database showed it as inactive. Root cause of
*that* was never fully pinned down (the workflow engine's internal
scheduler apparently doesn't simply re-derive its registered jobs from the
database flag on every boot, the way assumed) — rather than keep chasing
an unconfirmed mechanism, the standard was raised: **verifying something
is "deactivated" by reading a database flag is not sufficient; it has to
be confirmed by checking that no new executions occur afterward.** The
practical fix: stop relying on the flag as a safety mechanism at all —
the standby's automation engine was fully stopped at the process level,
since nothing can execute if the process isn't running, regardless of
what its internal state claims.

### 4. Second incident — it came back

The following night, the standby host's automation engine was found
running again. A real duplicate/triple send had already occurred:
**6,732 distinct real people received the same message two or three
times**, confirmed via a direct database join against the actual
recipient-list membership of each resulting campaign (not estimated).

Root cause, confirmed via direct log evidence: a **separate, independent
self-healing monitoring tool** — deployed on more than one host as
identical copies, by design, for redundancy — had genuine
auto-remediation logic (not just alerting): when it saw the automation
engine's process down, it restarted it. The fix applied the night before
had only patched *one* of the monitor's deployed copies; the other kept
restarting the standby's engine on its own independent 5-minute schedule,
completely unaffected by anything done to the first copy.

**Structural root cause, not just bad timing:** the primary and standby
each kept their own separate "who has already been sent to" state,
synced only once nightly, while both could reach the *same* shared
downstream delivery service over the network. There was no real-time
shared source of truth between them — so any send the standby managed to
execute was structurally likely to overlap heavily with the primary's own
sending population, independent of exact timing.

**Fix:**
- Stopped the standby's engine again.
- Removed the container runtime's own auto-restart policy entirely (not
  just stopping the process — closing the category of "something
  restarted it and I don't know what," regardless of mechanism).
- Backfilled the "already sent to" log with everyone who'd been touched by
  the rogue sends but had no record anywhere (their sends had errored
  before reaching the normal logging step) — found via exact count
  reconciliation before/after, not assumed complete.

### 5. Third occurrence — different root cause, same symptom

It happened once more, touching ~9,044 more real people. This time:
- The first monitoring tool's logs showed two consecutive clean passes —
  **ruled out directly**, not assumed innocent.
- The nightly sync script was checked line-by-line for any restart
  command — **not present**.
- **A third, previously-unknown monitoring script was found** on the
  standby host itself — a leftover from an earlier, different operational
  role, still running locally every 5 minutes, alerting (but not
  auto-fixing) on the same container.

The exact restarting agent for this third occurrence was **never
conclusively identified** — stated honestly rather than guessed. Rather
than keep searching for a specific mechanism, the fix was made structural
instead of mechanism-specific: the container runtime's auto-restart policy
was already removed (step 4); this time the newly-found third watchdog
script was also patched, and a live Slack alert a day later caught the
actual mechanism red-handed anyway — the self-healing monitor was deployed
as **three** separate copies across the fleet (one on a third host
entirely, left over from a retired operational role), and only two of the
three had been patched across the two prior fixes. All three were
reconciled to an identical, correct configuration; verified by manually
running the patched tool and confirming the standby's process state
directly afterward, not trusting its own log output.

## Real impact

- ~16,000 combined real recipients received a duplicate (and in one case,
  triple) send across two separate nights.
- Zero data loss; the actual underlying "who was sent to" records were
  fully reconciled and backfilled in both incidents, verified by exact
  count before/after, not estimated.
- The organization's explicit, considered call on both incidents: the
  affected recipients did not need outreach or correction — a duplicate
  message is an inconvenience, not a correctness-critical failure for
  this use case. (For a transactional or billing system, this call would
  likely go the other way — flagging that distinction deliberately, since
  "redundant delivery is harmless" is a judgment about the *content*, not
  a general property of duplicate-send bugs.)

## What this incident actually demonstrates

1. **A "fix" isn't done until it's verified against the right signal.**
   Twice in the same investigation, a check that looked sufficient
   (a database flag, a monitoring tool's silence) turned out to be
   necessary but not sufficient — the real verification had to be
   "did a new execution actually happen," checked directly.
2. **Config/logic that exists in more than one place will eventually drift
   out of sync, and the second copy is often invisible until it causes
   damage.** The same self-healing tool deployed identically to three
   hosts, by design, became the exact mechanism that keep undoing a fix —
   not because redundancy is bad, but because a fix applied to one copy
   silently doesn't apply to its siblings.
3. **A structural fix beats a mechanism-specific one when the exact
   mechanism can't be pinned down.** The investigation didn't always find
   *why* something restarted — but closing "nothing can run if the
   process isn't running, and nothing can restart the process" solved the
   actual problem regardless of which of three possible agents was
   responsible on a given night.
4. **Honest, direct verification over convenient trust.** Every real
   number in this writeup — recipient counts, overlap counts, which
   executions actually sent vs. stopped at a budget gate — came from a
   direct database query or log read, including at least one case where
   an initial estimate was wrong (a naive join inflated a count) and was
   caught before being reported.
