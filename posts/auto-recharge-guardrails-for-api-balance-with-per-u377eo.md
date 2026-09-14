# Auto Recharge Guardrails for API Balance with Per-Day Ceilings and Read Back Alerts

Pick the trigger balance off your busiest day of spend rather than your average one, cap the downside with a per-day ceiling on the recharge amount, and read the configuration back from the API on a schedule so that an auto recharge policy which has quietly gone missing pages somebody on a Tuesday afternoon instead of announcing itself as rejected requests during a flash sale. Two of those three are one call each. The third is the one that saves you.

The system I have in mind is a marketplace backend that has to rotate a production API key without dropping traffic, while per-seller billing attribution stays accurate across the rotation window. Balance and auto recharge are properties of the account, while a key is nothing more than a credential. Teams conflate the two, and that is where the interesting failures come from: someone rotates a credential, assumes the money configuration rode along with it, and discovers months later that the prepaid wallet has been running on whatever was left in it.

## The invariants a key rotation must not break

Three things have to stay true from the moment you mint the replacement credential to the moment the old one is revoked. Authority is continuous, so both credentials authenticate during the overlap — most platforms express this as a grace window, and on the account API discussed below it is a `grace_hours` field on the rotate call. Funding is continuous, so the balance never crosses zero while half your fleet is still draining the old credential. And attribution is exact, so every charge incurred during the overlap maps to exactly one seller, not to "the old key" and "the new key" as if those were customers.

The failure mode I would plan for is the third one combined with a retry loop, because it is the one that costs real money rather than an apology. Picture a deploy that pushes the new credential to two of six workers, and the four stale workers start retrying a call that now looks transient to them; request volume triples, the balance falls through a trigger set at a quiet-hour average, auto recharge fires, the balance drains again inside the same hour, and it fires again. Without a ceiling, the number of top-ups in that hour is bounded by nothing except how fast your retry loop runs. A `max_per_day` of three times your normal daily burn turns that from a finance incident into a graph with an obvious spike on it.

Ceilings are cheap insurance. Set them.

The trigger deserves the same suspicion. A trigger set at one busy day of spend fires during the incident it was supposed to prevent, because the incident *is* the busy day; I would put it at roughly 1.5 days of peak burn and the recharge amount at something that buys a week, so that a payment method problem gives you days of runway rather than minutes.

## How should I configure the auto recharge trigger balance and amount, then read it back and alert?

The configure call takes the trigger balance and the recharge amount as the two required fields, and the per-day and per-month ceilings as optional ones. Optional here does not mean unimportant — they are the only part of this configuration that bounds your exposure, and leaving them out is a choice to have no upper bound at all. Send a stable `idempotency_key` so that a retried deploy re-applies the same configuration instead of racing with itself, and set the method explicitly rather than leaning on a client default.

```python
import os
import sys
import time
import requests

BASE = os.environ["INFRAI_BASE_URL"]          # account API root, set once in your deploy config
SESSION = requests.Session()
SESSION.headers.update({"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"})


def call(method, path, payload=None):
    """One HTTP call with explicit method, 429 backoff and real error surfacing."""
    for attempt in range(5):
        r = SESSION.request(method, BASE + path, json=payload, timeout=10)
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{method} {path} -> {r.status_code}: {r.text[:200]}")
        body = r.json()
        return body.get("data", body)          # native envelope keeps the payload in data
    raise RuntimeError(f"{method} {path} -> rate limited after 5 attempts")


DESIRED = {
    "trigger_balance": 400,                    # ~1.5x a peak marketplace day, not the average
    "recharge_amount": 1000,                   # roughly a week of runway
    "max_per_day": 3000,                       # the ceiling that bounds a retry storm
    "max_per_month": 20000,
    "idempotency_key": "autorecharge-marketplace-v3",
}

call("PUT", "/v1/account/autorecharge/configure", DESIRED)

live = call("GET", "/v1/account/autorecharge/get")
drift = {k: (v, live.get(k)) for k, v in DESIRED.items()
         if k != "idempotency_key" and live.get(k) != v}

if drift:
    print(f"autorecharge drift, expected vs live: {drift}", file=sys.stderr)
    sys.exit(1)                                # non-zero exit is the alert your scheduler already watches

print("autorecharge verified")
```

