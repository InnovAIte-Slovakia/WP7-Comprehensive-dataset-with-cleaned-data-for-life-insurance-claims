# ASTRA Synthetic Life-Insurance Claims (Slovak)

**An evaluation set of synthetic insurance claim files — for testing and evaluating claim-handling
systems, not for training models.**

| | |
|---|---|
| **Publisher** | Asseco Central Europe, a.s. |
| **Repository** | [InnovAIte-Slovakia/WP7-Comprehensive-dataset-with-cleaned-data-for-life-insurance-claims](https://github.com/InnovAIte-Slovakia/WP7-Comprehensive-dataset-with-cleaned-data-for-life-insurance-claims) |
| **License** | [Apache License 2.0](LICENSE) |
| **Version** | 1.0.0 |
| **Language** | Slovak (`sk`) — data and documents; this README is in English |
| **Contact** | [michal.porazik@asseco-ce.com](mailto:michal.porazik@asseco-ce.com) |

## Overview

The dataset holds **100 claim files** of a fictional Slovak life insurer, *ASTRA Životná
poisťovňa, a.s.*, for two riders of its life product: **temporary work incapacity** (a daily
benefit, *denná dávka pri práceneschopnosti*) and **injury treatment** (*liečenie úrazu*). Each file
is what a claims system receives when a claim is notified:

- the **core-system records** — the policy and its persons, the covers on the policy and the claim
  notification, in the shape of an insurer's own export;
- the **claim documents** the client attached — notification form, incapacity certificate, medical
  reports, discharge summary, police or work-accident report, policy certificate — as their
  **full text**.

Next to it, every file carries its **ground truth**: how the claim should be routed (decide
automatically, ask for more, or hand to a human), what should be decided, how the claim is
classified (cause and claim type per tariff), and how much is paid per tariff and for which days.
**10 closed historical claims** of the same insurer come with it, as the precedents a claim handler
looks up.

**The ground truth is computed, not annotated.** The expected outcome of every file follows
deterministically from the rules of the fictional product; no label was assigned by hand or by an
LLM.

**No real person, policy or claim is in the dataset.** All people, addresses, doctors, employers
and identifiers are synthetic. The insurer and its product are fictional.

### Intended uses

- **Information extraction** from Slovak medical and insurance documents: dates, diagnosis,
  treating doctor, employer, cause of the event, hospital stay.
- **Document classification**: which of twelve document types an attachment is.
- **PII detection and anonymisation**, with every planted personal value labelled per document.
- **End-to-end regression tests** of claim-handling pipelines — extraction → rules → reasoning →
  decision — for systems that implement the product's rules (see *Limitations*).
- **Routing and decision support**: does the system recognise a claim it can decide, a claim that
  needs more documents, and a claim a human must decide?

### Out of scope

- **Training or fine-tuning.** The set is small, covers one fictional product, and a model trained
  on it can no longer be evaluated on it.
- **Actuarial or statistical analysis.** The mix of scenarios is designed to cover rules and
  failure modes, not to resemble real claim frequencies.
- **Insurance or legal advice.** The product is fictional and binds no one.
- **OCR on real images.** This release carries the documents as text only (see *Limitations*).

## How the dataset was created

Each file is generated from a scenario: a fictional policy with its persons and covers, a claim
event, and the documents a client of a Slovak life insurer would attach to that claim. People,
places, doctors, employers and event stories are fictional, and every story is consistent with the
structured facts and with the season it happens in. The expected outcome is computed from the
same facts by the product's rules.

Every release is validated before it is published: the input records carry no part of the answer
and no claim record, identifiers are well-formed, the expected outcome agrees with the policy (daily
benefit × days, paid days within the policy term, the total), dates are possible, and no marker of
any real insurer appears anywhere. Generation is deterministic, so the same set can be produced
again.

**The 100 files are chosen to cover the generator's variability, not drawn at random.** Every value
that changes the expected outcome or routing, and every document variant — which documents are on
file, each document type in each file format, each conditional document in each scenario — appears
at least once. So does every story, diagnosis, injury item, employer, occupation and month of the
event. A generated scenario therefore appears several times, each time with a different story; 83
of the 93 possible scenario × story pairs are in the set.

### Scenario families

| Family | Ids | Files | What they test |
|---|---|---|---|
| Named | S01–S10 | 10 | Hand-designed claims: each daily-benefit tariff pair (29+, 57+, 85+), whole or half, injury treatment, events a human must decide |
| Generated, clean | G01–G12 | 49 | The tariff pairs whole and half, each injury-treatment tariff, an injury claimed on both risks. G01 and G02 are **one policy insuring two people**, each notifying their own event |
| Generated, one defect | G13–G20 | 41 | One defect each: a missing required document, an event after the policy ended, an illness in the waiting period, an incapacity shorter than the elimination period, an incapacity still running, a certificate contradicting the notification's dates, an exclusion a police report proves, a treatment longer than the injury allows |
| History | H01–H10 | 10 | Closed claims of 2023–2024: claims similar to an evaluated one, an earlier claim of the same insured, the same doctor with the same diagnosis for other people, and ordinary unrelated claims |

## Dataset structure

```
README.md
data/
  cases.jsonl     100 evaluated claim files
  history.jsonl    10 closed claims
LICENSE            Apache License 2.0
```

Both files are UTF-8 JSON Lines, one record per line. Conventions:

- **Amounts** are decimal strings with two places (`"598.00"`), never floats; currency is `EUR`.
- **Dates** in the core-system records keep the insurer's export format
  (`"2025-02-13 00:00:00.000"`); every field this dataset adds uses ISO dates (`"2025-02-13"`).
- **Core-system field names are kept as the insurer exports them** (`zml_id`, `rodne_cislo`,
  `POISTNA_SUMA1`, `loss_occurance_date` …), including their mixed case and spelling, so a system
  built for the real export reads them unchanged. The data dictionaries below explain them.
- A value that does not exist is `null`; an empty list means "none".

### The one rule: `input` vs ground truth

> **Only `input` is given to the system under test. Everything else in a case record is ground
> truth.**

Some ground truth looks harmless and still gives the answer away: a document's `kind` (the system
must classify it), its `issued_on` (it must read it), the PII labels, `scenario.description` and
`defects`. Keep all of it out of the prompt, the retrieval index and the tools of the system being
evaluated.

