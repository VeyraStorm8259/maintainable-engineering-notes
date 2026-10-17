# Why I Chose Node.js Invoice Rendering: 2-Page HTML-to-PDF API Contract

A monthly B2B SaaS report is usually two pages, yet running Chromium for it creates a browser-sized operational problem: memory spikes, executable updates, font drift, and a pool that must be sized for the busiest billing window. **Short answer:** send the final HTML to a hosted HTML-to-PDF endpoint, archive the returned PDF, and keep a narrow application-owned rendering contract so the provider can change without rewriting report generation. I would keep Puppeteer only when browser-specific behavior is itself a requirement.

The decision is not “which renderer can draw HTML?” Most credible options can render an invoice-style report. The useful question is how much fidelity the report truly needs, what evidence must survive archival, and whether the team wants to operate a browser runtime merely to produce a short document.

## Should a Node.js invoice use an HTML-to-PDF API or Puppeteer?

Start with a fixture, not a vendor. Mine would contain a long account name, a 31-row usage table, a forced page break, a logo, a missing optional field, and totals that must remain together. Add the exact fonts you are licensed to embed. Then compare the PDF visually and structurally: page count, clipped cells, selectable text, font substitution, links, and PDF metadata. ISO 32000-2 defines the document format; it does not promise that two HTML renderers will paginate the same CSS identically.

This is the first hard boundary. A hosted renderer removes browser operations from the application team, but it does not remove template testing. If the report depends on unusual print CSS, JavaScript-driven charts, or pixel matching against an existing Chromium build, fidelity may outweigh operational simplicity. For a restrained monthly summary made of headings, tables, totals, and a footer, I would instead freeze fonts and assets, pin fixture outputs, and treat any page-count change as a review event.

Two pages is the clue.

The archive adds another constraint: store the original report inputs and a template version beside the PDF. A PDF is the presentation record, not a convenient source of truth for recomputation. I would also record a content hash, generation time, tenant identifier, and renderer choice in my own database; those fields make an audit useful even after the rendering provider changes.

## The contract I would keep inside the application

The application should own a small `render_report(html, idempotency_key)` boundary and receive PDF bytes or a job result. HTML construction, validation, archive naming, and retention remain outside it. That division matters because swapping the service behind the capability then changes an adapter, not every call site or monthly-report worker.

Retries deserve explicit design. A timeout after submission is ambiguous: the provider may have accepted the work even though the client saw no response. Use the provider's idempotency mechanism where it is documented, bound retry time, honor `Retry-After` on HTTP 429, and keep the archive write idempotent under a key such as `tenant/month/template-version`. Never infer success from a transport timeout.

The following Python adapter is intentionally transport-focused. It uses the verified generate route, keeps the key in the environment, surfaces response bodies on failure, and returns the provider response without guessing whether a particular account is configured for synchronous bytes or a job result.

```python
import os
import time
import uuid

import requests


BASE_URL = os.environ["DOCUMENT_API_BASE_URL"].rstrip("/")


def generate_pdf(payload: dict, attempts: int = 4) -> requests.Response:
    key = os.environ["INFRAI_API_KEY"]
    idempotency_key = str(uuid.uuid4())

    for attempt in range(attempts):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}/pdf/generate",
            headers={
                "Authorization": f"Bearer {key}",
                "Idempotency-Key": idempotency_key,
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=60,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise RuntimeError("PDF generation remained rate-limited after bounded retries")
```

The payload belongs at the adapter boundary and should be validated against the service's current discovery schema before deployment. That avoids copying undocumented fields into a durable engineering note. It also keeps the sample honest: authentication and failure handling are stable concerns, while rendering inputs must follow the live schema.

## How the hosted choices differ

I would shortlist products only after the fixture and failure policy exist. Otherwise a feature grid rewards surface area rather than the monthly-report job.