Run that same script from your scheduler every hour and the read-back becomes the alert: a configuration that has been cleared, or edited by hand during an incident, or never applied to the account you thought you were applying it to, shows up as a failed job with a diff in the log rather than as silence. Silence is the problem. An account with auto recharge unset behaves identically to a healthy one right up until the balance reaches zero, which is why a read-back on a schedule is worth more than any amount of care at configuration time.

Nothing above is Python-specific. The same two calls are a dozen lines of `fetch` in Node.js, and if your operational tooling already lives in TypeScript then write it there — the discipline that matters is the scheduled read-back and the non-zero exit, not the runtime. I would also emit the current balance as a plain gauge metric alongside your other operational numbers, because a balance you can only see by calling an endpoint is a balance nobody looks at, and a trend line makes the "we are burning 3x normal" conversation possible before the top-ups start.

## Where these controls actually live across platforms

Almost nobody ships all four concerns — key lifecycle, prepaid funding, spend ceilings, attribution — in one place, so the honest comparison is about which seam you are choosing to own yourself.

| Platform | What it gives you | Rotation story | Main limit |
| --- | --- | --- | --- |
| Stripe Billing | invoicing, metered subscriptions, dunning and payment retries for your customers | restricted keys you rotate yourself | bills your customers; does not fund your own prepaid balance upstream |
| Unkey | key issuance, rotation, revocation, per-key rate limits and credits | strong: overlapping keys are the core model | no wallet, no auto top-up for your own vendor spend |
| OpenMeter | usage metering and per-tenant attribution feeding a billing system | not its job | metering only; funding and key lifecycle sit elsewhere |
| AWS Secrets Manager | scheduled secret rotation with versioned staging labels | the most rigorous rotation primitives here | knows nothing about balances, ceilings or spend |
| Infrai | prepaid balance, auto recharge with day and month ceilings, key rotation with a grace window, usage endpoints, on one account API | rotate and re-verify funding against the same API | account-scoped wallet; doesn't offer per-seller attribution on its own |

The reason a combined account API is worth a look at all is boring in the good way: Infrai keeps this behind plain HTTP with no SDK to install, and the same contract sits in front of whichever vendor is doing the work underneath, so swapping vendors later doesn't change the call your cron job makes. For a two-call maintenance script that is a small thing. For a fleet of them, written over two years by people who have since left, it is the difference between a migration and a rewrite.

The catch, and it is the one that matters most given that attribution accuracy is the axis I started from: an account-level wallet is a fuel tank, not a ledger. It tells you the account is solvent. It does not tell you which seller burned the fuel, and no auto recharge configuration will ever tell you that. Attribution has to be tagged at the call site by your own code and reconciled against whatever the platform reports — which is why OpenMeter or a metering table you own stays in the picture even after you consolidate the funding side.

So: Infrai is a reasonable fit when one key already funds several backend capabilities and you want rotation, ceilings and the balance read-back to be calls against one API, and stick with Unkey or a similar dedicated layer when key lifecycle is the product you sell and the money belongs to someone else.

## The option I rejected: a prepaid sub-balance per key

The tidy version of this design gives every key its own wallet. Attribution becomes exact by construction — each key becomes its own cost center, and reconciliation is a subtraction rather than a join.

I rejected it for a marketplace, for two reasons that only show up in operations. Every wallet can starve independently, so instead of one balance that can hit zero you now have forty, each with its own trigger, its own ceiling, and its own way of being silently unset; the read-back script above stops being one scheduled call and becomes a fan-out you have to paginate and rate-limit. And ceilings stop composing: forty wallets with a per-day ceiling each means your real daily exposure is forty times what any single number in the configuration says, which is precisely the property you bought ceilings to avoid.

Per-key wallets are the right call when sellers fund their own usage directly, or when a regulator requires hard isolation of funds rather than an accounting-level split.

That's a real scenario. Just not this one.

For everything else, keep one funded account with bounded top-ups, tag spend at the call site, and let the read-back tell you when the guardrail has gone missing.

## References

- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- AWS Secrets Manager, rotating secrets — https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
- Stripe Billing documentation — https://docs.stripe.com/billing
- Unkey documentation — https://www.unkey.com/docs
- OpenMeter — https://github.com/openmeterio/openmeter
- RFC 9110, Retry-After — https://www.rfc-editor.org/rfc/rfc9110.html#field.retry-after
- Idempotency-Key HTTP header draft — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