### `cases` — one evaluated claim file per record

| Field | Type | Description |
|---|---|---|
| `case_id` | string | `ASTRA-` + notification number, e.g. `ASTRA-HL-2025-002209` |
| `dataset_version` | string | Version of this dataset, e.g. `1.0.0` |
| `input` | object | What the system receives: the core-system records and the documents |
| `scenario` | object | Which scenario the file realises, and its planted defects — **ground truth** |
| `expected` | object | Routing, decision, classification, amounts and facts — **ground truth** |
| `labels` | object | Document types, PII entities, related history — **ground truth** |

#### `input.case`

| Field | Description |
|---|---|
| `case_id` | Same as the top-level `case_id` |
| `policy_number` | `ZP-<year concluded>-<6 digits>` |
| `country_code` | `SK` |

#### `input.structured` — the core-system records

Three datasets of the insurer's export, each `{"rows": [...]}`. Nothing in them is decided yet:
no claimed cover, no cause code, no reserve, no payout. The claim record itself — indemnity,
reserves, costs, payout — does not exist until the claim is registered; the system has to derive it
from the documents.

**`dataset_1_policies`** — one row per person and role on the policy.

| Field | Description |
|---|---|
| `policy_number`, `zml_id` | Policy number; technical contract id (16 digits) |
| `policy_start_date`, `policy_end_date`, `policy_payment_end_date` | Start and end of the insurance and of premium payment |
| `partner_id`, `fo_id`, `zmlpar_id` | Technical ids of the person and of the person-on-contract link |
| `rodne_cislo` | Slovak birth number (synthetic) |
| `fo_partner_name`, `fo_partner_surname`, `fo_birth_date`, `fo` | First name, surname, birth date, gender (`M` / `Z` = female) |
| `titul1`, `po_nazov` | Academic title; name of a legal person (both `null` here) |
| `pstat`, `adr_ulica`, `adr_sup_cislo`, `psc`, `adr_obec`, `astat` | Citizenship, street, house number, postcode, town, country |
| `fun_kod`, `FUN_NAZOV` | Role on the policy: `FUN_004` *Poistník* (policyholder), `FUN_005` *Platiteľ poistného* (premium payer), `FUN_006` *Poistený* (insured) |

**`dataset_2_policy_covers`** — one row per cover (tariff) on the policy.

