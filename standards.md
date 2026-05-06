# Standards & API Reference

> Project: Scientific Peer Review Manager · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 27729 / ISNI — International Standard Name Identifier**
- URL: https://www.iso.org/standard/44292.html
- ORCID iDs are a subset of ISNI and comply with ISO 27729, providing a globally unique, persistent identifier for researchers and contributors. Any peer review system that records reviewer and author identities should store and expose ORCID iDs in ISO 27729-compliant form.

**ISO 15836 — Dublin Core Metadata Element Set**
- URL: https://www.iso.org/standard/71339.html / https://www.dublincore.org/
- The fifteen Dublin Core elements (Title, Creator, Subject, Description, Publisher, Date, Identifier, etc.) are the baseline metadata vocabulary for describing journal articles and peer review records. Also standardised as ANSI/NISO Z39.85 and IETF RFC 5013.

**ISO/IEC 7064:2003 — Check Character Systems**
- URL: https://www.iso.org/standard/31531.html
- Specifies the MOD 11-2 algorithm used to compute the check digit at position 16 of ORCID iDs. Implementations that generate or validate ORCID iDs must implement this checksum correctly.

### ANSI/NISO Standards

**ANSI/NISO Z39.96-2024 — JATS: Journal Article Tag Suite (v1.4)**
- URL: https://www.niso.org/standards-committees/jats / https://jats.nlm.nih.gov/
- The canonical XML schema for encoding journal article content, including manuscript structure, metadata, and — since recent JATS4R guidance — peer review materials such as referee reports, decision letters, and author responses. Any system that ingests or exports structured manuscript XML should target JATS 1.4 compliance. The JATS4R peer review guidance at https://jats4r.niso.org/peer-review-materials/ covers tagging of peer review objects specifically.

**ANSI/NISO Z39.106-2023 — Standard Terminology for Peer Review**
- URL: https://www.niso.org/standards-committees/peer-review-terminology / https://peerreviewterminology.niso.org/
- Defines a controlled vocabulary for peer review models — covering identity transparency (open/anonymous/double-anonymous), reviewer interactions, published review process information, and post-publication commenting. Implementing this taxonomy in system UI and metadata surfaces enables interoperability and clear communication to authors and reviewers about the review model in use.

**ANSI/NISO Z39.85 — Dublin Core Metadata Terms**
- URL: https://www.niso.org/publications/ansiniso-z3985-2012-dublin-core-metadata-element-set
- U.S. national standard encoding of the Dublin Core element set, relevant to OAI-PMH metadata interchange between peer review systems and repositories.

### W3C & IETF Standards

**OAI-PMH v2.0 — Open Archives Initiative Protocol for Metadata Harvesting**
- URL: https://www.openarchives.org/OAI/openarchivesprotocol.html
- An HTTP/XML protocol that exposes repository metadata for harvest by aggregators (Google Scholar, BASE, CORE). Journal publishing systems and peer review archives that wish to be indexed should implement OAI-PMH v2.0 endpoints, with Dublin Core as the mandatory baseline metadata format.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- Governs the OAuth 2.0 flows used by ORCID, CrossRef, and institutional SSO integrations. Required for any peer review system that delegates authentication or requests delegated access to researcher records.

**RFC 8414 — OAuth 2.0 Authorization Server Metadata**
- URL: https://www.rfc-editor.org/rfc/rfc8414
- Defines the `.well-known/oauth-authorization-server` discovery document format, enabling clients to auto-configure their OAuth integrations with ORCID and institutional identity providers.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- The identity layer over OAuth 2.0, used for institutional SSO sign-in to peer review platforms and for ORCID's authenticated iD collection flow. Needed for GDPR-compliant identity federation with minimal PII transmission.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://www.rfc-editor.org/rfc/rfc7807
- Standard JSON error response format for REST APIs. Relevant to any REST API surface the system exposes (submission ingest, reviewer recommendation, notification webhooks).

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry standard for documenting RESTful APIs. OJS exposes its REST API via a Swagger/OpenAPI source document (docs/dev/swagger-source.json). Any new peer review management platform should publish an OpenAPI 3.1 specification to enable third-party integrations and tooling.

**JATS4R — JATS for Reuse: Peer Review Materials Guidance**
- URL: https://jats4r.niso.org/peer-review-materials/
- Practical recommendations for tagging peer review objects (referee reports, decision letters, author responses) in JATS XML, supporting a range of implementation levels from minimal PDF tagging to full text XML. Directly relevant to any system exporting peer review records for publishers.

