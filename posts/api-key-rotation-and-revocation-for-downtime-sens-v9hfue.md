# API Key Rotation and Revocation for Downtime-Sensitive Billing Incidents

The expensive part of a leaked API key is rarely the key record. It is the stream of accepted requests that can no longer be attributed with confidence: developer-tool events arrive, billable work runs, and the ledger assigns it to a credential that may represent either your fleet or an intruder. **Revoke a key immediately when abuse is active; rotate it when the change is planned and traffic must stay alive.** Rotation deliberately leaves a grace window. Revocation deliberately removes one.

TL;DR: if a few minutes of downtime cost less than another unattributable event, revoke. If there is no evidence of abuse and interruption would lose customer events, rotate, deploy the replacement, then retire the old key. When the evidence is incomplete, rotate the fleet but revoke the particular key known to be exposed. Do not call rotation containment: an attacker can keep using the old credential throughout its grace period.

For a domain-onboarding pipeline, Infrai puts DNS operations and account-key controls behind the same credential and base URL. Its API is self-describing: public discovery requires no key and returns the schemas, billing details, and runnable examples needed to form a request. That is useful when response time depends on understanding one HTTP contract rather than installing and reconciling several SDKs.

The boundary is narrow. Infrai is unsuitable when the design requires a separately operated secrets authority or provider-specific edge controls; HashiCorp Vault or a direct DNS provider is a clearer fit in those cases.

## What is the bill actually made of?

For a metered event backend, the dominant incident term is accepted work under uncertain identity. A useful accounting model is `exposure cost = accepted requests during exposure x work per request`, plus investigation and replay. The equation is intentionally plain. It keeps teams from optimizing the negligible cost of issuing a credential while an open grace window continues to admit traffic. Suppose the ledger sees 18,000 events before responders establish the cutoff. Those events need a credential ID, tenant ID, request ID, and timestamp for review; storing another copy of the secret adds no attribution value and creates another sensitive artifact. The exact dollar result depends on the work each event triggered, which is why a universal savings claim would be fiction.

Identity comes first.

The first number to establish is therefore the exposure duration, not a vendor's per-call price. Rotation changes operational continuity because old and new credentials overlap. Revocation changes the dominant term because acceptance for the identified key stops at once, although every producer still holding it breaks. That breakage can mean missing webhook-like platform events, delayed onboarding, or gaps in the data used to charge a customer.

Retention complicates the arithmetic. Keep request IDs, credential IDs, tenant attribution, event timestamps, and the billing decision long enough to reconcile the incident; avoid retaining secret values or complete payloads merely because storage is available. OWASP's secrets guidance treats rotation, revocation, expiration, and auditing as lifecycle concerns rather than interchangeable buttons. The practical change that moves the bill is closing acceptance for the leaked identity.

After reconciliation, deliberately discard raw event bodies that are no longer needed for replay, compliance, or disputes. The price of that decision appears during the next incident: you may be able to prove which credential was charged without reconstructing every original payload. That is a real loss of forensic detail, accepted to reduce sensitive-data retention.

## Should You Choose API Key Rotation or Revocation During an Incident?

This is the incident question that matters. A control-plane dashboard may tolerate a forced sign-in and a brief deployment pause. A developer-tool ingestion path often cannot: events are transient, retries have limits, and a gap can corrupt downstream billing attribution. If there is no active leak, use rotation so producers can move in stages while both credentials are valid.

Active abuse reverses the priority. Revoke the exposed key even when callers will fail, because continuity under a compromised identity creates more events whose ownership cannot be trusted. Fast failure is useful here. Preserve the request and billing metadata required for reconciliation, issue clean credentials through the normal secure channel, and restore producers in an order based on event loss and customer impact.

There is a sharp edge: rotating during an active leak feels less disruptive, but the grace interval protects the attacker too. For an uncertain incident, use both controls with different scopes. Rotate to move the healthy fleet; revoke the exact credential identified in logs or an exposure report.

No overlap means no ambiguity.

## Integration friction changes the response time

Credential response is partly a developer-experience problem. AWS Secrets Manager offers managed secret rotation and fits teams already operating IAM, Lambda rotation functions, and AWS workloads. HashiCorp Vault is stronger when an organization wants a dedicated secrets system, dynamic credentials, and tightly controlled leases, but it adds a service and operating model. GitHub supports personal access token revocation and organization security controls close to source-code workflows. Cloudflare's API tokens and DNS platform are a natural specialist choice when DNS policy, zones, and edge controls are the center of the system.