| Field | Description |
|---|---|
| `cpz` | Policy number |
| `kod`, `cover_name` | Cover code `COV_201`–`COV_208` and name |
| `POISTNA_SUMA1` | Insured amount of the cover; for the daily-benefit tariffs the daily benefit in EUR |
| `risk_kod`, `risk_desc` | `RISK_023` *Pracovná neschopnosť* (work incapacity) or `RISK_046` *Liečenie úrazu* (injury treatment) |
| `cover_indemnity_type_kod`, `cover_indemnity_type_desc` | Indemnity type of the cover |
| `product_code`, `product_name`, `sysowner_code`, `OWNER` | Product, the core-system book it is administered in, and the line of business (`LIFE`) |

**`dataset_5_notification_partner`** — the claim notification as the client filled it in.

| Field | Description |
|---|---|
| `notification_number`, `zml_zml_id`, `policy_number` | `HL-<year>-<6 digits>`; contract id; policy number |
| `notification_date`, `loss_occurance_date` | When the claim was notified; the event date the client gave |
| `notif_status_code`, `notif_status_desc` | Status of the notification (`RU` — registered) |
| `loss_notif_event_description` | What happened, **in the client's own words** |
| `loss_notif_place_description`, `loss_notif_note` | Where it happened; a free note |
| `notifier_*` | Who notified: name, surname, relation to the policy, e-mail, phone |
| `loss_partner_*` | The person claiming: type (`F` natural person), function, name, birth number, birth date |

#### `input.documents[]` — the attached documents, as text

| Field | Type | Description |
|---|---|---|
| `doc_id` | string | `doc1` … `docN`, unique within the file; `doc1` is always the notification form |
| `file_name` | string | The name of the rendered file, e.g. `doc4.png` |
| `format` | string | `pdf_electronic` (born-digital PDF), `pdf_scan` (a scanned page in a PDF), `img_jpg`, `img_png` (photographed or scanned page) |
| `file_created_on` | date | The day the file was made — what a system reads from the PDF `CreationDate` or the image EXIF. A policy certificate is issued at the policy start, but the copy attached to a claim is made when the claim is notified |
| `raw_text` | string | The full text of the document, see below |

**`raw_text`** is the text printed on the page, exactly as rendered, in reading order: issuer
block, title, labelled fields (`Label: value`), then each section's heading and text, the cover
table where there is one (`item  amount EUR`), the date, the signatory and the signature caption.
Lines are separated by `\n`. It carries no OCR noise — for a scanned or photographed document it is
the text an ideal OCR would return.

Example (`doc2`, *Potvrdenie o dočasnej pracovnej neschopnosti*, shortened):

```text
Nemocnica Javor Banská Bystrica, Interné oddelenie
Nemocničná 6, 974 01 Banská Bystrica
Potvrdenie o dočasnej pracovnej neschopnosti
Pacient: Alžbeta Tóthová
Rodné číslo: 9761257044
Diagnóza: J18.9 – Pneumónia, bližšie neurčená
Práceneschopnosť od: 13.02.2025
Práceneschopnosť do: 11.04.2025
Potvrdenie
Menovaná je od 13.02.2025 uznaná dočasne práceneschopnou pre diagnózu J18.9 – Pneumónia, bližšie neurčená. …
Ukončenie pracovnej neschopnosti
Dočasná pracovná neschopnosť ukončená dňa 11.04.2025 po kontrolnom vyšetrení. …
Dátum: 11.04.2025
MUDr. Oľga Oravcová
(podpis a pečiatka)
```

A defect shows the way it would in a real claim file: a withheld document is simply not in the
list, a running incapacity has no document of its end, a contradicting document states its own
date, and a police report states the finding that proves an exclusion.

#### `scenario` — ground truth

| Field | Description |
|---|---|
| `scenario_id` | `S01`–`S10`, `G01`–`G20` |
| `family` | `named`, `generated` |
| `description` | One sentence (Slovak) describing the scenario — it states the answer |
| `defects[]` | `{code, detail}` for each planted defect; empty for a clean file. Codes: `missing_document`, `outside_policy`, `waiting_period`, `elimination_not_reached`, `incapacity_ongoing`, `date_conflict`, `exclusion_evidenced`, `treatment_over_cap` |

#### `expected` — ground truth

