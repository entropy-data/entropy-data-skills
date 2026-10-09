---
name: pii-exposure-scanner
description: "Governance and security. Scans data contracts and uploaded example data in Entropy Data for personal data (emails, phone numbers, names, addresses, birth dates, IBANs, card numbers, national IDs), flags fields that are not classified accordingly, finds real-looking personal data in example data, and lists who has access to each exposure, with masked sample values. Use when a governance lead, privacy officer, or security team asks where personal data is, wants a PII or GDPR exposure check, or a daily privacy scan."
---

# PII Exposure Scanner

**Persona:** governance lead, privacy officer, security team
**Trigger:** daily or weekly, or before an audit
**Output:** ranked exposures with field, product, team, masked sample, and the consumers who can access it

Read-only. Never print an unmasked personal value, not in chat and not in files.

## Step 0: Connection

Follow the `entropy-data-cli` skill, Step 2.

## Step 1: Collect

Follow "Collecting data at scale" in the `entropy-data-cli` skill: one `export dir` of `dataproducts,datacontracts,access,policies,classification-schemes,example-data` into a scratch directory.

The example data in the export may contain real personal data. Never print or read it raw. All scanning in Step 3 is done by a script that outputs only masked matches. Delete the scratch directory at the end.

Classifications and PII tags: see "Where things live" in the `entropy-data-cli` skill. If an organization policy assigns personal data to a specific classification, that classification is the required one. Otherwise treat the upper half of the scheme (by `level`) as sensitive, for example `confidential` and `restricted`. If a policy makes classification optional, a PII tag is the primary marker and missing classification is a lower-ranked gap. A bare `pii` tag means `pii:true`; other markers (for example `sensitive`) count only if the organization's tags or policies define them so. If unclear, ask the user once.

Example data: see "Where things live". The script must flatten nested objects and lists, and use the port's `name` to join to agreements.

## Step 2: Scan contract fields

For every property in every contract schema, including nested ones, flag possible personal data by name, `businessName`, and only specific phrases in descriptions ("date of birth", not "customer"): email, phone, mobile, first/last/full/customer name, address, street, zip/postal code, birth, gender, nationality, iban, account number, card, ssn, national id, passport, tax id, salary, health, diagnosis, ip address, device id, user agent, session id, cookie, location, latitude/longitude. Match ip address before address. Exclude reference ids and type lookups (names ending in `id`, `type`, `style`, and a bare `name` in a schema whose name ends in `type`), and count a bare `name` or `address` only in a person or customer context. Report quasi-identifiers and pseudonymized fields (birth year, zip code, hashed or truncated emails) as lower-ranked findings.

A finding is raised when such a field:

- has no classification and no PII tag ("unclassified"), only a PII tag ("tagged but not classified"), or a classification below the required one, or
- is classified as sensitive, and a policy explicitly requires something for such fields (for example a retention period or a usage restriction in the contract), and the contract does not provide it. Quote the policy sentence. Policies can also set rules for the consumers, such as a purpose that justifies personal data: check those in Step 4 against the required classification, not only the actual one, and say so. Rules that the data cannot verify (for example data residency when servers name no region) are "not verifiable".
- sits on a port described as without personal data (direct identifiers, or quasi-identifiers that together identify a person).

Contract property `examples` are visible to everyone like example data: scan them with the Step 3 script as well.

## Step 3: Scan example data

Example data is shown to everyone who can see the product, so real personal data in it is a leak in itself. Write a script that checks values with these patterns and prints only the product, the column, the kind, the match count, and up to three masked samples:

| Kind | Pattern (simplified) |
|---|---|
| Email | `[^@\s]+@[^@\s]+\.[a-z]{2,}` |
| Phone | `\+[0-9][0-9 ()\-/]{7,}`, or any digit sequence of 7+ in a column whose name suggests phone |
| IBAN | `[A-Z]{2}[0-9]{2}[A-Z0-9]{11,30}` with a valid checksum |
| Card number | 13 to 19 digits passing the Luhn check |
| Birth date | dates in a column whose name suggests birth |
| IPv4 | `\b(\d{1,3}\.){3}\d{1,3}\b` in a column whose name suggests ip or address, with each octet at most 255 |
| Name, street address | non-empty text in a column whose leaf key (not the flattened path) suggests a person's name or a street address; city, state, and zip are quasi-identifiers |

Judge synthetic values with aggregate signals computed by the script, never by reading values: reserved domains (`example.com`, `.test`, `.invalid`), `test` or `foo`, fictional phone numbers (for example `555-01xx`), documentation IP ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`), placeholder names such as "John Doe", whether names reappear in the email local part. These are low confidence; judge each column, since one synthetic column does not make the row synthetic, and call the result heuristic. Values that are already masked (containing `***`) are not leaks. Report both separately.

Also compare the example columns with the contract's fields: personal data in columns the contract does not declare (for example on a port described as without personal data) is a finding, at medium rank when the values are masked.

## Step 4: Who can see it

For each exposure, list the active access agreements on the product's output port: consumer, consumer type, purpose. Check each purpose against the contract's `description.usage` and `description.limitations` and against policies on consumers. Rank example data leaks first (visible to everyone), then exposures with many consumers, individual users as consumers, vague purposes, or purposes that conflict with the limitations. Contracts that no product references have no consumers: list them last. Where a consuming product is itself consumed, note that transitive exposure without expanding further. In the chat summary, count individual users ("2 individual users"); put their identities only in the report file.

## Step 5: Output

Ranked list, highest risk first. Team is the product's team, else the contract's `team.name`, else "(no team)":

| Rank | Product | Team | Contract / schema / field | Finding | Masked sample | Consumers |
|---|---|---|---|---|---|---|

Mask samples by keeping the shape only, for example `j***@e***.com`, `+49 *** *** **12`, `DE** **** **** **** **34`, `J*** D***` for names, `**** **-**-**` for dates.

Then a summary: number of possible PII fields, how many are unclassified, example data entries with real-looking personal data, and the teams with the most findings. Map each finding type to a fix the owning team can make: classify the field, replace example data with synthetic rows, restrict terms of use.

Mark field-name findings as "possible": the data owner decides.
