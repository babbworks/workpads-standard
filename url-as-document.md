# The URL-as-Document Model
## A structural shift in how businesses produce, distribute, and present content

**Status:** v1 (2026-05-12) — architectural analysis
**Scope:** Business model, web architecture, template economics

---

## Background

This document describes a pattern that emerged from designing the Workpads template system and note share mechanism. It is not a product pitch. It is a description of what becomes possible — and what changes structurally — when the encoding of content into URL strings is combined with a distributed template registry and a stateless orchestrator page.

---

## The Traditional Model

A business that produces documents — invoices, job reports, inspection records, delivery notes, proposals — currently chooses between:

**File hosting**: Generate a PDF or HTML file per document, store it on a server, share a URL that points to the file. Storage grows linearly with document volume. Files require versioning, backup, access control, expiry management. Each document is an object that must be managed.

**Platform lock-in**: Use a SaaS tool (Google Docs, DocuSign, Xero) that hosts documents on the vendor's infrastructure. Documents are accessible only while the subscription is active and the vendor exists. Sharing requires the recipient to have or create an account in some cases, or receive a link that depends entirely on the vendor's servers being online.

**Email attachment**: Send the document as an attached file. The file is now in multiple inboxes, impossible to update, disconnected from any live template, and produces storage costs at the recipient end as well.

All three share a common assumption: **the document is a stored object** that must live somewhere before it can be shared.

---

## The Structural Alternative

The Workpads model separates two things that have historically been bundled together:

- **Content**: the data — who, what, when, how much, what happened
- **Presentation**: the template — how that data is displayed, styled, and interacted with

In the URL-as-document model:

- **Content is encoded into the URL string itself**, specifically the fragment (the part after `#`). This data never touches a server. It is compressed, encoded, and carried entirely in the URL.
- **Presentation is a template** identified by a URI. The template is fetched once, cached locally, and reused for every subsequent render. It is small (typically 1–5KB compressed), platform-targeted, and version-controlled.
- **The rendered document** exists only at the moment of viewing, produced by the combination of the two.

There is no stored document object. There is no file to host. The "document" is a URL string that any compatible receiver can render.

---

## What a Share URL Actually Contains

A Workpads note share URL has the form:

```
https://workpads.me/n#n1/{compressed_payload}
```

The fragment contains a compressed JSON object:

```json
{
  "v": 1,
  "text": "Replaced the main valve. Customer confirmed working.",
  "ts": 1747045200000,
  "tpl": "https://acme-plumbing.co.uk/tpls/job-note-v2.json",
  "rec": {
    "job": "Annual boiler service",
    "customer": "J. Okafor",
    "date": "2026-05-12"
  }
}
```

The `tpl` field references the presentation template. The receiving page — `workpads.me/n`, a static HTML file — reads this fragment, fetches or retrieves the template from cache, and renders the document entirely in the browser.

Nothing in this exchange involves a database query, a server-side render, or any infrastructure beyond static file serving for the orchestrator page itself.

A record share URL operates identically, using the existing `workpads.me/p#` path and codec.

---

## The Orchestrator Page

The static page at `workpads.me/n` (or `workpads.me/p` for records) is a shell with one responsibility: receive a fragment, resolve a template, and render content.

Its logic:

1. Parse the URL fragment
2. Decode the compressed payload
3. Extract the template URI
4. Check local cache for the template payload
5. If not cached: fetch from the canonical URI (HTTPS GET, a static JSON file)
6. If fetch fails: walk the fallback chain (superseded versions, then built-in default)
7. Render content using the template inside a sandboxed iframe
8. Display result

The orchestrator page itself never changes. It has no state, no user accounts, no database. Its entire behaviour is determined by what is in the URL fragment. This means it can be hosted on any static hosting provider, cached at CDN edges globally, and served essentially for free at any scale.

The orchestrator is not the product. It is infrastructure — the same way a web browser is infrastructure.

---

## Template JavaScript and the Sandboxed Experience

Templates in their basic forms (schemas A, B, and P) are pure HTML and CSS with string substitution. They cannot execute code. This covers the majority of business document use cases: formatted notes, invoices, job cards, inspection reports, delivery confirmations.

When a template includes JavaScript (Schema C, available only to verified templates), it executes inside a sandboxed iframe:

