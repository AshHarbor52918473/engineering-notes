# When Missing Variables Break a Transactional Password Email Preview

Short answer: validate the template source, its declared variable contract, and the exact preview payload as three separate inputs before calling the delivery adapter. A malformed template and a missing value may look identical in a preview UI, but they belong to different owners and should produce different application error codes.

Do not send yet.

The least complex reliable design is a strict local render step shared by preview and production. It accepts a pinned template revision plus a typed payload, renders the subject, HTML, and plain-text body, and rejects incomplete output before the message reaches a transactional email API. This isolates a Node.js caller from provider-specific diagnostics: JavaScript can submit ordinary JSON, while the rendering boundary remains testable in any language.

## Why can a blank reset link have three different causes?

Start with the constraint: a password reset message is useful only if the recipient can identify the action, reach the intended HTTPS origin, and use the token within the policy enforced by the recovery service. An accepted delivery request proves none of those things. It proves only that the transport accepted its input.

Template failures happen earlier and fall into three buckets. First, the template source can be malformed: an unclosed expression, an unsupported formatter, or invalid escaping. Second, the source can be valid while its data contract is unsatisfied; `${reset_url}` exists, but the payload contains `reset_link`. Third, the render can be complete yet semantically wrong because the URL points at an untrusted host, the plain-text alternative omits the action, or preview resolves a different revision from the queued send.

Consider a concrete version-drift trace. Revision `v7` declares `reset_url`, while an independently deployed event producer still emits `reset_link`. The preview editor appears healthy because its saved fixture already uses the new name. A production job then carries the old event shape, and a permissive renderer replaces the unknown value with an empty string. Looking only at the final API request encourages the wrong questions about authentication, throttling, or recipient suppression. Instead, record the event schema version and template revision beside the correlation identifier, reproduce the render with a sanitized payload, and compare declared names before inspecting the transport. If the subject and bodies cannot be produced from those immutable inputs, stop there. Do not send a second test message. Correct either the producer mapping or the published contract, add the mismatched pair as a regression fixture, and only then exercise the delivery adapter. This sequence is deliberately dull — it turns a vague email incident into a deterministic comparison of two small sets.

Those distinctions matter operationally. A syntax error belongs to template publication. A missing variable usually belongs to the event producer or a version mismatch. An unsafe URL belongs to the recovery boundary. Folding all three into `PREVIEW_FAILED` makes an alert easy to emit and hard to act on. I prefer internal codes such as `TEMPLATE_SYNTAX_INVALID`, `TEMPLATE_VARIABLE_MISSING`, and `RESET_URL_INVALID`; these are names for the application's own contract, not claims about a provider's API.

Spam filtering and rate limits sit downstream. Repeatedly sending test messages to diagnose a deterministic binding error consumes quota, adds noise to deliverability signals, and still does not reveal why the field disappeared. Render first. Transport second.

No retry helps.

## How should a Node.js password reset email preview expose missing template variables?

Make the preview response compiler-like. On success, return the resolved revision identifier and all three rendered parts. On failure, return a stable application code, the missing field names, and the template revision; do not return a half-rendered body. A Node.js handler can map that result to a client error such as `422`, but the renderer should not depend on HTTP status codes to express its domain result.

Keep the payload boring:

```python
preview_payload = {
    "template_revision": "password-reset.en.v7",
    "locale": "en-US",
    "variables": {
        "display_name": "Avery & Co.",
        "reset_url": "https://accounts.example.test/reset?token=inert-test-token",
        "expiry_minutes": "20",
    },
}
```

The revision is not decoration. If preview silently selects the newest draft while a worker uses the currently published revision, both systems can be correct in isolation and the overall test is still false. Put the revision into the queued message envelope and preserve it through retries. The same rule applies to locale; fallback behavior must be explicit, since a fallback subject paired with a localized body can pass a superficial placeholder check.

Return variable names in diagnostics, never variable values. Reset tokens, full URLs, recipient addresses, and rendered bodies are credentials or personal data in practice, even when a logging library sees plain strings. The error response can safely say that `reset_url` is absent. It should not echo a replacement URL supplied by an upstream service.

What about extra variables? Reject them during template publication and contract tests, where drift is actionable. At runtime, rejecting every additive field can make independently deployed producers brittle, so accepting known supersets may be reasonable if the renderer consumes only a declared allowlist. I'm not sure one policy fits every event platform — schema ownership and deployment cadence decide it — but missing security-critical fields must always fail closed.

## A strict renderer makes the failure reproducible

This Python example uses only the standard library. It validates the manifest before substitution, checks the reset origin, escapes values for HTML, and renders plain text from the unescaped payload. The fixed origin is important: HTML escaping prevents markup injection, but it does not turn an attacker-controlled host into a legitimate recovery destination.

