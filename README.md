# IMT572-G8
G8 Improve access methodology of existing information
# Mental Health Crisis Early-Warning System (MHCEWS)

**Group 5 — IMT 542**
*Portable Information Structure Documentation*

> **Note on scope:** MHCEWS is a *proposed* information structure designed for this assignment. It is not a deployed service. The example in this document demonstrates how MHCEWS records can be assembled from **real, publicly accessible APIs** — primarily the [CDC PLACES](https://www.cdc.gov/places/) Socrata Open Data API and the [SAMHSA FindTreatment.gov](https://findtreatment.gov/) locator — so that the structure can be implemented today using existing federal open-data infrastructure.

---

## About

The Mental Health Crisis Early-Warning System (MHCEWS) is a portable information structure for publishing ZIP-code-level weekly mental-health risk records in the United States. The intended audience includes **public-health agencies, 988/crisis-line operators, community mental-health organizations, and academic researchers** who need an early signal to guide outreach, staffing, and resource allocation. Aggregate risk scores are designed to be openly available without authentication so that any responder, journalist, or researcher can act on them, while record-level operational data (such as real-time dispatch logs) would be gated behind OAuth 2.0 for privacy reasons. Every record is uniquely identified by the composite key `zip_code + week_start_date` (ISO 8601) and is described through a four-layer metadata structure — **Identity, Signal, Prediction, Action** — so downstream systems can consume the data without additional context.

---

## Methodology

- **Inputs are drawn from real federal open-data services**, including:
  - **CDC PLACES** (`data.cdc.gov`) — ZCTA-level prevalence estimates for *frequent mental distress*, *social isolation*, *lack of social and emotional support*, and other health-related social needs, derived from BRFSS and ACS.
  - **SAMHSA FindTreatment.gov locator** — list of nearby licensed mental-health and crisis-treatment facilities by ZIP code.
  - **988 Suicide & Crisis Lifeline** aggregate call data (where state agencies publish it; not yet uniformly available as an API).
- **A composite risk model** combines these inputs into a 0–100 weekly risk score per ZCTA and assigns a categorical `risk_tier` (low / moderate / elevated / high).
- **Each record carries its own provenance block** capturing `data_source` (with the upstream dataset ID), `model_version`, and `data_freshness` timestamps for every contributing input, so every score is fully traceable back to the upstream federal release.
- **The schema follows FAIR principles**: Findable (composite key + four-layer metadata), Accessible (REST over HTTPS, OAuth 2.0 only where needed), Interoperable (snake_case JSON-LD, Census FIPS and ZCTA ZIP standards), Reusable (CC BY 4.0 license, full provenance).
- **Field names use snake_case** aligned with JSON-LD practice; geographic identifiers use the same ZCTA codes used by CDC PLACES and Census Bureau vocabularies (FIPS, ZCTA).
- **Both human-readable labels and machine-readable typed values** are emitted for every field so analysts and automated pipelines can read the same payload.
- **The model would be re-trained on each CDC PLACES annual release**; `model_version` is bumped on every retrain and old versions remain queryable for reproducibility.
- **Data freshness is monitored** because PLACES updates annually while SAMHSA locator data updates weekly — the `data_freshness` block surfaces each input's last-updated timestamp rather than silently mixing stale and fresh data.
- **Public outputs are released under Creative Commons Attribution 4.0**, consistent with the open-data licensing of the underlying federal sources.

---

## Access

Until MHCEWS is deployed as a standalone API, users can access its underlying inputs directly through the real federal endpoints listed below. The intended workflow is:

1. **Read the CDC PLACES documentation** at <https://www.cdc.gov/places/tools/data-portal.html> and the SAMHSA FindTreatment developer guide at <https://findtreatment.gov/assets/FindTreatment-Developer-Guide.pdf>.
2. **Decide which tier of data is needed.** Aggregate health-prevalence estimates from CDC PLACES are fully open (no auth). Treatment-facility lookups from SAMHSA are also open. Any future record-level operational stream (e.g. dispatch logs) would require OAuth 2.0.
3. **Query the CDC PLACES ZCTA dataset** via the Socrata Open Data API (SODA). The resource ID for the current ZCTA release is `qnzd-25i4`; the JSON endpoint is:
   `https://data.cdc.gov/resource/qnzd-25i4.json`
4. **Filter by ZCTA and measure** using SoQL query parameters, for example:
   `?locationname=98101&measureid=MHLTH`
   (`MHLTH` = *frequent mental distress among adults*).
5. **Query the SAMHSA FindTreatment.gov locator** for nearby crisis-care resources to populate the Action layer.
6. **Parse the JSON response** and map it into the MHCEWS four-layer schema (Identity, Signal, Prediction, Action) plus provenance block.
7. **Cite the data** using the upstream `data_source` IDs (e.g. `cdc_places_zcta_qnzd-25i4`) and `model_version`. Reuse falls under CC BY 4.0 for MHCEWS-derived outputs; upstream sources retain their own licenses (CDC PLACES is U.S. Government work; SAMHSA locator is public-domain).

---

## Structure

Every record is a JSON object organized into four metadata layers plus provenance. The composite primary key is `zip_code + week_start_date`.

### Identity layer — *who and where*

| Field | Type | Description |
|---|---|---|
| `zip_code` | string (ZCTA, 5-digit) | U.S. Census ZCTA ZIP code (matches CDC PLACES `locationname`) |
| `state_fips` | string (2-digit) | Census state FIPS code |
| `county_fips` | string (5-digit) | Census county FIPS code |
| `week_start_date` | string (ISO 8601 date) | Monday of the ISO week |

### Signal layer — *the raw inputs*

| Field | Type | Description | Upstream source |
|---|---|---|---|
| `frequent_mental_distress_pct` | float (0–100) | % adults reporting frequent mental distress | CDC PLACES `MHLTH` |
| `social_isolation_pct` | float (0–100) | % adults reporting feeling socially isolated | CDC PLACES `ISOLATION` |
| `lack_emotional_support_pct` | float (0–100) | % adults lacking social and emotional support | CDC PLACES `EMOTIONSPT` |
| `nearby_treatment_facility_count` | integer | Count of SAMHSA-listed mental-health facilities within the ZIP | SAMHSA FindTreatment.gov |

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
| `suggested_resources` | array of strings | URIs of relevant local resources (e.g. 988, SAMHSA facility pages) |

### Provenance block

| Field | Type | Description |
|---|---|---|
| `data_source` | array of strings | Identifiers of contributing inputs (e.g. `cdc_places_zcta_qnzd-25i4`) |
| `model_version` | string (semver) | Version of the risk model used |
| `data_freshness` | object `{source: timestamp}` | Last-updated timestamp per input |
| `license` | string | `CC-BY-4.0` for MHCEWS-derived outputs |

---

## Example

### Use case
A King County public-health analyst wants this week's mental-health crisis risk for ZIP code 98101 (downtown Seattle) so she can decide whether to surge mobile crisis team coverage. The MHCEWS Signal layer is populated by a live call to the real CDC PLACES API.

### Request to CDC PLACES (Signal-layer input — real, executable)

```http
GET /resource/qnzd-25i4.json?locationname=98101&measureid=MHLTH HTTP/1.1
Host: data.cdc.gov
Accept: application/json
```

No authentication header is required — CDC PLACES is fully open via the Socrata SODA API.

### Response from CDC PLACES (truncated, illustrative)

```json
[
  {
    "year": "2024",
    "locationname": "98101",
    "datasource": "BRFSS",
    "category": "Health Status",
    "measure": "Frequent mental distress among adults",
    "measureid": "MHLTH",
    "data_value_type": "Crude prevalence",
    "data_value": "16.8",
    "low_confidence_limit": "15.4",
    "high_confidence_limit": "18.3",
    "totalpopulation": "13095",
    "geolocation": { "type": "Point", "coordinates": [-122.3331, 47.6097] }
  }
]
```

### Assembled MHCEWS record (what MHCEWS would return downstream)

```json
{
  "identity": {
    "zip_code": "98101",
    "state_fips": "53",
    "county_fips": "53033",
    "week_start_date": "2026-05-11"
  },
  "signal": {
    "frequent_mental_distress_pct": 16.8,
    "social_isolation_pct": 22.1,
    "lack_emotional_support_pct": 19.4,
    "nearby_treatment_facility_count": 12
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
      "https://988lifeline.org",
      "https://findtreatment.gov/locator?zip=98101",
      "https://kingcounty.gov/depts/community-human-services/mental-health-substance-abuse.aspx"
    ]
  },
  "provenance": {
    "data_source": [
      "cdc_places_zcta_qnzd-25i4",
      "samhsa_findtreatment_locator",
      "wa_doh_988_aggregate"
    ],
    "model_version": "0.1.0-prototype",
    "data_freshness": {
      "cdc_places_zcta_qnzd-25i4": "2025-12-11T00:00:00Z",
      "samhsa_findtreatment_locator": "2026-05-15T00:00:00Z",
      "wa_doh_988_aggregate": "2026-05-17T23:00:00Z"
    },
    "license": "CC-BY-4.0"
  }
}
```

### Interpretation
The analyst sees `risk_tier: "elevated"` with a `rising` 4-week trend and an `urgent` recommended priority. She uses the `recommended_action` text directly in her staffing memo, cites the upstream `data_source` IDs (so a reviewer can re-pull the exact CDC PLACES row), and routes the suggested SAMHSA and 988 links to her communications team.

---

## Real APIs referenced in this document

| Service | Base URL | Auth | License |
|---|---|---|---|
| CDC PLACES ZCTA dataset (Socrata SODA) | `https://data.cdc.gov/resource/qnzd-25i4.json` | None (public) | U.S. Government work |
| CDC PLACES portal | <https://www.cdc.gov/places/> | n/a | n/a |
| SAMHSA FindTreatment.gov | <https://findtreatment.gov/> | API key on request | Public-domain |
| 988 Suicide & Crisis Lifeline | <https://988lifeline.org/> | n/a | n/a |

*License for MHCEWS-derived outputs: CC BY 4.0 — see `provenance.license` on every record.*