**Crossref Schema — Peer Reviews Record Type**
- URL: https://www.crossref.org/documentation/schema-library/markup-guide-record-types/peer-reviews/
- Crossref's deposit schema for registering peer reviews and linking them to the reviewed article via DOI relationships. Supports referee reports, decision letters, author responses, and post-publication reviews. Peer review management systems that enable open peer review publication should implement Crossref peer review deposition.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) — Delegated Authorisation**
- Used by ORCID's Member API, CrossRef's API, and institutional identity providers. Peer review systems must implement OAuth 2.0 to collect authenticated ORCID iDs and to act on behalf of users in connected systems.

**OpenID Connect Core 1.0 — Federated Identity**
- Required for institutional SSO (university login, publisher SSO) and ORCID authenticated iD collection. Enables GDPR-friendly minimisation of PII transmitted during login.

**GDPR (EU 2016/679) — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Directly applicable to peer review systems operating in the EU or processing EU researchers' data. Reviewer identities, manuscript content, and review reports are personal data under GDPR. Systems must implement lawful basis for processing, data minimisation, right to erasure, and data portability. Reviewer anonymity features interact with GDPR right-of-access obligations.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-ten/
- Baseline security checklist for the web application layer of any peer review platform, covering injection, broken authentication, XSS, insecure direct object references (especially relevant to manuscript confidentiality), and API security.

### MCP Server Specifications

Not directly applicable to core peer review workflow infrastructure. AI-powered features (reviewer matching, manuscript pre-screening, review quality analysis) may expose internal AI agents via MCP if the platform adopts an agentic architecture. The MCP specification is maintained at https://modelcontextprotocol.io/.

---

## Similar Products — Developer Documentation & APIs

### Editorial Manager (Aries Systems / Wolters Kluwer)

- **Description:** The dominant peer review and manuscript submission management system globally, used by thousands of journals across publishers including Springer Nature, Elsevier, and major society publishers.
- **API Documentation:** https://emhelp.editorialmanager.com/robohelp/index.htm (editorial workflow help); Web Services and Notification Services API announced in version 15.1.
- **Repository API:** A generic Repository API enabling flexible vendor integrations is available to customers and third-party repositories (Dryad, Figshare, Mendeley Data). See: https://www.ariessys.com/news-and-events/press-releases/aries-systems-to-offer-new-repository-api-for-flexible-vendor-integrations/
- **Standards:** REST-based web services and notification services; XML data exchange; ORCID integration; CrossRef/DOI registration; iThenticate plagiarism detection integration.
- **Authentication:** Customer-authenticated API access; ORCID OAuth 2.0 for researcher identity.
- **Developer Guide:** Contact Aries Systems directly; some PDF API documentation available to partners.

### ScholarOne Manuscripts (Clarivate / Silverchair)

- **Description:** Full manuscript submission and editorial workflow management platform, widely adopted by Wiley, Oxford University Press, and large society publishers.
- **API Documentation:** https://developer.scholarone.com/
- **Developer Resources:** https://clarivate.com/academia-government/training-support/scholarone-manuscripts/developers/
- **SDKs/Libraries:** REST API with XML (default) and JSON response formats; sample client guide available for integrators.
- **Notification Services (Webhooks):** Real-time HTTP/HTTPS notifications to customer-defined endpoints on manuscript workflow events. See: https://developer.scholarone.com/docs/notification-services-overview
- **Relay API:** Silverchair's ScholarOne Relay API enables real-time bidirectional communication with partner platforms (e.g., Prophy AI reviewer matching). See: https://www.silverchair.com/news/silverchair-expands-integration/
- **Standards:** REST/JSON and REST/XML; ORCID integration; CrossRef integration; iThenticate integration.
- **Authentication:** API key-based; ORCID OAuth 2.0.

### Open Journal Systems (OJS) — Public Knowledge Project