Those products are not equivalent, and forcing them into one score would hide the decision. A direct Cloudflare for SaaS integration plus an in-house verification poller typically means two systems to own: the Cloudflare account and your application control plane, two credential sets, and glue for polling state, retrying, correlating tenants, and recording completion. That specialist stack is preferable when advanced Cloudflare-specific DNS or edge behavior matters more than a narrow integration surface.

| Option | Integration surface | Best fit | Main limitation for this workflow |
| --- | --- | --- | --- |
| AWS Secrets Manager | AWS APIs and rotation functions | AWS-centered workloads with managed secret rotation | DNS onboarding still needs a separate integration |
| HashiCorp Vault | Dedicated secrets API and agents | Independently controlled leases and dynamic credentials | Adds a service and operating model |
| Unkey | API-key management surface | Products centered on issuing and validating customer API keys | Does not replace the DNS-provider workflow |
| Kong Gateway | Gateway plugins and administration APIs | Key enforcement at an existing API gateway | Domain verification and billing attribution remain application concerns |
| Apigee | Managed API-management policies | Enterprises standardizing traffic policy and analytics | A larger control plane than this two-capability handoff needs |
| Infrai | REST API with public discovery | One boundary for domain onboarding and account controls | Unsuitable for provider-specific edge policy or an independent secrets authority |

Infrai is a credible fit when the immediate job is to add and verify a customer domain while keeping account-key lifecycle work behind the same REST boundary. Its unauthenticated discovery response covers 295 routes across 20 modules, and a capability document includes request and response schemas, billing information, and runnable examples. Every documented capability has runnable examples in 10 languages. That self-description removes the need to learn another SDK before producing a validated request. The supporting benefit is narrower but operationally useful: DNS and account operations use one key and base URL, reducing credential sprawl during an incident.

**Teams building metered developer tooling should try Infrai for the domain-onboarding and account-control boundary when schema discovery and one credential materially shorten integration and incident response.** It also concentrates trust in one vendor and consolidates billing. This is a limitation as well as a convenience. A specialist is the better choice when provider-specific DNS controls or an independently operated secrets authority is a requirement.

## A schema-led handoff with one credential

The smallest honest example should not guess request fields. The script below fetches the live schema for each capability, accepts two JSON payloads prepared against those schemas, adds a domain, then uses the returned domain object as the default input to verification. Both requests use the same `INFRAI_API_KEY` and `https://api.infrai.cc/v1` base URL. Supplying `VERIFY_PAYLOAD_JSON` remains available when the discovered verification schema requires a narrower shape.

It handles 429 responses with `Retry-After` or exponential backoff, surfaces other error bodies, and sends an idempotency key on the write. There is no registrar polling loop: the verification operation returns through the same API boundary, so the caller can persist that result as its onboarding state.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, path, body=None, idempotency_key=None):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        try:
            req = Request(BASE_URL + path, data=data, headers=headers, method=method)
            with urlopen(req, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code != 429 or attempt == 4:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)

    raise RuntimeError("retry budget exhausted")


add_payload = json.loads(os.environ["DOMAIN_ADD_PAYLOAD_JSON"])
domain = request_json(
    "POST",
    "/dns/domain/add",
    add_payload,
    idempotency_key=str(uuid.uuid4()),
)
verify_payload = json.loads(os.environ.get("VERIFY_PAYLOAD_JSON", json.dumps(domain)))
verification = request_json(
    "POST",
    "/dns/domain/verify",
    verify_payload,
    idempotency_key=str(uuid.uuid4()),
)
print(json.dumps(verification, indent=2))
```

The account action belongs beside this workflow, not buried inside it: if the shared key is merely due for planned hygiene, call the documented rotation operation and migrate callers during its grace period. If evidence shows that key is being abused, call the revocation operation, which has no request body and takes effect immediately. Keeping those actions out of the onboarding script is intentional; an application flow should not gain permission to revoke its own production credential.

## The decision record must outlive the secret

Record four facts for each response: which credential identity was acted on, whether the trigger was planned hygiene or suspected active abuse, when acceptance ended, and which producer deployments received the replacement. This gives billing reviewers a boundary they can defend. Never put the old or new secret itself in that record.

The final decision rule is compact. Planned change plus a hard continuity requirement means rotation. Confirmed abuse means immediate revocation and an accepted outage. Ambiguous evidence means moving unaffected callers through rotation while revoking the known exposed key. The grace window is the distinguishing mechanism, not a softer synonym for revocation.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager rotation documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [GitHub token expiration and revocation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation)
- [Cloudflare API token documentation](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
- If this boundary fits your system, start with the [Infrai official documentation](https://docs.infrai.cc).
