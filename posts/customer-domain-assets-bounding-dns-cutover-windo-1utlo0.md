# Customer Domain Assets: Bounding DNS Cutover Windows and Signed URL Access

A media publisher serving protected assets through a customer domain has several costs to account for: asset delivery, overlapping service during a DNS change, and the evidence retained to investigate failures. Without traffic measurements, none can honestly be called the largest line on the bill. The dominant term to quantify for a cutover is the overlap window: how long the previous destination must remain usable while cached DNS answers and previously issued signed URLs still matter.

Short answer: configure the customer hostname for asset delivery, authorize each protected request independently of DNS, and keep both destinations compatible through the planned transition. For a publisher sending those links in newsletters, publish and verify SPF, DKIM, and DMARC separately. Mail authentication does not authorize asset downloads. Use propagation delay to set the earliest safe retirement point, then constrain signed URL lifetimes and diagnostic retention so that overlap has an explicit end.

This note takes a lifecycle angle: what must remain valid, for whom, and until when. The proposed example is a publication sending subscriber artwork and editorial downloads by email. All durations below are planning inputs, not observed delivery or propagation measurements.

## How long should DNS cutovers preserve customer domain access to assets?

Start with a deliberately small planning example. Suppose the old asset DNS record has a TTL of 3,600 seconds, replacement signed links live for 900 seconds, and operations reserves a 300-second margin. If the old service stops issuing links at the DNS change, a conservative planned overlap is the larger of the remaining DNS cache lifetime and the remaining accepted link lifetime, plus that margin: 3,900 seconds. If it continues issuing old-service links for another hour, that calculation no longer describes the migration. The issuance cutoff belongs in the change record.

The clock matters.

Lowering a TTL immediately before changing the destination does not remove answers already cached under the previous TTL. RFC 1034 describes TTL as the interval a resource record may be cached. Let the previous TTL age out before relying on the lower value for the next change. Even then, treat the resulting window as a planning bound under the stated assumptions, and use observations from both destinations to decide whether retirement is justified.

For budgeting, estimate overlap as service time multiplied by the incremental cost of maintaining both destinations, then add any duplicated delivery and retained diagnostic data. These are accounting categories, not a claim that all deployments pay twice for every request. Volume and the actual service contract decide which term dominates. The change that shortens this window is a coordinated issuance cutoff and compatibility policy, supported by a TTL reduction made early enough to matter.

Deleting logs sooner doesn't make cached answers disappear.

That trade-off is useful during a newsletter launch. A fast hostname change may look complete from one resolver while some recipients still reach the previous endpoint. Retiring it early saves overlap time but can turn a valid subscriber link into a failed request. Keep the overlap budget visible beside the editorial launch deadline.

## Give asset routing and mail authentication separate records

For this proposed design, use `assets.publisher.example` as the customer-facing resource hostname and a separate sending subdomain for newsletter mail. A DNS record chooses where clients resolve the asset name; it does not establish the subscriber's entitlement. Configure the receiving service to recognize that hostname, provide the corresponding HTTPS identity, and map it to the correct customer before moving traffic. The exact address or alias record depends on the destination's supported configuration. Do not invent a universal record value.

The mail side has a different job. SPF evaluates authorization for the SMTP identity described in RFC 7208. DKIM signs messages using a domain and selector, with verification keys published in DNS. DMARC connects the visible From domain to aligned SPF or DKIM authentication and expresses the domain owner's handling policy. Under RFC 7489, an aligned pass through either mechanism can satisfy DMARC; requiring both to pass would misstate its authentication rule.

For the publication, keep an inventory of legitimate newsletter and transactional senders before changing policy. Subscriber login messages deserve attention here: a delayed login email and a denied artwork download can produce the same support complaint, even though their causes live in different systems. Verify the mail authentication results and the asset authorization result independently.

DKIM selector records, the SPF policy, and the DMARC policy are not substitutes for the asset routing record.

Nor does a DMARC pass establish inbox placement.

Authentication supplies evidence that receivers can use; delivery diagnosis still needs the relevant receiving result. Treat a successful DNS lookup as a narrower observation than successful mail delivery or successful subscriber access.

## Where should a signed URL stop being valid?

At a boundary the authorization service can explain. For this design, bind a signed grant to a customer, the intended hostname, one exact asset path, a permitted method, and an expiration. These are proposed application rules, not a standardized signed URL format. DNS possession alone must never grant access to another customer's files.

The hostname binding has a cost: moving an already issued link to a different hostname requires another grant. That is acceptable when customer separation is more important than transparent link portability. Conversely, changing the DNS destination while keeping the public hostname stable can preserve grants if both destinations implement the same verification contract and have the required verification material. Make that compatibility an explicit cutover requirement.

Here is a focused Python example of that contract. It uses an HMAC over deterministic serialized claims. The example assumes an HTTPS application, a secret supplied by a secret store, an authenticated issuer that has already checked entitlement, and a request layer that supplies the validated external hostname and exact resource path. It is not a complete URL parser or subscriber authorization service.

