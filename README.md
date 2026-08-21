# FINRA U4 Exam Amendment API Guide

## 1. Purpose

This document explains how to submit an **Exam Information update for an existing individual** using the FINRA U4 Submission API.

This is **not** a new U4 initial submission. It is a **U4 Amendment** that requests one or more exams for an individual who already has an Individual CRD Number.

> Important: The U4 API can be used to request/add an exam, such as Series 65. It cannot be used to manually change an exam result, grade, appointment status, confirmation number, or exam-window dates. FINRA and its testing systems maintain those values.

## 2. Business Scenario

Example:

- An individual already exists in FINRA CRD.
- The individual has CRD Number `71372`.
- The firm wants to request the Series 65 exam.
- The FINRA exam code is `S65`.
- The firm submits a U4 filing with `filingType` set to `amendment`.

## 3. End-to-End Process

1. Select the existing individual.
2. Obtain and verify the individual's CRD Number and date of birth.
3. Select the requested exam from a controlled list.
4. Map the displayed exam name to the appropriate FINRA exam code.
5. Build the U4 Amendment payload.
6. Include the authorized firm signatures.
7. Validate the amendment with FINRA.
8. Correct any validation errors.
9. Submit the amendment.
10. Save the FINRA submission ID returned by the API.
11. Poll the GET Submission endpoint for the processing result.
12. Store the final FINRA filing ID, status, errors, and timestamps.

## 4. Information Required

| Field | Example | Notes |
|---|---:|---|
| Individual CRD Number | `71372` | Existing individual's FINRA CRD Number |
| Date of Birth | `1958-09-11` | Must match FINRA data and the active schema format |
| Filing Type | `amendment` | This is not an initial U4 |
| Filing Status | `validate` or `submitted` | Validate first, then submit |
| Exam Code | `S65` | Use a valid FINRA enumeration |
| Firm Signature | `John Compliance` | Authorized firm signatory |
| Firm Signature Date | `2026-08-21` | Use the date format required by the schema |
| Amended Firm Signature | `John Compliance` | Signature representation for the amendment |
| Amended Signature Date | `2026-08-21` | Amendment signature date |

Never let users enter arbitrary exam codes. Display a controlled dropdown populated from the current FINRA schema or a maintained reference table.

Example mappings:

| Display Name | FINRA Code |
|---|---|
| Series 63 | `S63` |
| Series 65 | `S65` |
| Series 66 | `S66` |

These are examples, not a complete enumeration. Validate all exam codes against the current FINRA U4 schema.

## 5. FINRA U4 Submission Endpoint

```http
POST https://api.finra.org/v1/submissions/filings/registration/u4
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json
Accept: application/json
```

The access token must be generated through the FINRA Identity Platform using authorized credentials. Never expose the client secret or access token in browser code, logs, screenshots, or source control.

## 6. Validate an Exam Amendment

Validation is the recommended first action. Set `filingStatus` to `validate` so FINRA creates a draft and performs its completeness checks without completing the final filing.

```json
{
  "filing": {
    "metadata": {
      "individualCrdNumber": 71372,
      "filingType": "amendment",
      "filingStatus": "validate",
      "dateOfBirth": "1958-09-11"
    },
    "operations": [
      {
        "op": "add",
        "path": "/examsInformation",
        "value": {
          "examCode": [
            "S65"
          ]
        }
      },
      {
        "op": "add",
        "path": "/signatures",
        "value": {
          "firmSignatoryRepresentations": {
            "firmSignature": "John Compliance",
            "firmSignatureDate": "2026-08-21"
          },
          "amendedFirmSignatoryRepresentations": {
            "amendedFirmSignature": "John Compliance",
            "amendedFirmSignatureDate": "2026-08-21"
          }
        }
      }
    ]
  }
}
```

The `operations` array is a partial update. It tells FINRA to apply only the described changes to the information FINRA already holds and create a U4 Amendment.

### Meaning of the exam operation

```json
{
  "op": "add",
  "path": "/examsInformation",
  "value": {
    "examCode": ["S65"]
  }
}
```