| Field | Type | Description |
|---|---|---|
| `control_state` | string | `ready` — the claim can be decided; `needs_info` — documents or facts are missing and must be requested from the client; `needs_hitl` — a human adjuster must decide |
| `decision_outcome` | string | `approved`, `partial`, `denied`, or `n/a` when the claim is not decided automatically |
| `decision_outcome_candidate` | string \| null | The decision a human is expected to reach for a `needs_hitl` file (for a `ready` file equal to `decision_outcome`; `null` for `needs_info`) |
| `amount`, `currency` | string | Total benefit to pay |
| `tariffs[]` | list | One entry per claimed tariff, see below |
| `missing[]` | list of strings | What must be requested for a `needs_info` file: a document kind (e.g. `work_incapacity_certificate`) or a fact (e.g. `incapacity_end`) |
| `hitl_reasons[]` | list | `{code, ground}` — why a human decides (e.g. `event_within_signal_period`, `conflicting_facts`, `exclusion_evidenced`) and the provision that says so |
| `facts` | object | The facts of the claim, see below |

**`expected.tariffs[]`**

| Field | Description |
|---|---|
| `tariff` | Cover code, e.g. `COV_203` |
| `status` | `paid`, `denied`, `pending` (waits on a human or on a document), `not_arising` (the claim does not reach this tariff) |
| `cause_code`, `cause_desc`, `claim_type` | How the registered claim is classified: the cause of the claim (`CC_…`) with its description, and the claim type (`CT_…`, the indemnity type of the cover, described in `dataset_2_policy_covers`) |
| `days`, `daily_benefit`, `amount` | Days paid, daily benefit, amount for the tariff |
| `paid_from`, `paid_to` | The paid period, `null` when nothing is paid |
| `grounds[]` | The provisions of the product's terms that decide it, as an adjuster cites them (`"PP-ASTRA-PU, čl. 16.2"`) |
| `capped` | `true` when the amount was cut by a limit |

**`expected.facts`** — the facts of the claim as the generator's world holds them, in the same
shape as `history.claim`: `claimed_risks[]`, `cause` (`illness` / `injury` / `pregnancy`),
`event_date`, `notified_on`, `diagnosis_code` (ICD-10), `diagnosis_name`,
`provider {doctor, facility, address}`, `employer {name, address}`, `incapacity_from`,
`incapacity_to`, `hospital_from`, `hospital_to`, `treatment_from`, `treatment_to`,
`valuation_item` (the code of the injury item, for injury treatment).
Facts are the truth of the world, not what the documents prove: where a defect withholds or
contradicts a fact (`incapacity_ongoing`, `date_conflict`), the documents do not support it — that
is the point of the defect.

#### `labels` — ground truth

**`labels.documents[]`** — one entry per document in `input.documents`.

| Field | Description |
|---|---|
| `doc_id` | Joins `input.documents[].doc_id` |
| `kind` | Document type, see the table below |
| `title` | The document's own title, e.g. *Lekárska správa z prvého vyšetrenia* |
| `language` | `sk` |
| `issued_on` | The date printed on the document |
| `is_claim_evidence` | `true` for documents that evidence the claim; `false` for the policy certificate and proposal |
| `pii_entities[]` | `{type, value, occurrences, spans}` — every planted personal value in `raw_text`; `spans` are `[start, end)` character offsets into `raw_text`, longest values matched first so a street is not counted again inside its full address |

PII types: `person_name`, `national_id`, `birth_date`, `address`, `email`, `phone`, `iban`,
`employer`, `insurance_number`, `special_category_health` (diagnosis code and name).

| `kind` | Document |
|---|---|
| `loss_notification_form` | *Oznámenie poistnej udalosti* — claim notification form |
| `work_incapacity_certificate` | *Potvrdenie o dočasnej pracovnej neschopnosti* — incapacity certificate |
| `medical_report` | *Lekárska správa z prvého vyšetrenia* — first examination |
| `medical_course_report` | *Lekárska správa z priebehu liečenia* — course of treatment |
| `medical_findings_current` | Current medical findings after the treatment ended, with the day it ended (injury treatment) |
| `discharge_summary` | *Prepúšťacia správa z nemocnice* — hospital discharge summary |
| `sickness_benefit_confirmation` | Social Insurance Agency's confirmation of sickness benefit (self-employed insured) |
| `employment_confirmation` | Confirmation of employment or self-employment |
| `work_accident_report` | Final report of a work-accident investigation |
| `police_report` | Police record or conclusion of a police investigation |
| `policy_certificate` | *Poistka* — policy certificate |
| `policy_proposal` | *Návrh poistnej zmluvy* — policy proposal |