```python
import hashlib
import hmac
import json


def encode_claims(claims):
    return json.dumps(
        claims, sort_keys=True, separators=(",", ":"), ensure_ascii=True
    ).encode("ascii")


def sign_grant(secret, claims):
    return hmac.new(secret, encode_claims(claims), hashlib.sha256).hexdigest()


def permits(secret, claims, signature, *, tenant, host, path, method, now):
    expected_fields = {"tenant", "host", "path", "method", "exp"}
    if not isinstance(claims, dict) or set(claims) != expected_fields:
        return False
    if type(claims["exp"]) is not int:
        return False
    for field in ("tenant", "host", "path", "method"):
        if not isinstance(claims[field], str):
            return False
    if not isinstance(signature, str):
        return False
    if len(signature) != 64:
        return False
    if any(char not in "0123456789abcdef" for char in signature):
        return False
    if now >= claims["exp"]:
        return False
    requested = (tenant, host, path, method)
    granted = tuple(claims[key] for key in ("tenant", "host", "path", "method"))
    if requested != granted:
        return False
    expected_signature = sign_grant(secret, claims)
    return hmac.compare_digest(expected_signature, signature)
```

The caller must enforce input size limits before decoding claims and handle malformed input as a denied request. Reject duplicate claim keys during parsing. Define path normalization once, before signing and verification, and reject ambiguous encodings at the request boundary; otherwise two components may disagree about which file a grant covers. Do not derive a trusted tenant from an arbitrary incoming header. Resolve it through the configured hostname mapping.

A signature doesn't consume itself.

This proposal permits reuse until expiration, so a forwarded newsletter link can remain usable by its recipient. If the asset requires subscriber identity on every download, require a session and an entitlement check as well. One-time redemption requires server-side state. Choose deliberately: shorter link validity reduces the exposure window but increases the chance that a reader opening yesterday's email needs a fresh grant.

## Test the destination that cached DNS can still reach

Before changing the record, test both destinations with the same hostname and authorization cases. A valid grant should reach only its assigned asset. An expired grant, a changed path, a different customer hostname, and a modified expiration should be denied. Include an asset already present in cache: testing only cache misses leaves the most consequential authorization boundary unexamined.

In this proposed design, perform authorization before serving cached protected content. The first authorized request must not make that object publicly retrievable. RFC 9111 describes HTTP caching rules, but a custom signature in a query string does not establish a complete authorization policy for every intermediary. Specify cache behavior explicitly and verify it on the deployed request path. A subscriber's browser and a shared delivery cache are different retention surfaces.

Observe request outcomes by destination, tenant identifier, authorization reason, and deployment version. Keep expired-grant failures distinct from hostname mapping errors and origin failures. Otherwise a cutover dashboard can flatten several actionable problems into one error rate. On the mail side, retain aggregate authentication observations separately, so a change in DMARC results is not mistaken for an asset migration regression.

The rollback destination needs the same authorization contract. If compatibility cannot be demonstrated, delay retirement or issue replacement links through an authenticated flow. A rollback that restores DNS but rejects the links already in readers' inboxes restores only part of the service.

## Retire the old destination, then reduce the evidence

Retirement should require the planned cache interval to have elapsed, the old-link issuance cutoff to hold, and both routing and grant checks to pass. Record the previous TTL, change time, maximum remaining grant lifetime, and decision owner. Keep the previous destination available longer if the observations contradict the plan. A shorter deadline is not evidence that propagation has finished.

Then apply a separate retention decision. A proposed diagnostic policy could retain detailed request events for 7 days and aggregate counts for 30 days, subject to the publisher's actual incident-response and compliance requirements. These durations are examples, not legal requirements. Estimate retained volume as events per day multiplied by bytes per event and retained days; reducing fields or duration moves that term without changing the DNS overlap requirement.

Stop keeping full signed URLs, raw signatures, and unnecessary recipient identifiers in routine logs. Keep a request identifier and the minimum fields needed to distinguish expiry, tenant mismatch, and routing failure. Restrict access to that evidence. Email links can carry access authority, so copying them into support tools creates another place that needs protection.

There is a cost to forgetting.

After detailed events expire, the team may be able to identify an aggregate spike without reconstructing a particular subscriber's download attempt. After the old destination is retired, rollback may require rebuilding compatibility rather than changing one record. Accept those limits explicitly. The useful endpoint is a documented overlap window and a defensible evidence window, with neither kept indefinitely just because deletion feels uncomfortable.

## References

The specifications below establish DNS caching, mail authentication, message signatures, HTTP caching, and HMAC behavior. The cutover schedule, grant fields, and retention periods above are application design choices.

## Further reading

- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 9111, HTTP Caching: https://datatracker.ietf.org/doc/html/rfc9111
- RFC 2104, HMAC — Keyed-Hashing for Message Authentication: https://datatracker.ietf.org/doc/html/rfc2104