- `op: add` requests that new information be added.
- `path: /examsInformation` targets the U4 Exam Information section.
- `examCode: ["S65"]` requests the Series 65 exam.

This structure follows FINRA's published **U4 Amendment to add Exam** example. Always validate the final payload against the current U4 JSON schema because schema rules and permitted enumerations can change.

## 7. Requesting Multiple Exams

When permitted by the current schema and FINRA business rules, the exam array can include more than one code:

```json
{
  "op": "add",
  "path": "/examsInformation",
  "value": {
    "examCode": [
      "S63",
      "S65"
    ]
  }
}
```

Do not assume that every exam combination is allowed. FINRA evaluates eligibility, prerequisites, existing exam history, registration requests, and other completeness rules.

## 8. Submission Response

The POST request returns a unique submission ID. A simplified response looks like this:

```json
{
  "id": "9084b744-7639-11ed-a1eb-0242ac120002",
  "group": "registration",
  "name": "u4"
}
```

Store the returned `id` immediately. It is the tracking identifier required for subsequent GET, PUT, PATCH, or eligible DELETE operations.

Suggested internal fields:

| Internal Field | Example |
|---|---|
| `finraSubmissionId` | `9084b744-7639-11ed-a1eb-0242ac120002` |
| `individualCrdNumber` | `71372` |
| `filingType` | `AMENDMENT` |
| `requestedExamCodes` | `S65` |
| `submissionStatus` | `DRAFT` |
| `resultStatus` | `PROCESSING` |
| `finraFilingId` | Initially null |
| `lastStatusCheckAt` | Timestamp |
| `errorDetails` | JSON or structured child records |

## 9. Check Validation or Processing Status

```http
GET https://api.finra.org/v1/submissions/filings/registration/u4/9084b744-7639-11ed-a1eb-0242ac120002
Authorization: Bearer <ACCESS_TOKEN>
Accept: application/json
```

U4 processing is asynchronous. Do not treat a successful POST as confirmation that the amendment was processed. Poll the GET endpoint using a controlled schedule and stop when the request reaches a final result.

Common result statuses include:

| Result Status | Meaning | Application Action |
|---|---|---|
| `PROCESSING` | FINRA is still processing | Wait and check again |
| `PROCESSED` | Filing processed successfully | Store filing ID and close workflow |
| `FAILED_VALIDATION` | Completeness or schema validation failed | Display errors and allow correction |
| `FAILED` | Processing failed | Record failure and route for investigation |
| `DELETED` | Draft was deleted | Close or archive the local request |

FINRA documentation states that asynchronous U4 processing may take approximately 5–15 minutes. Avoid aggressive polling.

## 10. Successful Result Example

```json
{
  "id": "9084b744-7639-11ed-a1eb-0242ac120002",
  "group": "registration",
  "name": "u4",
  "filing": {
    "metadata": {
      "filingStatus": "submitted",
      "filingType": "AMENDMENT",
      "individualCrdNumber": 71372,
      "resultStatus": "PROCESSED",
      "filingId": "84435978",
      "resultStatusDesc": {
        "message": "Successfully processed",
        "errors": []
      }
    }
  }
}
```

The `id` is the Submission API tracking ID. The `filingId` is assigned after FINRA processes the filing and is the identifier associated with the filing in FINRA Gateway.

## 11. Validation Failure Example

```json
{
  "filing": {
    "metadata": {
      "filingStatus": "draft",
      "filingType": "AMENDMENT",
      "individualCrdNumber": 71372,
      "resultStatus": "FAILED_VALIDATION",
      "resultStatusDesc": {
        "message": "Validation failed",
        "errors": [
          {
            "errorType": "VALIDATION_ERROR",
            "message": "Invalid or ineligible exam request"
          }
        ]
      }
    }
  }
}
```

The exact error structure and messages may differ. Store the raw FINRA response securely for audit and troubleshooting, but show the user a readable field-level message.

## 12. Submit After Successful Validation

After validation succeeds and any required internal approval is complete, change the status to `submitted` and update the existing draft using its submission ID, following the operation supported by the current FINRA documentation and environment.