```
workpads.me/n  (orchestrator — cannot be touched by template code)
    └── <iframe sandbox="allow-scripts">
            ← template HTML + CSS + JS loaded here
            ← note/record data passed in via postMessage
            → template renders, animates, responds to interaction
            → cannot access parent page, cookies, localStorage, or DOM
```

This enables genuinely interactive document experiences: a job report with a print button, a delivery note with an interactive signature field, a proposal with expandable cost sections, a health-and-safety checklist with live completion tracking. All from a URL string, all rendered client-side, all without any server involvement after the initial template fetch.

The sandboxing is not a limitation added for safety — it is the mechanism that makes arbitrary third-party templates safe to execute. The orchestrator page trusts no template code. The content data is passed in through a defined channel. The template cannot exfiltrate it.

---

## Business Document Volume Without Storage Growth

This is the economic break from the traditional model.

Consider a sole trader who generates 800 job records and 2,400 field notes per year. Under traditional file hosting:

- 800 PDF invoices × ~50KB average = 40MB of stored files, growing annually
- Access control, backup, and versioning overhead for each file
- Sharing requires either email attachment or a hosted link with server dependency

Under the URL-as-document model:

- 3 templates (invoice, job report, field note) — each ~3KB compressed, stored once, cached by every recipient who views a document
- 3,200 URL strings — generated on demand, stored only in the app's local record list, which itself is a few hundred KB of localStorage
- Sharing produces a URL. The URL is the document. There is no file.

The storage cost of those 3,200 documents is effectively the storage cost of 3,200 short strings. Template updates propagate to all future renders without re-generating any files. Old URLs remain valid through version pinning in the payload.

This relationship holds at any scale. A business generating 100,000 documents per year has the same template storage overhead as one generating 100. Only the number of URL strings grows, and those cost nothing to store.

---

## The Template as Business Infrastructure

In this model, a business's templates are infrastructure in the same sense as their website or their letterhead. They define how the business appears to customers and partners. They carry brand, structure, and — when JS-enabled — interaction.

Unlike a website, templates are:

- **Portable**: cached on every recipient's device that has ever viewed a document from this business. If the business's own site goes offline, previously cached templates continue rendering correctly for all prior recipients.
- **Versioned**: the `version` field ensures the registry accepts only upgrades. A template update takes effect for all future renders. Recipients who already have version 1 cached will receive version 2 the next time they view a new document.
- **Composable**: the `lineage.derivedFrom` field allows a business to fork a community template as a starting point. The fork carries its own URI and can be independently updated.
- **Cross-platform by design**: a single template can serve KaiOS feature phones, Android browsers, iPhones, and desktop browsers, with platform-targeted CSS variants declared in the manifest.

The cost of maintaining these templates is the cost of maintaining a small JSON file at a stable URL. There is no rendering infrastructure to run, no document storage to scale, and no proprietary format to migrate away from.

---

## Business Presence Without File Hosting

The traditional model of business web presence looks like this:

```
Business → produces files → uploads to server → shares links → recipients download
```

The URL-as-document model looks like this:

```
Business → defines templates (once) → generates URL strings (per document)
Recipients → visit URL → orchestrator resolves template → document rendered locally
```

The business's web presence in this model is:

1. **A domain that hosts template JSON files** — static, small, cacheable, no database, can be a GitHub Pages site or an S3 bucket
2. **A Workpads app or API** that generates encoded URL strings for each document
3. **A template URI** that acts as the business's document identity

No CMS. No document management system. No storage that grows with volume. No rendering servers.

A small business whose IT infrastructure is a mobile phone can have the same document distribution capability as one running a dedicated server, because the rendering happens at the recipient's end using their own device and browser.

---

## The Registry Service Model

Because templates need to be fetchable at their canonical URI, and because businesses benefit from permanence, trust verification, and discovery, a registry service has a natural role.

The service does not host documents — that distinction is important. It hosts templates: small, static JSON files with defined schemas. What it sells is not storage of volume — it sells:

**URI permanence**: A template at `https://registry.workpads.me/tpl/{id}` is guaranteed to resolve as long as the registry operates. Self-hosted URIs depend on the author's own infrastructure.