```python
from html import escape
from string import Template
from urllib.parse import urlparse


REQUIRED_VARIABLES = {"display_name", "reset_url", "expiry_minutes"}
TRUSTED_RESET_HOST = "accounts.example.test"

SUBJECT = Template("Reset your password, $display_name")
HTML_BODY = Template(
    "<p>Hello $display_name,</p>"
    '<p><a href="$reset_url">Choose a new password</a></p>'
    "<p>This link expires in $expiry_minutes minutes.</p>"
)
TEXT_BODY = Template(
    "Hello $display_name,\n\n"
    "Choose a new password: $reset_url\n"
    "This link expires in $expiry_minutes minutes."
)


class PreviewError(ValueError):
    def __init__(self, code: str, fields: set[str] | None = None) -> None:
        self.code = code
        self.fields = sorted(fields or set())
        super().__init__(code)


def render_password_reset(variables: dict[str, str]) -> dict[str, str]:
    missing = REQUIRED_VARIABLES - variables.keys()
    if missing:
        raise PreviewError("TEMPLATE_VARIABLE_MISSING", missing)

    selected = {name: variables[name] for name in REQUIRED_VARIABLES}
    parsed_url = urlparse(selected["reset_url"])
    if parsed_url.scheme != "https" or parsed_url.hostname != TRUSTED_RESET_HOST:
        raise PreviewError("RESET_URL_INVALID")

    html_values = {
        name: escape(value, quote=True) for name, value in selected.items()
    }
    try:
        rendered = {
            "subject": SUBJECT.substitute(selected),
            "html": HTML_BODY.substitute(html_values),
            "text": TEXT_BODY.substitute(selected),
        }
    except (KeyError, ValueError) as error:
        raise PreviewError("TEMPLATE_SYNTAX_INVALID") from error

    if not all(part.strip() for part in rendered.values()):
        raise PreviewError("TEMPLATE_OUTPUT_EMPTY")
    return rendered
```

There is a subtle edge case in the HTML branch. Escaping the complete URL changes `&` to `&amp;`, which is correct inside an HTML attribute; the browser decodes it when following the link. The plain-text branch must retain the original `&`. Applying one escaping function to every output context is a common way to produce a message that looks fine in HTML preview and breaks when copied from text.

Add contract fixtures for missing fields, special characters, an untrusted host, every supported locale, and both published and draft revisions. Use inert tokens. A fixture should assert the full subject and both bodies, but snapshots stored in CI should never contain a token that the recovery application would accept.

The catch is that strict rendering cannot prove inbox placement, token usability, or accessibility. It catches deterministic construction defects. Keep a smaller set of end-to-end tests for the actual recovery journey, including expired and already-used test tokens, without treating those slower tests as a substitute for the local contract.

## Put each check at the boundary that knows enough

No single component has all the evidence. The event producer understands why a reset was requested. The registry knows which revision is publishable. The renderer understands substitution and output context. The delivery adapter understands the final transport schema. The recovery application alone can decide whether a token is valid.

| Boundary | Reject here | Do not claim here |
|---|---|---|
| Event producer | Missing user or recovery intent | That a template can render |
| Template publication | Invalid syntax, undeclared fields, broken locale fixtures | That future events contain every value |
| Preview renderer | Missing values, unsafe origin, empty output | That a mailbox will accept the message |
| Delivery adapter | Invalid transport request shape | That the reset token works |
| Recovery application | Expired, reused, or invalid token | That the email was delivered |

This separation also improves observability. Count validation failures by internal error code, revision, and locale. Measure render latency separately from queue acceptance and delivery outcome. Hash or omit recipient identifiers, and never attach the full payload to an exception tracker. A dashboard that says “delivery failed” when rendering never reached the delivery adapter sends the on-call engineer toward the wrong system.

Retries need the same discipline. A missing placeholder is deterministic for a fixed revision and payload, so retrying it unchanged cannot help. Route it to a quarantined workflow that records the correlation identifier, event schema version, and template revision. Temporary transport throttling is different and may be retryable under the adapter's policy. Mixing both classes in one retry queue can turn one bad deployment into a burst of duplicate recovery mail after the template is corrected.

This design is not suitable for an exploratory template playground that intentionally accepts arbitrary key-value data. Keep that renderer isolated from production credentials and password-reset events, and label its output as non-production. For account recovery, permissive substitution is the wrong trade-off: an empty action can look polished enough to escape review.

## Roll out the contract without doubling sends

Begin with sanitized fixtures captured from the event schema, not production messages. Validate every template revision during publication, then run the new renderer in shadow mode beside the current construction path. Compare validation results and rendered hashes; never dispatch the shadow output.

Next, pin revision and locale in newly queued jobs. Block publication on syntax failures, block a send on missing recovery fields, and leave transport retry behavior unchanged. Canary one template revision, watch error codes and recovery completion as separate signals, and retain the prior published revision for rollback. Queued jobs must not switch revisions halfway through processing.

Finally, remove permissive defaults once every producer supplies the declared schema. Keep one owner for the contract. It's less exciting than chasing an API error through logs, and far more effective.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API

## Further reading

The email service documentation above is useful for separating transport concerns from local rendering. The WebOTP reference covers a different recovery channel and its security considerations; use it when comparing email links with one-time-code flows rather than assuming the channels have identical browser behavior.

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