Conceptually:

```json
{
  "filing": {
    "metadata": {
      "filingStatus": "submitted"
    }
  }
}
```

Do not create duplicate filings by repeatedly issuing new POST requests after a draft already exists. Preserve and reuse the submission ID for that workflow.

## 13. What Cannot Be Updated Through This U4 Exam Request

The application must not provide editable controls for these system-maintained values:

- Exam result or grade
- Passed/failed status
- Exam score
- Appointment status
- Confirmation number
- Window start date
- Window end date
- Waiver decision
- Official result timestamp

For example, do not create a U4 payload like this:

```json
{
  "examStatus": "PASSED",
  "score": 80,
  "windowBeginDate": "2026-09-01"
}
```

Those details must be retrieved from the appropriate FINRA registration datasets after FINRA and the testing provider process the request.

## 14. Suggested Application API Design

Your application can expose the following internal endpoints:

### Validate exam request

```http
POST /api/u4/exams/validate
```

Example application request:

```json
{
  "individualCrdNumber": 71372,
  "dateOfBirth": "1958-09-11",
  "examCodes": ["S65"],
  "firmSignatoryName": "John Compliance"
}
```

### Submit validated amendment

```http
POST /api/u4/exams/{submissionId}/submit
```

### Retrieve status

```http
GET /api/u4/exams/{submissionId}/status
```

### Retrieve request history

```http
GET /api/individuals/{individualCrdNumber}/exam-requests
```

## 15. Suggested Components

| Component | Responsibility |
|---|---|
| Exam Request UI | Select individual and exam codes |
| Local Validation Service | Check required fields and allowed formats |
| Exam Code Reference | Maintain permitted exam codes and labels |
| U4 Payload Builder | Build metadata, operations, and signatures |
| FINRA Authentication Client | Obtain and securely cache access tokens |
| FINRA Submission Client | Call POST, GET, PUT, or PATCH endpoints |
| Status Poller | Check asynchronous processing results |
| Error Mapper | Convert FINRA errors into readable messages |
| Audit Service | Record approvals, payload hashes, statuses, and actors |
| Notification Service | Notify compliance users of success or failure |

## 16. Recommended Data Model

### `u4_exam_request`

| Column | Suggested Type | Purpose |
|---|---|---|
| `id` | UUID | Internal primary key |
| `individual_crd_number` | BIGINT | Existing FINRA individual |
| `date_of_birth_encrypted` | Encrypted date/string | Sensitive identity verification value |
| `status` | VARCHAR | Local workflow status |
| `finra_submission_id` | UUID/string | Submission API tracking ID |
| `finra_filing_id` | VARCHAR | Assigned after processing |
| `result_status` | VARCHAR | Latest FINRA result status |
| `result_message` | TEXT | Latest readable result |
| `requested_by` | User identifier | User initiating the request |
| `approved_by` | User identifier | Compliance approver |
| `created_at` | Timestamp | Creation time |
| `updated_at` | Timestamp | Last update |
| `submitted_at` | Timestamp | Final submission time |

### `u4_exam_request_item`

| Column | Suggested Type | Purpose |
|---|---|---|
| `id` | UUID | Primary key |
| `u4_exam_request_id` | UUID | Parent request |
| `exam_code` | VARCHAR | Example: `S65` |
| `exam_name` | VARCHAR | Example: Series 65 |
| `item_status` | VARCHAR | Requested, processed, or rejected |

### `u4_exam_status_history`

| Column | Suggested Type | Purpose |
|---|---|---|
| `id` | UUID | Primary key |
| `u4_exam_request_id` | UUID | Parent request |
| `result_status` | VARCHAR | Status received from FINRA |
| `response_payload_encrypted` | Encrypted JSON | Raw response for controlled audit use |
| `received_at` | Timestamp | Status-check timestamp |

## 17. Security and Compliance Requirements

