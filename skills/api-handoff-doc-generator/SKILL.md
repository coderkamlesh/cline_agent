---
name: api-handoff-doc-generator
description: Generates frontend API handoff documentation from recently created or
  modified backend endpoints. This skill should be used when the user asks for API
  documentation, frontend handoff docs, API contract, "API list for frontend",
  "postman docs", or wants the newly built backend APIs written up for the UI team.
  Also trigger on Hinglish requests such as "frontend ke liye api de de",
  "frontend ko dena hai api de de", "api docs bana de", "frontend ke liye api
  documentation bana do", or "api bana de frontend ke liye".
agent_created: true
allowed-tools: Read, Grep, Glob, Bash, Write
---

# API handoff doc generator

## When to use

- The user wants documentation of recently created or modified APIs for the frontend team.
- The user asks for API documentation, frontend handoff, an API contract, or API list for the UI.

Trigger phrases the user actually says, in Hinglish:

- "frontend ke liye api de de"
- "frontend ko dena hai api de de"
- "api docs bana de"
- "frontend ke liye api documentation bana do"
- "api bana de frontend ke liye"
- "frontend handoff doc bana do"
- "api contract bana de"

## Core principles

1. Frontend perspective. The frontend does not need internal DB structures, class names,
   ORM entities, or implementation details. It needs the exact URL, request payload, headers,
   realistic response shape, every possible error code, and how to react to each.

2. Contract-grade. Each endpoint document is a self-contained contract. A frontend
   developer should be able to implement the integration without asking a single backend
   question afterwards.

3. Realistic dummy data. Never use lazy placeholders like string, xyz, foo, 1.
   Use domain-realistic values: valid JWT structure (three base64url segments separated
   by dots), ISO 8601 UTC timestamps, 10-digit Indian mobile numbers, valid enum values,
   real-looking ULIDs/UUIDs.

4. Fidelity to the codebase. Extract auth requirements, rate limits, validation rules,
   and error codes from the actual controllers, DTOs, exception handlers, and
   @RestControllerAdvice / global error middleware. Do not invent error codes that the
   backend does not actually emit.

5. Deterministic structure. Every endpoint document follows the same section order so
   frontend devs can scan fast.

## Workflow

1. Identify the scope.
   - If the user names a module ("wallet APIs", "user auth APIs"), restrict the scan to
     that module's controllers or routes.
   - If no module is given, extract newly created or modified endpoints from the recent
     session or from git diff.

2. Analyse the project conventions before writing.
   - Read the exception handler / error middleware to get the exact error envelope shape
     and error code naming convention.
   - Read the security config to get auth schemes (Bearer JWT, session cookie, API key),
     and role/scope requirements.
   - Read the DTOs to get exact field names, types, nullability, and validation constraints.
   - Read application.yml / .env.example / config classes to identify any
     configuration-dependent behavior that changes response shape or limits.
   - If the project has a global response wrapper (Response<T>, ApiResponse, etc.),
     use that. If it does not, write the response shape directly. Do not force an
     envelope that the backend does not emit.

3. Extract endpoint details. For each endpoint capture:
   - HTTP method and path
   - Auth and role/scope requirements
   - Path variables, query parameters, required headers
   - Request body (JSON, form-data, multipart)
   - Every success response variant (e.g. 2FA required vs. session issued)
   - Every error response: HTTP status, error code, cause, retry semantics
   - Any Retry-After, WWW-Authenticate, or custom headers in error responses
   - Configuration variables that alter behavior

4. Generate realistic dummy data. Always include:
   - Valid JWT for access_token (three base64url segments, dot-separated).
   - Opaque string for refresh_token (32+ chars, no dots).
   - ISO 8601 UTC timestamps with Z suffix.
   - ULID or UUID format for IDs (match what the codebase uses).
   - Real enum values actually emitted by the backend.
   - Realistic expires_in values matching the codebase's configured TTLs.

5. Save and format.
   - Default: one file per endpoint at docs/api-handoff/<endpoint-slug>.md
   - If the user asks for a module-level doc: one file at
     docs/api-handoff/<module-name>-apis.md with one top-level heading per endpoint.
   - Create docs/api-handoff/ automatically if it does not exist.
   - Overwrite existing files with the latest data.
   - End with: Saved API handoff docs to docs/api-handoff/<filename>.md
## Output format template

Follow this exact structure. Do not reorder or drop sections. Omit a section only if
it is genuinely not applicable.

