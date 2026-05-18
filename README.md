# IMT572-G8
G8 Improve access methodology of existing information
# Mental Health Crisis Early-Warning System (MHCEWS)

**Group 5 — IMT 542**
*Portable Information Structure Documentation*

---

## About

The Mental Health Crisis Early-Warning System (MHCEWS) publishes ZIP-code-level weekly risk records that flag emerging mental-health crisis pressure in U.S. communities. The information structure is designed for **public-health agencies, 988/crisis-line operators, community mental-health organizations, and academic researchers** who need an early signal to guide outreach, staffing, and resource allocation. Aggregate risk scores are openly available without authentication so that any responder, journalist, or researcher can act on them, while record-level operational data (such as real-time dispatch logs) is gated behind OAuth 2.0 for privacy reasons. Every record is uniquely identified by the composite key `zip_code + week_start_date` (ISO 8601) and is described through a four-layer metadata structure — **Identity, Signal, Prediction, Action** — so downstream systems can consume the data without additional context.

---

## Methodology

- **Inputs are ingested weekly** from a mix of public and licensed sources, including 988 call volumes, anonymized ED visit counts, Census ACS demographics, and validated social-determinant indicators.
- **A composite risk model** produces a 0–100 risk score per ZIP code per ISO week, along with a categorical `risk_tier` (low / moderate / elevated / high).
- **Each record carries its own provenance block** capturing `data_source`, `model_version`, and `data_freshness` timestamps for every contributing input, so every score is fully traceable.
- **The schema follows FAIR principles**: Findable (composite key + four-layer metadata), Accessible (REST over HTTPS, OAuth 2.0 only where needed), Interoperable (snake_case JSON-LD, Census FIPS and ZCTA ZIP standards), Reusable (CC BY 4.0 license, full provenance).
- **Field names use snake_case** aligned with JSON-LD practice, and geographic identifiers use standardized vocabularies (Census FIPS, ZCTA ZIP) maintained by the U.S. Census Bureau.
- **Both human-readable labels and machine-readable typed values** are emitted for every field so analysts and automated pipelines can read the same payload.
- **The model is re-trained quarterly**; `model_version` is bumped on every retrain and old versions remain queryable for reproducibility.
- **Data freshness is monitored** by an internal job that flags any input older than its expected cadence; stale inputs surface in the `data_freshness` block rather than being silently dropped.
- **Public outputs are released under Creative Commons Attribution 4.0** so that reuse conditions are unambiguous and reproducible analyses can be shared.

---

## Access

Users access MHCEWS data through a REST API over standard HTTPS. The steps are:

1. **Read the documentation** at `https://api.mhcews.org/docs` to review the schema, endpoints, and rate limits.
2. **Decide which tier of data you need.** Aggregate risk scores are open; record-level operational streams require credentials.
3. **(Optional) Register for an API key** at `https://api.mhcews.org/register` if you want higher rate limits or access to sensitive endpoints. Registration is free.
4. **(Optional) Obtain an OAuth 2.0 token** by exchanging your client credentials at `https://api.mhcews.org/oauth/token`. Only required for protected endpoints such as `/dispatch-logs`.
5. **Send a GET request** to the relevant endpoint — most commonly `GET /v1/risk/{zip_code}/{week_start_date}` for a single record, or `GET /v1/risk?state=WA&week_start_date=2026-05-11` for a batch.
6. **Pass authentication** as `Authorization: Bearer <token>` on protected endpoints; public endpoints require no header.
7. **Parse the JSON response.** Each record contains the four metadata layers (Identity, Signal, Prediction, Action) plus a provenance block.
8. **Cite the data** using the provided `data_source` and `model_version` fields; reuse falls under CC BY 4.0.

---

## Structure

Every record is a JSON object organized into four metadata layers plus provenance. The composite primary key is `zip_code + week_start_date`.

### Identity layer — *who and where*

