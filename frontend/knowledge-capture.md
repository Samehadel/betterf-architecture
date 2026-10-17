# Knowledge Writing Studio (BTF-13)

The capture route `/knowledge/new` is available from organization home to active members and administrators. `/knowledge/tickets/:id` displays a published ticket. Both routes load the shared AuthStore and direct anonymous users to login. Backend authorization is authoritative. Route-scoped KnowledgeStore owns API operation state; the form owns unsaved content locally.

The selected Linear Writing Studio mockup governs the charcoal/lavender layout: writing area, contextual sidebar, and clear publish action. Mobile collapses the sidebar below the form. The block editor provides headings, paragraphs, lists, HTTP(S) links, and language-selected code, with add/remove/reorder controls and a rendered preview. Published code is highlighted through highlight.js; Angular escapes text and sanitizes rendered highlighting. Lists render with native list semantics.

Writing templates provide guidance; Coding adds a code block without removing existing content. The Details AI control and controls beside free-text fields all request the same optional Title/Capture refinement using complete content. Suggestions show separately; accept changes only Title/Capture, discard preserves the original, and editing invalidates old suggestions. Editing is paused while a refinement is running so late results cannot overwrite newer text. Details never get rewritten.

Publishing is disabled until all required content validates. Submission and refinement failures preserve local content. An uncertain publication response pauses editing and allows an unchanged retry with the same submission key, preventing duplicate tickets after lost responses. Browser reload/close uses `beforeunload`; Angular navigation uses a confirmation guard. No draft or form content is saved to browser storage.

Successful publication opens the ticket with confirmation. Ticket views show author, time, rendered content, classification state, and catalog IDs. Pending/processing/retrying states poll every five seconds; terminal states stop polling. Errors offer retry without hiding a previously loaded ticket. There is no approval, manual classification, browsing, search, or editing workflow in this story.

See the [backend contract](../backend/knowledge-capture.md) for storage, validation/counting rules, catalog, job semantics, and the owner's explicit deferral of content sanitization.
