# Affise (affise)

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Affise is a performance marketing platform with a REST API for managing affiliate offers, publishers, conversions, payouts, and accessing detailed campaign analytics. The platform serves affiliate networks, agencies, and advertisers with tools for tracking, attribution, automation, and partner management across its Performance, MMP, and Reach product lines.

APIs.json: https://raw.githubusercontent.com/api-evangelist/affise/refs/heads/main/apis.yml

Naftiko: https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=affise-api-evangelist&utm_content=repo

## Tags

- Affiliate Marketing
- Performance Marketing
- Conversions
- Publishers
- Analytics
- Attribution

## APIs

- **Affise Performance API** — REST API for the Affise Performance platform enabling admins and affiliates to manage offers, track conversions, retrieve statistics, handle publisher payouts, and automate billing operations programmatically.
- **Affise MMP API** — Mobile Measurement Partner API enabling mobile app attribution tracking, install measurement, event tracking, and audience analytics for iOS, Android, and cross-platform mobile applications.

## Plans / Rate Limits / FinOps

- **Plans**: [plans/affise-plans-pricing.yml](plans/affise-plans-pricing.yml) — Affise Performance (Beginner $625/mo, Core $888/mo, Expand $1,239/mo, Custom $2,499/mo) and Affise MMP (Scale $700/mo, Enterprise $1,500/mo, Custom) with ~20% annual billing discounts.
- **Rate Limits**: [rate-limits/affise-rate-limits.yml](rate-limits/affise-rate-limits.yml) — Rate limits enforced per API key; specific numeric thresholds are not publicly documented and vary by plan tier. Authentication uses an API-Key header or query parameter.
- **FinOps**: [finops/affise-finops.yml](finops/affise-finops.yml) — Cost drivers include monthly conversion volume, impression volume, tracking domain count, and billing cycle selection. Annual billing saves approximately 20% versus monthly.

## Timestamps

- Created: 2026-06-13
- Modified: 2026-06-13

## Common

| Type | URL |
|------|-----|
| Website | https://affise.com/ |
| Documentation | https://help-center.affise.com/en/ |
| GitHub Org | https://github.com/affise |
| LinkedIn | https://www.linkedin.com/company/affise-com/ |
| Blog | https://affise.com/blog/ |
| Pricing | https://affise.com/pricing/ |
| Status Page | https://status.affise.com/ |
| X | https://twitter.com/GetAffise |

## Maintainers

- **Kin Lane** — kin@apievangelist.com
