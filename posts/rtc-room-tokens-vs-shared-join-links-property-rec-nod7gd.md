# RTC Room Tokens vs Shared Join Links: Property Reconnects in 2026

TL;DR: In this RTC authorisation comparison, use scoped, short-lived room tokens when reconnecting must preserve one person's role without reopening admission. A shared join link fits low-risk, host-supervised entry, but the URL itself becomes authority. The meaningful cost and complexity difference is operational: tokens require issuance, refresh, revocation, and backfill rules; shared links shift work into host decisions, link rotation, audit review, and incident containment.

A leasing agent, resident, and maintenance contractor may enter the same call without deserving the same powers. WebRTC supplies browser real-time media primitives, but application authorization must decide who enters, publishes, reconnects, or recovers missed workflow events.

## Should RTC room tokens or a shared join link survive reconnects?

A reconnect is a continuation attempt after transport loss, laptop sleep, or a network change. Room identity and the last acknowledged event sequence may survive locally. Permission cannot be inferred from either. The server should re-check the application session, room status, participant status, expiry, and role before issuing fresh authority.

Keep the credential narrow: one room, one participant subject, an explicit role, a short expiry, and an authorization version. Do not place lease details, phone numbers, access codes, or unit notes inside it. Delivery by email or SMS proves delivery to an endpoint, not possession by the intended person; a join URL has the same limitation.

A shared link can fit a supervised open house where a host admits every entrant and the room carries little private context. It fits poorly when a contractor must not return after a work order closes, or when a resident should reconnect without being confused with the next holder of a forwarded link.

Separate three concerns. The invitation locates the workflow. The authenticated application session identifies the actor. A room credential authorizes a limited real-time action. One durable URL can combine them, but then discovery, identity, and authority share one bearer secret. This separation is the heart of room-token authorisation, not an implementation detail: the locator may remain stable while the current permission changes after a contractor is removed, a resident leaves the call, or an inspection closes.

It matters.

This generic Python decision runs before credential minting. Its monotonic version invalidates grants issued before removal or a role change.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from secrets import token_urlsafe

@dataclass(frozen=True)
class JoinGrant:
    room_id: str
    subject_id: str
    role: str
    auth_version: int
    expires_at: datetime
    capability: str

def issue_join_grant(*, room, membership, now=None):
    now = now or datetime.now(timezone.utc)
    if room.status != "open" or membership.revoked_at is not None:
        raise PermissionError("entry is no longer authorized")
    if membership.room_id != room.id:
        raise PermissionError("membership is for another room")
    if membership.role not in {"resident", "agent", "contractor"}:
        raise PermissionError("role is not allowed")
    return JoinGrant(room.id, membership.subject_id, membership.role,
                     membership.auth_version, now + timedelta(minutes=5),
                     token_urlsafe(32))
```

Five minutes is an example policy value, not a universal recommendation. Too short creates a refresh dependency during network failure; too long enlarges the reuse window. Test both against the longest credible interruption. The version closes the gap left by expiry after removal.

Short-lived does not mean harmless. Store a capability fingerprint or identifier in logs, never the secret. Visible join URLs are awkward because they can travel through history, screenshots, messages, and link previews.

## How should reconnect and backfill share authority?

Media recovery and application backfill solve different loss. A property workflow also has admission, role-change, inspection-marker, consent, and closure events. Recovered audio does not prove that the client has current authorization state.

On reconnect, the client supplies its application session and last acknowledged sequence. The server checks current membership, then returns allowed events after that sequence or a fresh snapshot if retention no longer covers the gap. Only then should it renew room authority. If the room closed offline, recovery returns a closed state.

Consider the edge case in full: a contractor joins at authorization version 7 and acknowledges event 184. Connectivity drops while the maintenance worker walks from the building corridor into an elevator. During that gap, the property manager closes the work order; removal becomes event 185 and increments the membership to version 8. The old client can still possess its earlier token and the shared join link can still be visible in a text thread, but neither should override current state. When the client requests events after 184, the server returns removal state and refuses a new grant. A shared link has no participant version unless the application adds a waiting room, single-use invitation, or authenticated exchange. Each addition moves it closer to scoped authorization and raises its real implementation complexity. This is also why media recovery cannot lead the sequence: reconnecting tracks before checking event 185 briefly restores the wrong participant.

Backfill needs bounded batches, retention, duplicate handling, and a response for unknown cursors. Event application must be idempotent. Sensitive fields should be filtered using the participant's current role, not the role held when an event was created.

## Compare work, not sticker prices

Per-minute rates do not settle this 2026 choice. Estimate operational cost from authorization states, reconnect frequency, invitation exposure, and the consequence of mistaken admission.

| Decision surface | Scoped room credential | Shared join URL |
| --- | --- | --- |
| Initial entry | Issuance service and authenticated exchange | Link possession may be enough |
| Reconnect | Re-check subject, room, role, expiry, version | Repeats bearer decision unless guarded |
| Removal | Reject refresh and increment state | Rotate link, close room, or use host admission |
| Audit | Attribute issuance and denial to a subject | Attribute entry to a link until identity is collected |
| Failure bias | Issuer outage may block valid re-entry | Broad validity may admit the wrong holder |

Neither side is free. Scoped authorization needs latency budgets, capacity planning, key rotation, and a degraded-mode policy. Shared URLs reduce issuance code but can add human work: someone must decide whether “Guest 4” is the plumber expected at Unit 12.

Measure denials by reason, issuance latency, refresh attempts, reconnect completion time, backfill gaps, unknown cursors, and host admissions. Never attach raw credentials or full invitation URLs to telemetry. Test expiry boundaries, two tabs, removal during outage, room closure, duplicate pages, forwarded invitations, and recovery beyond event retention.

## Roll out without breaking active inspections

First separate the public meeting locator from authority and add structured denial reasons. Then bind grants to subject and room, introduce authorization versions, and decouple backfill from media state. Move occupied-unit inspections, dispute calls, and time-bounded contractor access first; leave low-context, host-supervised sessions on shared links until their risk or support burden warrants change.

The decision rule is compact: choose scoped credentials when reconnect must preserve identity while reflecting revocation; choose shared links when possession plus live host admission is intentionally sufficient.

Rehearse rollback. If issuance is unavailable, private rooms should fail closed with a recoverable state, not silently fall back to a reusable link. Decide separately whether an already connected participant may continue and how long stale application state is acceptable.

## References

The primary specification defines the browser real-time communication surface; authorization, invitation handling, event backfill, and role policy remain application responsibilities.

## Sources

- https://www.w3.org/TR/webrtc/
