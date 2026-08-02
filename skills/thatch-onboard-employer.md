---
name: Onboard an employer onto Thatch ICHRA benefits
description: >-
  End-to-end flow to bring an employer onto Thatch for Platforms: create the
  employer, add their employees, launch the hosted onboarding iframe, then
  retrieve payroll deductions each month.
api: openapi/thatch-partners-openapi.yml
operations:
  - createEmployer
  - createEmployee
  - createEmployerOnboardingSession
  - listDeductions
  - listEnrollments
---

# Onboard an employer onto Thatch ICHRA benefits

Use the Thatch for Platforms API to enroll an employer in ICHRA health benefits.
All requests go to `https://partners.thatchcloud.com/api/partners/v1` and carry
`Authorization: Bearer <YOUR_API_KEY>` (partner API key from the Thatch dashboard).
Bodies are JSON. This API does not document an idempotency key, so avoid blind
retries on POSTs — check for an existing resource first.

## Steps

1. **Create the employer** — `POST /employers` (`createEmployer`). Supply the
   required fields: `email`, `name`, `business_type` (enum: c_corp, s_corp, llc,
   llp, partnership, sole_proprietorship, non_profit), `ein`, `address_line1`,
   `city`, `state` (USPS code), `zip`. Optionally `dba`, `industry_code` (NAICS),
   `phone_number`, `offers_health_insurance_today`, `coverage_start_date`,
   `pricing.platform_fee`, and `metadata`. The response returns the employer `id`
   (prefix `empl_`). Save it.

2. **Create employees** — `POST /employees` (`createEmployee`) for each worker,
   passing the `employer_id` plus required `first_name`, `last_name`,
   `date_of_birth` (YYYY-MM-DD), and `zip`. Send as much optional data as you
   have (`personal_email`, `work_email`, `employment_subtype`, `pay_type`,
   `start_date`, `dependents[]`, `native_employee_id`) because it feeds the
   employer's quoting process. Each `dependents[]` entry needs `relationship`
   and `date_of_birth`.

3. **Launch employer onboarding** — `POST /employer_onboarding_sessions`
   (`createEmployerOnboardingSession`) with `{ "employer": "empl_..." }`. The
   response includes a `claim_url` and `expires_at`. Embed the `claim_url` in a
   Thatch onboarding iframe. Listen for the `THATCH_REDIRECT` window message from
   origin `https://app.thatch.com`, validate the origin, and redirect the parent
   window to the provided `redirectUrl`. The employer then creates their account,
   connects a bank account, and invites employees inside the iframe.

4. **(Optional) Model pay schedules** — `POST /employers/{employer_id}/pay_schedules`
   (`createPaySchedule`) with `name`, `frequency` (monthly/semi_monthly/bi_weekly/
   weekly), and `bank_closure_strategy` so deductions compute against the right
   cadence. Prefer `first_pay_date`/`second_pay_date` over the deprecated
   `reference_pay_date`/`first_day`/`second_day` fields.

5. **Track enrollments** — `GET /enrollments` (`listEnrollments`), optionally
   filtered by `member_id` or `status` (in_member_cart, member_confirmed,
   submission_processing, carrier_processing, completed, canceled), to confirm
   employees have selected plans.

6. **Retrieve deductions monthly** — `GET /deductions` (`listDeductions`) with the
   required `employer_id`, plus `periods[start_after]`/`periods[end_before]`. The
   list becomes available five days before month-end. Push the returned amounts
   into your payroll provider as pre-tax (`s125_pretax`) deductions.

## Conventions

- **Pagination**: list endpoints take `page[number]` and `page[size]` (1-1000,
  default 20) and return `{ data: [...], pagination: {...} }`. The pay-schedules
  list returns a bare array.
- **Money**: amounts are `{ amount: <integer minor units>, currency_code: "USD" }`.
- **Errors**: failures return `{ "message": "..." }` (not RFC 9457); a missing or
  invalid key returns `401 {"message":"Unauthorized"}`.
- **Metadata**: attach `metadata` key-value strings to employers and employees;
  unset a key with `""`, clear all with `{}`.