**Trust elevation**: Community templates (self-published) cannot execute JavaScript. Verified templates (`official` trust) can. Verification is a one-time review of a specific template version, not an ongoing audit. The trust level is encoded in the manifest and checked by the app before any JS execution is permitted.

**Analytics**: Because template fetches are explicit HTTPS requests to a known endpoint, the registry can record render counts, platform distribution, and version adoption rates. This is metadata about how a template is being used — not about who is using it or what their documents contain (the content never touches the registry server).

**Discovery**: A curated gallery index makes templates findable by type, industry domain, and platform. This is an editorial function, not a gating function — templates can exist and spread outside the gallery.

**Versioning guarantees**: The registry enforces the monotonic version constraint and maintains supersession chains, ensuring fallback resolution works correctly across template versions.

### Tier structure

| Tier | URI location | Trust level | JS enabled | Analytics | Gallery |
|---|---|---|---|---|---|
| Free (self-hosted) | Author's own domain | community | No | No | Optional submission |
| Paid (registry-hosted) | registry.workpads.me | official | Yes | Yes | Included |
| Business | Custom subdomain | official | Yes | Full | Priority |

The paid tier's value is not storage — it is reliability, trust, and reach. A business paying for registry hosting is buying the guarantee that their template URI resolves correctly for every recipient, on every device, at any time.

---

## Customer-to-Business and Business-to-Customer Sharing

The URL-as-document model changes the direction of information flow in ways that are worth making explicit.

**Business to customer** (traditional): Business generates a PDF invoice, emails it or uploads it to a portal, customer downloads it, files it somewhere.

**Business to customer** (URL model): Business generates a URL string. Customer opens URL, sees invoice rendered through business's template in their browser — or, if the customer also uses a compatible app, the app ingests the record and stores it locally in a format it can process.

**Customer to business**: A customer with a Workpads app can generate a purchase order or a service request as a URL string using their own or a shared template. They send the URL. The business's app opens it, decodes the record, stores it, acts on it. No file transfer. No format negotiation. No re-keying data.

**Business to business**: Two businesses that have both adopted the Standard can exchange structured records — job handoffs, purchase orders, delivery confirmations — as URL strings. The data is structured (validated against a schema), the display is templated (each party can choose their own presentation), and the exchange requires no shared infrastructure beyond the ability to send and receive URLs.

This is not a new concept in principle — EDI (Electronic Data Interchange) has done structured business document exchange for decades. The difference is the barrier to entry: EDI requires dedicated software, trading partner agreements, and often a VAN (Value Added Network) intermediary. The URL model requires a browser and a URL string.

---

## What This Is Not

To be precise about the scope:

**It is not a replacement for databases.** URL strings carry a snapshot of data at the time of encoding. They are not live-updating. If a job record changes after a share URL is generated, the URL reflects the state at generation time. This is a feature for document sharing (invoices should not change after sending) and a limitation for live dashboards.

**It is not a replacement for authenticated document portals.** URL fragments are not secret — anyone with the URL can view the document. For documents requiring access control, a different mechanism is needed. The model is designed for sharing, not for secure vaulting.

**It is not dependent on Workpads.** The Standard is public. The orchestrator page pattern can be implemented by anyone. The template registry service is one implementation of the Standard, not the only possible one. A business can run its own orchestrator page and its own template hosting independently of any Workpads service.

**It does not require online access to render.** Once a template is cached, it renders offline. The orchestrator page itself can be cached as a PWA. A field worker with no signal can receive a URL via SMS, open it, and see the document rendered correctly from cached templates.

---

## Summary

The URL-as-document model is a consequence of three design choices made independently and for other reasons:

1. Put data in the URL fragment, not on a server
2. Separate content from presentation and give each a stable identity
3. Make the rendering engine available on every device through a static orchestrator page

The result is a document distribution system with no storage infrastructure, no rendering servers, no proprietary format, and no growing cost curve as document volume increases. Businesses define templates once and generate URL strings forever. Recipients render documents locally using cached templates. The system works offline, across platforms, and without central coordination.

The registry service extracts value from the parts of this that genuinely benefit from centralisation — URI permanence, trust verification, and discovery — without making the core function dependent on that centralisation.

This is not a new category of software. It is the web's existing mechanism — URLs, static files, browser rendering — applied to a use case (business document exchange) that has historically been handled by heavier infrastructure than it requires.