- Keep FINRA Client ID and Client Secret in a secret manager.
- Obtain access tokens on the server, never in the browser.
- Encrypt date of birth and other sensitive personal data at rest.
- Use TLS for all communications.
- Mask personal data in application and infrastructure logs.
- Restrict exam submission to authorized registration/compliance roles.
- Separate request creation and approval when firm policy requires maker-checker control.
- Record who created, validated, approved, submitted, retried, or cancelled a request.
- Do not log Authorization headers, access tokens, or client secrets.
- Prevent duplicate submission using an idempotency strategy in the application.
- Retain payloads and responses according to the firm's regulatory record-retention policy.
- Apply least-privilege access to raw FINRA responses and audit data.

## 18. Error Handling Recommendations

Handle at least these situations:

1. Missing or invalid CRD Number.
2. Date of birth does not match the individual.
3. Invalid exam code.
4. Duplicate exam request.
5. Individual is not eligible for the requested exam.
6. Required firm signature is missing.
7. Token expired or authentication failed.
8. FINRA returns `401`, `403`, `404`, `409`, `429`, or a server error.
9. Request remains in `PROCESSING` longer than the expected period.
10. Network failure occurs after POST and the client does not know whether FINRA received it.

For an uncertain POST result, do not automatically create another submission. First reconcile using stored correlation information, submission history, and the FINRA APIs to avoid duplicates.

## 19. Testing Scenarios

### Positive scenarios

- Submit one valid exam code.
- Submit multiple permitted exam codes.
- Validate first and then submit the same draft.
- Poll from `PROCESSING` to `PROCESSED`.
- Store the returned filing ID.

### Negative scenarios

- Invalid CRD Number.
- Incorrect date of birth.
- Unsupported exam code.
- Empty `examCode` array.
- Missing signatures.
- Malformed date.
- Duplicate exam code in the array.
- Exam already requested or held.
- Ineligible exam combination.
- Expired access token.
- FINRA throttling response.
- Validation warnings requiring review.

### Security scenarios

- Unauthorized user attempts submission.
- User attempts to change the CRD Number after approval.
- Sensitive fields appear in logs.
- Repeated submit-click attempts occur.
- A submitted payload is altered without a new approval.

## 20. Implementation Checklist

- [ ] Obtain access to the FINRA API Console and QA/Test environment.
- [ ] Download and review the current U4 schema and enumerations.
- [ ] Configure secure FINRA authentication.
- [ ] Create the exam-code reference table.
- [ ] Build the existing-individual selection screen.
- [ ] Build the exam selection screen.
- [ ] Implement local required-field validation.
- [ ] Implement the U4 Amendment payload builder.
- [ ] Implement firm-signature handling.
- [ ] Implement validation mode.
- [ ] Store the returned FINRA submission ID.
- [ ] Implement status polling with controlled retry intervals.
- [ ] Map FINRA validation errors to user-friendly messages.
- [ ] Implement compliance approval before final submission.
- [ ] Implement final submission using the existing draft ID.
- [ ] Store the FINRA filing ID and final result.
- [ ] Implement audit history and sensitive-data masking.
- [ ] Test in the FINRA QA/Test environment before production.

## 21. Official References

- [FINRA API Developer Documentation](https://developer.finra.org/docs)
- [FINRA Submission API Overview](https://developer.finra.org/products/submission-api)
- [FINRA Validate Feature](https://developer.finra.org/news/validate-feature-submission-api)
- [FINRA API Support and Test Environment FAQ](https://developer.finra.org/support)

## 22. Final Notes

The smallest correct implementation is:

1. Accept CRD Number, date of birth, and exam code.
2. Create a U4 `amendment` using an `add` operation on `/examsInformation`.
3. Add the authorized firm-signature information.
4. Validate the request.
5. Correct any reported errors.
6. Submit the validated draft.
7. Poll until FINRA returns `PROCESSED` or a final failure.
8. Save the FINRA filing ID and complete audit history.

Before production use, confirm every payload field, date format, enum value, signature requirement, and endpoint behavior against the current FINRA U4 schema and the firm's FINRA QA environment.


<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/4ef4068e-d33c-4590-b229-d73045d00cba" />