For code blocks inside the generated docs: use triple backticks with a language
identifier (http, json, bash). Do not nest code blocks more than one level deep.

### Section order (mandatory)

Every endpoint doc must contain these top-level sections, in this exact order:

1. H1 heading: human-readable endpoint name
2. Intro paragraph: what the endpoint does, who calls it, unusual behavior
3. H2 "Endpoint": method, path, auth, content type, status, cache policy
4. H2 "Request": JSON example, field table, curl example
5. H2 "Flow": numbered post-request behavior (omit only if single-step stateless)
6. H2 "Successful responses": one H3 subsection per response variant
7. H2 "Error response": error envelope, error code table, rate-limit example
8. H2 "Configuration-dependent behavior": env var table (omit if none)
9. H2 "Client handling": numbered integration steps

### Section 3 (Endpoint) - required format

    ## Endpoint

    \`\`\`http
    [METHOD] [path]
    \`\`\`

    **Authentication:** [None | Bearer JWT | Session cookie | API key]
    **Request content type:** application/json
    **Successful response:** [2xx code]
    **Response content type:** application/json; charset=utf-8
    **Cache policy:** no-store

    [Optional strictness note if backend enforces it]

### Section 4 (Request) - required format

    ## Request

    \`\`\`json
    {
      "field1": "realistic-value"
    }
    \`\`\`

    | Field | Type | Required | Description |
    |---|---|---:|---|
    | field1 | string | Yes | Description with validation rules |

    Example request:

    \`\`\`bash
    curl -i -X POST "[base-url]/[path]" \
      -H "Content-Type: application/json" \
      -d '{"field1": "realistic-value"}'
    \`\`\`

### Section 5 (Flow) - required format

    ## Flow

    1. **Case A:** [what happens, side effects, next client step]
    2. **Case B:** [alternative branch]

    [Any configurable defaults that gate these branches]

### Section 6 (Successful responses) - required format

    ## Successful responses

    ### [Variant 1 name]

    \`\`\`http
    HTTP/1.1 [status] [reason]
    Content-Type: application/json; charset=utf-8
    Cache-Control: no-store
    \`\`\`

    \`\`\`json
    {
      "realistic": "response-body"
    }
    \`\`\`

    | Field | Type | Description |
    |---|---|---|
    | field | string | Description |

    ### [Variant 2 name]

    [Same structure as Variant 1]

    [Security notes paragraph if tokens are involved]

### Section 7 (Error response) - required format

    ## Error response

    Every failed request uses this envelope:

    \`\`\`json
    {
      "error": {
        "code": "error_code",
        "message": "Human-readable message."
      }
    }
    \`\`\`

    Clients should branch on error.code, not on the human-readable message.

    ### Possible errors

    | HTTP status | Error code | Cause |
    |---:|---|---|
    | 400 | error_code | Cause |
    | 401 | error_code | Cause |
    | 403 | error_code | Cause |
    | 429 | error_code | Cause |
    | 500 | error_code | Cause |

    Example rate-limit response:

    \`\`\`http
    HTTP/1.1 429 Too Many Requests
    Content-Type: application/json; charset=utf-8
    Cache-Control: no-store
    Retry-After: 42
    \`\`\`

    \`\`\`json
    {
      "error": {
        "code": "too_many_attempts",
        "message": "Too many failed attempts. Try again in 42 seconds."
      }
    }
    \`\`\`

### Section 8 (Configuration-dependent behavior) - required format

    ## Configuration-dependent behavior

    | Variable | Default | Effect |
    |---|---:|---|
    | VAR_NAME | value | Effect on response shape, TTL, or limit |

### Section 9 (Client handling) - required format

    ## Client handling

    1. First integration step
    2. Second step
    3. How to handle errors and retry semantics

## Constraints

- No internal details. Never include internal class names, database column names,
  entity names, or internal method logic.
- Extract, don't invent. Error codes, header names, config variables, TTL values —
  all must come from the actual codebase. If unsure, read the file or ask the user.
- Strict realistic data.
  - Mobile numbers: 10 digits, Indian format.
  - Amounts: 2-3 decimal places when currency.
  - Dates: ISO 8601 UTC with Z suffix.
  - JWTs: three base64url segments separated by dots.
  - IDs: match the codebase's format (ULID, UUID, numeric).
- Consistency across endpoints. Same section order, same table column headers, same
  header block format.
- Confirmation. After saving the file, print:
  Saved API handoff docs to docs/api-handoff/<filename>.md

## Language

Documentation, headings, and descriptions are written in English.