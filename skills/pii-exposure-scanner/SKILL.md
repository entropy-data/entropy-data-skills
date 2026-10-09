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

Work out from the classification schemes which levels count as "sensitive" in this organization. If unclear, ask the user once.

## Step 2: Scan contract fields

For every property in every contract schema, flag possible personal data by name and description: email, phone, mobile, first/last/full name, address, street, zip/postal code, birth, gender, nationality, iban, account number, card, ssn, national id, passport, tax id, salary, health, diagnosis, ip address, device id, cookie, location, latitude/longitude.

A finding is raised when such a field:

- has no classification, or a non-sensitive one, or
- is classified as sensitive, and a policy explicitly requires something for such fields (for example a retention period or a usage restriction in the contract), and the contract does not provide it. Quote the policy sentence.

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

Values from obviously synthetic domains (`example.com`, `test`, `foo`) are low confidence. Report them separately.

## Step 4: Who can see it

For each exposure, list the active access agreements on the product's output port: consumer, consumer type, purpose. Exposures with many consumers, individual users as consumers, or vague purposes rank higher. In the chat summary, count individual users ("2 individual users"); put their identities only in the report file.

## Step 5: Output

Ranked list, highest risk first:

| Rank | Product | Team | Contract / schema / field | Finding | Masked sample | Consumers |
|---|---|---|---|---|---|---|

Mask samples by keeping the shape only, for example `j***@e***.com`, `+49 *** *** **12`, `DE** **** **** **** **34`.

Then a summary: number of possible PII fields, how many are unclassified, example data entries with real-looking personal data, and the teams with the most findings. Map each finding type to a fix the owning team can make: classify the field, replace example data with synthetic rows, restrict terms of use.

Mark field-name findings as "possible": the data owner decides.