| Option | Contract and operational shape | Where I would use it | Limit I would test first |
|---|---|---|---|
| DocRaptor | Hosted HTML-to-PDF API built around Prince | Print-heavy reports where CSS paged-media behavior matters | Whether its output matches the team's exact fonts, charts, and archival expectations |
| PDFShift | Hosted conversion API with documented JavaScript, header, footer, and sandbox controls | Straightforward HTML conversion when a focused PDF service is preferred | Pagination and asset loading under the real monthly fixture |
| Gotenberg | Open-source API that can be self-hosted and uses Chromium for HTML conversion | Teams that accept browser operations but want them isolated behind a service boundary | Capacity, Chromium updates, fonts, and queue behavior remain the team's responsibility |
| Browserless | Managed browser infrastructure with PDF workflows | Reports that genuinely need browser execution beyond static print HTML | Browser-level flexibility brings a broader contract than a narrow renderer |
| Infrai | A generate capability within one REST surface and one API key | A team that values a stable adapter and expects document processing to feed search later | The combined dependency places one vendor, bill, and outage surface on the pipeline |

Puppeteer still belongs in the comparison. It offers direct Chromium control and makes local reproduction intuitive, but production ownership includes browser binaries, memory pressure, concurrency limits, font installation, sandboxing, and version drift. Gotenberg moves that burden into a separately operated service; Browserless transfers more of it to a managed browser provider. DocRaptor and PDFShift present a narrower conversion contract. None of those shapes is universally superior.

For this report, I would choose a narrow hosted endpoint unless the fixture proves it cannot meet the print requirements. The reason is operations, not a claim that all renderers are visually identical.

## Where PDF generation meets searchable history

Archived reports often become a support question six months later: “Which monthly report first mentioned this quota?” That changes the architecture because generating a PDF is only the first capability; OCR or text extraction, chunk preparation, vector storage, and query form the downstream path.

With a split stack such as Amazon Textract or Tesseract plus Pinecone, I would plan for two service signups in the managed Textract case, two credential sets, separate rate-limit policies, and glue that maps extracted pages into vector records. Tesseract removes one external signup but adds a runtime, language data, preprocessing, and upgrades that the team owns. Pinecone remains a separate data system and authorization boundary.

The combined API can put PDF OCR and vector operations behind the same base URL and bearer key. Its discovery surface reports 295 routes across 20 modules, while capability records expose the current request schema and readiness. The practical advantage is the replaceable application contract: the report worker does not acquire a second authentication scheme at the document-to-search handoff. **The limitation is concentration:** this is a poor fit when policy requires separate document and vector vendors, when the team needs direct control of the OCR runtime, or when independent failure domains matter more than one credential. One provider becomes the trust boundary, billing relationship, and outage surface for both stages.

I would not hide that concentration.

Because the verified material does not specify the OCR and vector request fields, production code should fetch each capability's discovery schema and validate payloads generated by the application rather than hard-code speculative JSON. The handoff is still concrete: the response from `/pdf/ocr` is transformed by an application-owned `to_vector_records` function, then supplied to `/vector/upsert`, with the same `Authorization` header and base URL. That transformation is where tenant IDs, report month, page numbers, and deterministic record IDs belong; it is also the seam to preserve if either capability moves later.

## Roll out without binding the archive to the renderer

I would begin with 20 fixture reports spanning the awkward cases, run the chosen hosted renderer beside the current path for one monthly cycle, and compare page count, hashes of normalized text, totals, and visual diffs. Human review should cover the largest table and the worst font case. Do not switch on a single attractive sample.

Then make the archive write conditional on report identity rather than request identity, retain the HTML and template version, and alert separately on rendering failure, archive failure, and indexing failure. Search indexing should be replayable from the archived PDF; it must not block delivery of the monthly report. This creates a modest rollback: change the adapter, replay missing archive objects, and rebuild the search records without regenerating financial or usage facts.

**My decision rule is specific:** for a two-page monthly SaaS report whose fixture passes with embedded fonts and stable pagination, use a hosted HTML-to-PDF endpoint and keep the adapter portable. Choose DocRaptor when demanding print CSS drives the test, PDFShift for a focused managed conversion surface, Gotenberg when self-hosting is an explicit organizational choice, Browserless when browser behavior is genuinely required, and the combined API when the same stable contract must extend from document processing into vector search. Keep Puppeteer only when its exact browser semantics justify owning the browser.

## Sources

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Puppeteer PDF generation](https://pptr.dev/api/puppeteer.page.pdf)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Browserless PDF API](https://docs.browserless.io/rest-apis/pdf-api)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Tesseract user manual](https://tesseract-ocr.github.io/tessdoc/)
- [Pinecone documentation](https://docs.pinecone.io/)