**`labels.related_history[]`** — `{history_case_id, relation}` for each closed claim in `history`
linked to this file: `similar_to` (same diagnosis and course, another doctor), `earlier_claim_of`
(the same insured, same policy), `same_doctor_diagnosis` (the same doctor with the same diagnosis
for another person). Empty for most files.

#### Example record (shortened)

```json
{
  "case_id": "ASTRA-HL-2025-002209",
  "dataset_version": "1.0.0",
  "input": {
    "case": {"case_id": "ASTRA-HL-2025-002209", "policy_number": "ZP-2017-804424", "country_code": "SK"},
    "structured": {
      "dataset_1_policies": {"rows": [{"policy_number": "ZP-2017-804424", "fo_partner_name": "Alžbeta", "fo_partner_surname": "Tóthová", "fun_kod": "FUN_006", "FUN_NAZOV": "Poistený"}]},
      "dataset_2_policy_covers": {"rows": [{"kod": "COV_203", "POISTNA_SUMA1": "19.00", "risk_kod": "RISK_023"}, {"kod": "COV_204", "POISTNA_SUMA1": "10.00", "risk_kod": "RISK_023"}]},
      "dataset_5_notification_partner": {"rows": [{"notification_number": "HL-2025-002209", "notification_date": "2025-02-18 07:05:45.000", "loss_occurance_date": "2025-02-13 00:00:00.000", "loss_notif_event_description": "Po prechladnutí sa mi zhoršil kašeľ a niekoľko dní som mala vysoké horúčky, preto som vyhľadala lekára."}]}
    },
    "documents": [
      {"doc_id": "doc1", "file_name": "doc1.pdf", "format": "pdf_electronic", "file_created_on": "2025-02-18", "raw_text": "Alžbeta Tóthová\nDlhá 90, 97401 Banská Bystrica\nOznámenie poistnej udalosti\n…"},
      {"doc_id": "doc2", "file_name": "doc2.pdf", "format": "pdf_electronic", "file_created_on": "2025-04-11", "raw_text": "Nemocnica Javor Banská Bystrica, Interné oddelenie\n…"}
    ]
  },
  "scenario": {"scenario_id": "G04", "family": "generated", "description": "Dvojica 57+ celá, choroba: J18.9 Pneumónia, bližšie neurčená, PN 58 dní.", "defects": []},
  "expected": {
    "control_state": "ready",
    "decision_outcome": "approved",
    "decision_outcome_candidate": "approved",
    "amount": "598.00",
    "currency": "EUR",
    "tariffs": [
      {"tariff": "COV_203", "status": "paid", "cause_code": "CC_043", "cause_desc": "Ochorenia dýchacích ciest", "claim_type": "CT_064", "days": 2, "daily_benefit": "19.00", "amount": "38.00", "paid_from": "2025-04-10", "paid_to": "2025-04-11", "grounds": ["PP-ASTRA-PU, čl. 16.2"], "capped": false},
      {"tariff": "COV_204", "status": "paid", "cause_code": "CC_043", "cause_desc": "Ochorenia dýchacích ciest", "claim_type": "CT_061", "days": 56, "daily_benefit": "10.00", "amount": "560.00", "paid_from": "2025-02-13", "paid_to": "2025-04-09", "grounds": ["PP-ASTRA-PU, čl. 16.2"], "capped": false}
    ],
    "missing": [],
    "hitl_reasons": [],
    "facts": {"claimed_risks": ["RISK_023"], "cause": "illness", "event_date": "2025-02-13", "diagnosis_code": "J18.9", "incapacity_from": "2025-02-13", "incapacity_to": "2025-04-11", "hospital_from": "2025-02-13", "hospital_to": "2025-02-18"}
  },
  "labels": {
    "documents": [{"doc_id": "doc2", "kind": "work_incapacity_certificate", "title": "Potvrdenie o dočasnej pracovnej neschopnosti", "language": "sk", "issued_on": "2025-04-11", "is_claim_evidence": true, "pii_entities": [{"type": "person_name", "value": "Alžbeta Tóthová", "occurrences": 1, "spans": [[142, 157]]}]}],
    "related_history": []
  }
}
```

### `history` — closed claims

Sealed records of claims decided in 2023–2024. They are the material a system searches for
precedents and risk signals — for example a doctor who keeps certifying the same diagnosis — never
files to evaluate.

