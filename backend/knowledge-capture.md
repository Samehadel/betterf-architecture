# Knowledge capture and classification (BTF-13)

Implemented against architecture baseline `cdd59b639c3136a6311640b7f403a5aca60af5ba`.
Requirements: [BTF-13](https://linear.app/betterf/issue/BTF-13/create-and-classify-a-knowledge-ticket).
The product owner delegated outstanding product choices on 2026-10-18 and selected Spring AI ChatClient, OpenAI `gpt-5.4-nano`, optional Title/Capture preview and acceptance, and deferral of content sanitization.

## Ownership and contracts

The `knowledge` module owns published tickets and classification work. It obtains the active account and organization through `identity.api.service.IdentityService.current`; it never reads identity repositories. Both active members and administrators can create, refine, and read tickets within their organization. Organization and author fields are derived server-side. Cross-organization reads return 404.

- `POST /api/knowledge/tickets`: `{submissionId, content}`; 201 with the normal `{data,error}` envelope. Creation commits content, author ID/name, organization ID, creation time, and PENDING classification in one transaction. Publication never calls AI.
- `GET /api/knowledge/tickets/{id}`: organization-scoped published view, content, creation metadata, stored attributes, and classification status.
- `POST /api/knowledge/tickets/refine`: content only; optional suggested `{title,capture}`. Failure returns 503 `REFINEMENT_UNAVAILABLE` and never changes a ticket.

All operations require authentication; mutations require CSRF. Services independently enforce active membership. Creation uses a client-generated submission UUID. A transaction advisory lock and `(AUTHOR_ID,SUBMISSION_ID)` unique key serialize retries. Reusing a key with different content returns 409 `SUBMISSION_CONFLICT`; repeating the same content returns the existing publication. The UI freezes editing on uncertain responses until an unchanged retry resolves publication.

## Content contract

Content is `{title,capture,template,details}`. Title and Capture are plain text limited to 200 and 1,000 UTF-16 code units. Details use a version-one JSON block array, limited to 200 blocks and 30,000 UTF-16 code units across block text and link URLs; JSON syntax, block type, and language metadata do not count. Required content ignores Unicode whitespace and format characters when testing emptiness. No user HTML is stored or executed.

Block types: `paragraph`, `heading`, `bulletList`, `orderedList`, `link`, and `code`. Each has text; lists use one item per line. Links require a meaningful label and absolute HTTP(S) URL without user credentials. Only links have URLs; only code blocks have language metadata. Supported languages: plaintext, Java, TypeScript, JavaScript, Python, SQL, Bash, JSON, YAML, HTML, CSS, C#, C++, Go, Rust, and Kotlin.

Creator-selected writing templates are Knowledge, Coding, Problem, Decision, Accomplishment, and Learning. All require substantive Details; Coding also requires a nonempty code block. Other templates provide contextual writing guidance, without imposing artificial required subsections. Templates are composition aids and may differ from inferred classification.

## Classification and external AI

The immutable predefined catalog uses stable IDs grouped by `type`, `area`, and `technology`; its authoritative implementation is `AttributeCatalog.ALL`. Creators do not enter or approve attributes. The initial catalog covers the six writing subjects, eight software/product areas, and ten common technology labels, including `technology:other`. Catalog administration is out of scope.

Spring AI 2.0.1 ChatClient uses provider-native structured output and the OpenAI integration. Both operations send all three content fields and the template; classification additionally receives the catalog. They never send database records, ticket IDs, author names/emails/IDs, organization metadata, timestamps, or submission identifiers. No chat memory, tools, or retrieval is configured. Provider storage is disabled. Prompts treat user content as data and forbid invented facts or changed technical meaning; the creator reviews suggestions before acceptance.

**Explicit scope change:** the owner requested no free-text/code sanitization for now. Consequently, organization information embedded by a creator can reach OpenAI. This supersedes the original story's content-redaction acceptance criterion for this implementation. The UI discloses content transfer. Keys remain server-side through `OPENAI_API_KEY`; no real key is committed. Without a valid key, refinement fails safely and publication remains available.

A scheduled worker claims durable database work with `FOR UPDATE SKIP LOCKED`, commits a two-minute lease and random lease token, then calls AI outside the transaction. Completion validates every catalog ID and fences updates by lease token. No match completes with an empty attribute list. Invalid output and provider errors retry after 30 then 60 seconds, with three total attempts; terminal failure preserves publication. Abandoned leases recover automatically, including terminal failure after an abandoned third attempt. Up to ten claims are processed per run. Scheduling uses two threads so AI work does not occupy the only scheduler thread. Classification attempts expose a low-cardinality Micrometer counter without user metadata.

The additive Liquibase migration creates `KNOWLEDGE_TICKET`; existing tables and contracts remain compatible. Removing the feature does not justify dropping captured knowledge; restore a tested backup if rollback is required.

## Operations

Actuator exposes `/actuator/health`, `/actuator/health/liveness`, `/actuator/health/readiness`, `/actuator/info`, `/actuator/metrics`, and `/actuator/prometheus`. Health probes are public and unwrapped; readiness checks the database, and liveness does not depend on database or LLM availability. Aggregate health also reflects configured SMTP health. Metrics, info, and Prometheus require an active administrator session and remain unwrapped. HTTP/JVM metrics are available immediately; `betterf.knowledge.classification` appears after classification attempts, with `result=complete|failed_attempt`. Prometheus uses that counter's normalized `_total` name. No organization IDs or content are metric tags.

The deployment proxy forwards Actuator requests to Spring Security and allows 40 seconds for API responses; Spring AI requests have a 30-second timeout and no provider retries. The browser allows 35 seconds for refinement. No deployment or monitoring-server installation is part of this change.