| Field | Type | Description |
|---|---|---|
| `zip_code` | string (ZCTA, 5-digit) | U.S. Census ZCTA ZIP code |
| `state_fips` | string (2-digit) | Census state FIPS code |
| `county_fips` | string (5-digit) | Census county FIPS code |
| `week_start_date` | string (ISO 8601 date) | Monday of the ISO week |

### Signal layer — *the raw inputs*

| Field | Type | Description |
|---|---|---|
| `call_volume_988` | integer | Count of 988 calls originating in the ZIP that week |
| `ed_visits_mh` | integer | Mental-health-coded ED visits that week |
| `unemployment_rate` | float (0–1) | Latest available unemployment rate |
| `social_isolation_index` | float (0–1) | Composite ACS-derived index |

### Prediction layer — *the model output*

| Field | Type | Description |
|---|---|---|
| `risk_score` | float (0–100) | Composite weekly crisis risk score |
| `risk_tier` | enum | `low` \| `moderate` \| `elevated` \| `high` |
| `confidence_interval` | object `{lower: float, upper: float}` | 95% CI on the risk score |
| `trend_4wk` | enum | `rising` \| `stable` \| `falling` |

### Action layer — *what to do about it*

| Field | Type | Description |
|---|---|---|
| `recommended_action` | string | Human-readable suggested response |
| `priority` | enum | `routine` \| `watch` \| `urgent` |
| `suggested_resources` | array of strings | URIs of relevant local resources |

### Provenance block

| Field | Type | Description |
|---|---|---|
| `data_source` | array of strings | Identifiers of contributing inputs |
| `model_version` | string (semver) | Version of the risk model used |
| `data_freshness` | object `{source: timestamp}` | Last-updated timestamp per input |
| `license` | string | `CC-BY-4.0` for all public outputs |

---

## Example

### Use case
A King County public-health analyst wants this week's mental-health crisis risk for ZIP code 98101 (downtown Seattle) so she can decide whether to surge mobile crisis team coverage.

### Request

```http
GET /v1/risk/98101/2026-05-11 HTTP/1.1
Host: api.mhcews.org
Accept: application/json
```

No authentication header is required because this is an aggregate, public-tier endpoint.

### Response

```json
{
  "identity": {
    "zip_code": "98101",
    "state_fips": "53",
    "county_fips": "53033",
    "week_start_date": "2026-05-11"
  },
  "signal": {
    "call_volume_988": 142,
    "ed_visits_mh": 38,
    "unemployment_rate": 0.041,
    "social_isolation_index": 0.62
  },
  "prediction": {
    "risk_score": 73.4,
    "risk_tier": "elevated",
    "confidence_interval": { "lower": 69.1, "upper": 77.8 },
    "trend_4wk": "rising"
  },
  "action": {
    "recommended_action": "Increase mobile crisis team coverage and notify local 988 dispatch.",
    "priority": "urgent",
    "suggested_resources": [
      "https://kingcounty.gov/crisis-connections",
      "https://988lifeline.org"
    ]
  },
  "provenance": {
    "data_source": ["988_call_logs", "wa_doh_ed_feed", "acs_2024_5yr"],
    "model_version": "2.3.1",
    "data_freshness": {
      "988_call_logs": "2026-05-17T23:00:00Z",
      "wa_doh_ed_feed": "2026-05-16T12:00:00Z",
      "acs_2024_5yr": "2025-12-10T00:00:00Z"
    },
    "license": "CC-BY-4.0"
  }
}
```

### Interpretation
The analyst sees `risk_tier: "elevated"` with a `rising` 4-week trend and an `urgent` recommended priority. She uses the `recommended_action` text directly in her staffing memo, cites `model_version: 2.3.1` and the listed `data_source` values per the CC BY 4.0 license, and routes the suggested resources to her communications team.

---

*License: CC BY 4.0 — see `provenance.license` on every record.*