| Field | Description |
|---|---|
| `history_case_id` | `ASTRA-HL-<year>-<6 digits>` |
| `scenario_id` | `H01`–`H10` |
| `case_status` | `sealed` |
| `language` | `sk` |
| `product`, `terms_version` | Product code; the version of the terms the claim was decided under |
| `event_year`, `outcome_bucket` | Year of the event; the decision's bucket (all ten are `approved` in this release) |
| `policy_number`, `notification_number`, `claim_number` | Identifiers of the claim |
| `insured` | `{full_name, birth_date, birth_number}` |
| `claim` | `{event_date, notified_on, cause, claimed_risks[], diagnosis_code, diagnosis_name, provider {doctor, facility, address}, incapacity_from, incapacity_to, treatment_from, treatment_to, hospitalized, valuation_item}` |
| `outcome` | `{decision_outcome, decided_on, paid_on, amount, currency, tariffs[]}` — each tariff `{tariff, name, status, days, daily_benefit, amount, paid_from, paid_to, grounds[]}` |
| `case_summary` | The claim in a paragraph of Slovak prose |
| `decision_summary` | The decision in a paragraph of Slovak prose, with the calculation and its grounds |

## Statistics (version 1.0.0)

| Metric | Value |
|---|---|
| Evaluated claim files | 100 (10 named, 49 generated clean, 41 generated with one defect) |
| Closed historical claims | 10 |
| Documents in the claim files | 572 (4–9 per file) |
| Document formats | born-digital PDF 212, scanned PDF 119, JPG 121, PNG 120 |
| Document kinds | 12 |
| Tariffs on the policies | 8 (`COV_201`–`COV_208`) |
| Claimed risks | work incapacity 80, injury treatment 37 (17 files claim both) |
| Cause | illness 51, injury 45, pregnancy 4 |
| Diagnoses / injury items | 22 ICD-10 codes / 10 valuation items |
| Policies with two insured persons | 5 (each notifying two files) |
| Events | 2025–2026 |
| Labelled PII occurrences | 5,331 |

**Defects** (41 files): `date_conflict` 7, `treatment_over_cap` 7, `outside_policy` 6,
`waiting_period` 6, `elimination_not_reached` 6, `missing_document` 3, `incapacity_ongoing` 3,
`exclusion_evidenced` 3.

**Expected outcomes**

| `control_state` | `decision_outcome` | Files |
|---|---|---|
| `ready` | `approved` | 62 |
| `ready` | `denied` | 13 |
| `needs_hitl` | `n/a` (candidate: 11 × `denied`, 8 × `approved`) | 19 |
| `needs_info` | `n/a` | 6 |

**Tariff lines**: 182 — `paid` 121, `denied` 45, `pending` 12, `not_arising` 4.

## Evaluating a system

1. Load `cases`, and `history` if the system looks up precedents.
2. For each file give the system **`input` only** — the structured records and the documents'
   `raw_text` (or files rendered from it).
3. Compare the system's output with `expected` and `labels`.

The first group needs nothing but the dataset. The second group measures a system that implements
the product's rules; the rules themselves are not part of this release.

| What | Compare | Suggested metric |
|---|---|---|
| Extraction | `expected.facts` against the system's extracted facts | exact match per field |
| Document classification | `labels.documents[].kind` | accuracy |
| PII detection | `labels.documents[].pii_entities` | entity-level precision / recall per type |
| Precedent lookup | `labels.related_history` | recall |
| *Routing* | `control_state` | accuracy, confusion matrix over `ready` / `needs_info` / `needs_hitl` |
| *Decision* | `decision_outcome`; for `needs_hitl` the recommendation against `decision_outcome_candidate` | accuracy |
| *Classification* | per tariff `cause_code`, `claim_type` (`cause_desc` is the label's text, not a target) | accuracy |
| *Amount* | `amount`; per tariff `status`, `days`, `amount`, `paid_from`, `paid_to` | exact match |
| *Missing items* | `missing[]` | set precision / recall |

## Key insurance concepts

| Slovak | English | Meaning here |
|---|---|---|
| *pracovná neschopnosť* (PN) | temporary work incapacity | a doctor certifies that the insured cannot work; risk `RISK_023` |
| *denná dávka* | daily benefit | the amount paid per day of incapacity, set per tariff on the policy |
| *karenčná doba* | elimination period | how long the incapacity must last before the tariff pays: 29, 57 or 85 days |
| *dvojica taríf* | tariff pair | e.g. 29+: one tariff pays from day 29, the other pays days 1–28 once day 29 is reached, as the cover names say; a policy may carry the whole pair or one half |
| *čakacia doba* | waiting period | time from the start of a rider during which an illness is not covered |
| *liečenie úrazu* | injury treatment | benefit for the days an injury is treated, capped per type of injury; risk `RISK_046` |
| *výluka* | exclusion | a situation the insurance does not cover (e.g. alcohol); proven only by a document |
| *poistník / poistený* | policyholder / insured | who holds the policy / whose health is insured — one policy can insure several people |
| *rodné číslo* | birth number | Slovak personal identifier; synthetic here |
| PP-ASTRA-PU, SOP-ASTRA-PU | terms, adjuster procedure | the product's insurance terms and the insurer's claim procedure, which `grounds` cite; not part of this release |
| HITL | human in the loop | a human adjuster decides the claim |

## Data quality and ethics

- **Synthetic end to end.** No record, text or identifier comes from a real person, policy or
  claim. The insurer and its product are fictional; the documents are written to read like the real
  thing a Slovak life insurer's client attaches.
- **Validated.** Every release passes the generator's validation (see *How the dataset was
  created*), including a check that no marker of a real insurer appears in the data.