- **Description:** Free and open-source journal management and publishing platform; the most widely deployed journal platform globally, especially among independent and society journals.
- **API Documentation:** https://pkp.sfu.ca/software/ojs/ / https://deepwiki.com/pkp/ojs/6-rest-api
- **OpenAPI Spec:** https://github.com/pkp/ojs/blob/main/docs/dev/swagger-source.json
- **SDKs/Libraries:** No official SDK; community PHP plugins and REST client examples on GitHub (https://github.com/pkp/ojs).
- **Developer Guide:** PKP Documentation Hub; community forums at https://forum.pkp.sfu.ca/
- **Standards:** REST/JSON API under /api/v1/; OAI-PMH v2.0 (Dublin Core and JATS); CrossRef DOI integration; ORCID plugin; iThenticate/Turnitin plagiarism plugin.
- **Authentication:** Session cookie (same-origin) or Bearer token (API key, configured per user).

### Scholastica

- **Description:** Modern, user-friendly peer review and journal publishing platform targeting society and independent journals; no public REST API.
- **API Documentation:** No public API. Integration enquiries via https://scholasticahq.com/
- **Integrations:** CrossRef Similarity Check (iThenticate); Google Scholar indexing; DOAJ; Portico archiving; ROR for institutional affiliation disambiguation.
- **Developer Guide:** https://help.scholasticahq.com/article/215-does-scholastica-integrate-with-other-services
- **Standards:** CrossRef DOI; ORCID; ROR.
- **Authentication:** N/A (no public API).

### Prophy — AI Reviewer Matching

- **Description:** AI-powered semantic reviewer matching platform using full-text publication analysis to recommend qualified reviewers; integrates with Editorial Manager, ScholarOne, and OJS.
- **API Documentation:** https://www.prophy.ai/api-docs/
- **Journal Recommendation API:** https://www.prophy.ai/api-docs/recommend-journals/
- **Integration Guide:** https://blog.prophy.ai/how-to-integrate-peer-review-tools-with-editorial-management-systems-a-publishers-technical-guide
- **Standards:** REST/JSON; webhook endpoints for editorial management system callbacks.
- **Authentication:** API key (managed via Integrations dashboard); SSO via customer platform.

### Turnitin Core API (iThenticate 2.0)

- **Description:** The Turnitin Core API (TCA) enables integration of iThenticate plagiarism and similarity detection into manuscript submission workflows, replacing the legacy iThenticate v1 API.
- **API Documentation:** https://developers.turnitin.com/turnitin-core-api/information-for-ithenticate-integrators
- **Developer Guide:** https://www.turnitin.com/blog/building-a-technical-integration-with-turnitin-partner-checklist
- **SDKs/Libraries:** REST/JSON; reference client implementations available via Turnitin developer portal.
- **Standards:** REST/JSON; HTTP Bearer token (API key in header); Similarity Report delivered as PDF or JSON.
- **Authentication:** API key sent in HTTP header per tenant.

### ORCID Member API

- **Description:** The ORCID Member API allows publishing systems to collect authenticated ORCID iDs from authors and reviewers, read and write data to ORCID records, and receive webhook notifications on record changes.
- **API Documentation:** https://info.orcid.org/documentation/integration-and-api-faq/
- **Member API Guide:** https://orcidus.lyrasis.org/technical-integration-guide/
- **Tutorial:** https://orcid.github.io/orcid-api-tutorial/
- **SDKs/Libraries:** Official ORCID-Source on GitHub: https://github.com/ORCID/ORCID-Source
- **Standards:** REST/JSON and REST/XML; OAuth 2.0 (RFC 6749); OpenID Connect; ISO 27729 (ISNI) identifiers.
- **Authentication:** OAuth 2.0 with scopes: /authenticate, /read-limited, /activities/update, /person/update.

### CrossRef REST API

- **Description:** The CrossRef REST API supports DOI registration and metadata retrieval for journal articles, peer reviews, and related scholarly objects. Publishers can register peer review records (referee reports, decision letters, author responses) and link them to articles via DOI relationships.
- **API Documentation:** https://api.crossref.org/ / https://github.com/CrossRef/rest-api-doc
- **Peer Review Markup Guide:** https://www.crossref.org/documentation/schema-library/markup-guide-record-types/peer-reviews/
- **Standards:** REST/JSON; Crossref XML deposit schema; OpenAPI-compatible endpoints.
- **Authentication:** Public API is unauthenticated (with rate limits via polite pool); Crossref member deposit API uses member credentials.

### ROR — Research Organization Registry API

- **Description:** Free, open REST API for resolving and searching research organization identifiers (ROR IDs), used to disambiguate institutional affiliations of authors and reviewers.
- **API Documentation:** https://ror.readme.io/docs/basics
- **Registry:** https://ror.org/registry/
- **Standards:** REST/JSON; CC0 open data; integrated with CrossRef, DataCite, and ORCID metadata.
- **Authentication:** No authentication required; rate limits apply.

---

## Notes

- **ScholarOne Relay API** represents the emerging pattern of real-time bidirectional integration between editorial management systems and AI-powered tools. New platforms should design their integration layer to support similar event-driven webhooks and synchronous API calls.
- **JATS 1.4 with JATS4R peer review guidance** is the most important data standard for structured manuscript and review interchange. Implementing JATS export from day one avoids costly migration work when publishers require structured XML delivery.
- **ANSI/NISO Z39.106-2023 peer review terminology** is a recently published standard (2023) that should inform UI labelling of review models and be exposed as structured metadata — enabling interoperability with publisher reporting systems and open peer review indexes.
- **GDPR and reviewer anonymity** create a design tension: GDPR rights of access may conflict with double-anonymous review confidentiality. Platforms must document their legal basis and data handling policies clearly.
- The **Prophy AI API** (https://www.prophy.ai/api-docs/) is the only currently documented, publicly accessible AI reviewer matching API; it sets the reference for what an AI-native peer review manager's own reviewer recommendation API surface should support.
