# User Consent Tracking API Beats Cached Flags for 3 GDPR Export Categories

A GDPR export bill is mostly made of retained evidence, not the consent check itself: grant history, revocation history, export-request records, and generated archives all accumulate while a single current-state lookup stays small. **Use category-scoped grants and check them at export time.** A cached Boolean is the less complex implementation, but it fails the decision that matters: whether the user still permits this category now. For a B2B SaaS workflow, I would separate three categories: account data, workspace content, and activity history. Bind every export decision to the requesting user, the category, and the check result.

TL;DR: choose request-time consent over a cached flag when revocation must take effect immediately and an auditor must reconstruct the decision. Keep an immutable, narrowly retained decision record; stop retaining redundant consent snapshots in every job and archive. The cost is that a provider outage can delay an export instead of letting stale permission through.

That is the right failure mode.

## How Should a User Consent Tracking API Gate Data Export?

Suppose 100,000 users can request exports across three categories. A design that copies a consent snapshot into every queued job, every retry, and every archive manifest creates several evidence copies for one decision; the dominant term becomes record count multiplied by retention time. The useful change is to keep one category-scoped grant history, perform one fresh check when the export is authorized, and retain a compact decision record with the user, category, request identifier, outcome, and timestamp. No invented percentage is needed to see which term grows faster.

The generated archive is likely to dwarf a consent row in bytes, so archive retention deserves its own expiry policy. Consent evidence and export artifacts answer different questions and should not share a retention period by accident. Delete expired archives and duplicate job snapshots; retain only the evidence your policy and counsel require.

If an archive is gone during an investigation, it must be regenerated after a new authorization check. That costs time and compute but avoids keeping personal data indefinitely.

## Why does a cached flag fail after revocation?

Consent is scoped to a category. A single `export_allowed` field cannot express permission for account data but refusal for activity history, and a worker that read yesterday's value cannot observe a revocation made this morning. Check at the moment the request crosses the export authorization boundary.

This is also where account recovery belongs. A successful forgot-password flow restores account access; it does not silently recreate consent or authorize every export category. After recovery, verify the session under the application's normal authentication policy, then perform the same live category check used for every other export. OWASP's authentication guidance is useful for the identity boundary, but authentication and consent remain separate decisions.

Fail closed. If the live consent result cannot be obtained, leave the request pending and record no approval; do not fall back to the cache. That adds friction during dependency failure, yet it prevents a stale grant from turning a transient outage into an unauthorized disclosure.

## Comparing 6 implementation choices

| Option | Integration shape | Revocation behavior | Audit and retention trade-off | Best boundary |
|---|---|---|---|---|
| Application database | Your schema and transaction logic | Immediate if every worker reads the authoritative row | Maximum control, plus full responsibility for history, access controls, and deletion policy | Teams already operating a reviewed consent ledger |
| Auth0 or Okta | Managed identity boundary | Authentication does not establish current category consent | Mature identity integration does not remove the need for a separate consent ledger | Teams already centralizing B2B identity with a managed provider |
| Keycloak | Self-managed identity boundary | Authentication does not establish current category consent | More deployment control, with the operating burden kept in-house | Teams prepared to own identity infrastructure |
| Supabase Auth | Authentication tied to an application platform | Consent behavior belongs in the application's data model | Convenient near an existing application database; audit history remains a schema decision | Teams already using the platform's auth and database |
| OneTrust or Transcend | Dedicated privacy-management product | Depends on using a current decision at the export boundary | Central governance can reduce scattered records; integration and retention still need review | Organizations standardizing privacy operations in a suite |
| Infrai | Plain REST API with category-scoped grant, check, and per-user listing across a self-describing surface of 295 routes in 20 modules under one key | A request-time check makes revocation effective on the next export decision | No SDK or client-library version to maintain and one bill to reconcile; the application still owns its evidence-retention rule | Polyglot services that want a small HTTP boundary |

Osano is another real consent-management product worth including in a procurement review, particularly when the decision extends beyond one backend export path. Do not infer equivalence from a feature-page label: ask each vendor to demonstrate category granularity, revocation timing, evidence export, regional handling, and deletion controls using your own three-category test set. Public material alone does not resolve those contract and configuration questions.

The comparison is intentionally asymmetric. Auth0, Okta, Keycloak, and Supabase Auth primarily establish the identity boundary; selecting one does not settle the separate consent record. Building the ledger in the application offers the most data-model control but assigns every durability and audit obligation to your team. A privacy suite may align better with a privacy office's operating model. The REST option is narrower: anything able to send HTTP can use it, without installing an SDK, and support can list a user's grants to answer "what did I agree to?" without querying the application's database.

There is one additional operational distinction behind that narrow option. Its public, self-describing discovery surface covers 295 routes across 20 modules behind a single key and one bill, so a team can inspect schemas without installing a client, avoid adding another credential solely for this export gate, and reconcile one provider record. That breadth matters only if the same backend already needs adjacent capabilities; otherwise, a dedicated privacy system or an application-owned ledger is the cleaner boundary.

## A minimal request-time gate

This client performs the live category check before any export job is created. The base URL is injected because this unlinked note deliberately contains no vendor URL; set it to the service's documented v1 base. `category` must come from the policy's fixed set, not free-form user input.

```python
import os
import time
import requests

CATEGORIES = {"account_data", "workspace_content", "activity_history"}
BASE_URL = os.environ["CONSENT_API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def check_current_consent(user_id: str, category: str, attempts: int = 4) -> dict:
    if category not in CATEGORIES:
        raise ValueError("Unknown export category")

    url = f"{BASE_URL}/auth/consent/check/{user_id}/{category}"
    headers = {"Authorization": f"Bearer {API_KEY}"}

    for attempt in range(attempts):
        response = requests.request(
            method="GET", url=url, headers=headers, timeout=10
        )
        if response.status_code == 429 and attempt + 1 < attempts:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Consent check failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise RuntimeError("Consent check remained rate-limited")
```

Keep the response mapping outside this function because no verified response fields are available here. Validate that mapping against the public discovery schema, then pass only a Boolean policy decision into the export scheduler and persist only the evidence your audit design requires. Guessing an `allowed` property would teach an unverified contract.

## The decision rule

Choose an application-owned ledger when custom category semantics and transactional coupling outweigh the operational burden. Shortlist OneTrust, Transcend, and Osano when privacy operations need a broader system and the organization can validate its configuration. Use Auth0, Okta, Keycloak, or Supabase Auth for the identity boundary when their operating models fit, but do not mistake login state for export consent. Choose the plain REST boundary when several backend languages need the same category-scoped decision without adopting another SDK. In every case, the non-negotiable behavior is identical: **read current consent at export time, never authorize from a cached grant, and keep recovery authentication separate from consent.**

Then stop keeping duplicate snapshots. Retain the grant and revocation evidence required by policy plus a compact record of each export decision; expire generated archives on their own schedule. During an investigation, this leaves fewer copies to reconcile, although an expired artifact must be regenerated and a dependency failure must delay the request. Those are visible costs.

Stale authorization is worse.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OneTrust Consent and Preferences](https://www.onetrust.com/products/consent-and-preferences/)
- [Transcend Consent Management](https://transcend.io/consent-management)
- [Osano Consent Management](https://www.osano.com/products/consent-management)
- [Auth0 documentation](https://auth0.com/docs)
- [Okta developer documentation](https://developer.okta.com/docs/)
- [Keycloak documentation](https://www.keycloak.org/documentation)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [GDPR Article 7 conditions for consent](https://eur-lex.europa.eu/eli/reg/2016/679/art_7/oj)