- **Consistent by construction.** Policies, documents and expected outcomes come from one set of
  facts; the expected outcome is computed, not labelled.
- **Personal data is labelled, not removed.** The documents contain the synthetic personal data a
  real claim file would, so anonymisation can be tested; every planted value is in `pii_entities`.

## Limitations

- **The product's rules are not published.** The terms, the claim procedure and the product
  catalogue the expected outcomes follow are not part of this release; `grounds` cite them by
  article. Routing, decision and amount labels are therefore reference answers for a system that
  implements those rules, not something the rules can be learned from.
- **Text, not images.** The documents are published as their text. Layout, stamps, handwriting,
  scan noise and skew are not in this release, so it measures reading the content, not OCR.
- **One product, two risks, one country.** Slovak life riders for work incapacity and injury
  treatment only; no non-life, no Czech variant.
- **Small and designed.** 100 files cover the rules and the failure modes on purpose; they say
  nothing about how often each occurs.
- **Situations not in this release.** No file ends in a `partial` decision, claims a risk the policy
  does not carry, leaves the cause unstated, reaches the 730-day limit, or has a cover ending during
  the incapacity. Every exclusion file is the same kind of event (a road accident on a bicycle
  proven by a police record), pregnancy has one course, and a clean work-incapacity claim is always
  notified while the incapacity is still running.
- **Known artefact.** Work-incapacity certificates restate their period in one plain sentence at the
  end (*„Pracovná neschopnosť: od … do …"*), which real certificates do not.

## Getting started

```bash
git clone https://github.com/InnovAIte-Slovakia/WP7-Comprehensive-dataset-with-cleaned-data-for-life-insurance-claims.git
```

With the Hugging Face `datasets` library:

```python
from datasets import load_dataset

cases = load_dataset("json", data_files="data/cases.jsonl", split="train")
history = load_dataset("json", data_files="data/history.jsonl", split="train")

case = cases[0]
system_input = case["input"]            # give this — and only this — to the system under test
for doc in system_input["documents"]:
    print(doc["doc_id"], doc["format"], doc["raw_text"][:80])

print(case["expected"]["control_state"], case["expected"]["amount"])
```

With pandas:

```python
import pandas as pd

cases = pd.read_json("data/cases.jsonl", lines=True)
history = pd.read_json("data/history.jsonl", lines=True)
```

Or with the standard library only:

```python
import json

with open("data/cases.jsonl", encoding="utf-8") as f:
    cases = [json.loads(line) for line in f]
```

## Version history

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-09-28 | Initial release: 100 evaluated claim files and 10 closed claims of the fictional insurer ASTRA |

## Citation

```bibtex
@misc{astra_synthetic_claims_2026,
  title        = {ASTRA Synthetic Life-Insurance Claims (Slovak)},
  author       = {{Asseco Central Europe, a.s.}},
  year         = {2026},
  version      = {1.0.0},
  howpublished = {GitHub, InnovAIte Slovakia},
  url          = {https://github.com/InnovAIte-Slovakia/WP7-Comprehensive-dataset-with-cleaned-data-for-life-insurance-claims},
  license      = {Apache-2.0}
}
```

## License

The dataset is released under the [Apache License 2.0](LICENSE).
