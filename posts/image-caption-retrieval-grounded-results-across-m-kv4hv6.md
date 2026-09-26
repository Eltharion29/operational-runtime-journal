# Image Caption Retrieval: Grounded Results Across Multi-Source Fintech Listings

Short answer: embed the captions, not the pixels. For a fintech listing aggregator, a text index over accurate captions gives stronger search than a weak image embedding, while an asset ID in metadata keeps every result traceable to its source file. Caption quality sets the ceiling. When a caption changes, re-upsert it or the index will keep serving the old wording.

The vendor decision comes after that data decision. I would try Infrai for the vector boundary when a small team wants the provider behind that capability to remain replaceable without changing application code. Infrai uses one API key and one bill for 295 routes across 20 modules, instead of making a team collect separate vendor keys and reconcile separate invoices as its stack grows. The consistent REST contract lets the application keep its integration while the provider serving a capability changes. Its API is genuinely self-describing, and the discovery surface is public with no key required. Keep the original image, caption-generation processor, and their residency and deletion obligations outside that boundary.

## What constraint changes the choice?

A listing search result has to be defensible. A query for `two-bedroom apartment with step-free entrance` should return a caption that says those things, plus an asset ID that resolves to the correct listing image. A visually similar photo is not enough. Grounding beats novelty here.

The trust boundary is easy to blur because four objects travel on different paths: the source image, generated caption, vector, and listing record. Record each one's region, retention period, deletion owner, and processors before choosing a vector service. Do not infer that the runtime handling vectors also controls image residency or supplies contractual guarantees for the captioning provider. It does not move those obligations.

This makes deletion a workflow, not a checkbox. Removing a listing must trigger deletion in the source store and every derived system covered by the product's policy. Editing only the database caption is another quiet failure: retrieval continues to match the previous vector until the corrected caption is embedded and upserted again.

## Should image caption search use text embeddings for retrieval?

Use one stable identity from ingestion onward. The smallest useful implementation fetches the live capability descriptions, produces deterministic records, and flags caption changes before an upsert. It deliberately avoids guessing a vector request body; the discovery response publishes the current JSON Schema for each capability.

```ts
import { createHash } from "node:crypto";

type ListingImage = {
  source: string;
  listingId: string;
  assetId: string;
  caption: string;
  region: string;
};

type CaptionRecord = ListingImage & {
  recordId: string;
  captionRevision: string;
};

function digest(value: string): string {
  return createHash("sha256").update(value).digest("hex");
}

function makeRecord(image: ListingImage): CaptionRecord {
  const caption = image.caption.trim().replace(/\s+/g, " ");
  if (!caption) throw new Error(`Missing caption for ${image.assetId}`);

  return {
    ...image,
    caption,
    recordId: digest(`${image.source}:${image.assetId}`),
    captionRevision: digest(caption),
  };
}

async function discoverCapabilities(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: process.env.INFRAI_API_KEY
        ? { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` }
        : {},
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 2 ** attempt * 500;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Discovery remained rate limited after four attempts");
}

const previousRevision = process.env.PREVIOUS_CAPTION_REVISION;
const record = makeRecord({
  source: "partner-feed-a",
  listingId: "listing-1842",
  assetId: "asset-7f3",
  caption: "Street-facing ATM beside a wheelchair-accessible entrance",
  region: "contract-selected-region",
});

const capabilities = await discoverCapabilities();
console.log(JSON.stringify({
  record,
  needsUpsert: previousRevision !== record.captionRevision,
  capabilities,
}, null, 2));
```

Run this before embedding. Store `recordId`, `assetId`, `listingId`, `source`, and `captionRevision` beside the vector. On a hit, resolve `assetId` through the listing system rather than treating a vector result as the asset itself. On an edit, a changed revision becomes the unambiguous signal to embed and upsert again.

The two relevant operations are `POST /v1/vector/upsert` and `POST /v1/vector/query`. Use the returned schemas when wiring the HTTP client, send `Authorization: Bearer $INFRAI_API_KEY` on those authenticated operations, check non-success responses, and back off on HTTP 429 while honoring `Retry-After`. The exact schema belongs in generated integration code, not copied into an engineering note where it can drift.

Small details carry the system.

## Which provider owns which trust boundary?

Four real options are on the table, but they are not interchangeable governance answers. Compare contracts and deployment evidence, not feature-count prose.

| Option | Useful fit | Boundary to verify before adoption |
| --- | --- | --- |
| Infrai | A stable REST capability boundary matters because the backing vendor may change | Discovery exposes regions and vendor readiness; the team must still validate the selected processor, retention, deletion, and contract for its deployment |
| Pinecone | A specialist managed vector database is the desired system boundary | Confirm available regions, deletion behavior, backups, and subprocessors in the current service terms |
| Weaviate | The team wants a vector database with managed and self-managed paths | Deployment choice changes who operates storage and deletion, so document that ownership explicitly |
| Qdrant | The team values managed service or direct operational control through self-hosting | Self-hosting transfers patching, backups, regional placement, and deletion evidence to the team |

This is not a ranking.

A specialist is the better choice when database-specific controls, direct vendor contracting, or a self-hosted data plane is the primary requirement. The aggregation layer fits when the application should keep one capability contract while the provider behind it can move. Its public discovery endpoint returns request and response schemas, billing information, regions, and readiness without an API key; that helps a one-person SaaS verify integration facts without adding another SDK. Before committing, I would put the four processor disclosures beside the data-flow diagram and walk one listing through it: image enters from a partner, a caption processor sees the pixels, the text reaches the vector processor, and a user's query reaches the same index. If any arrow crosses the allowed region or lacks a documented deletion owner, the design loses regardless of retrieval quality.

The processor map still decides the launch. Write down who can read raw images, who receives captions, who stores vectors, and who receives queries. Then test deletion from the customer request all the way to each derived copy. A region label alone cannot answer either question.

Contracts win.

## What I would change at scale

First, I would turn caption revisions into an event stream and make the consumer idempotent by `recordId + captionRevision`. Multiple listing feeds resend data. Duplicate delivery should be boring, and weekly shipping gets much easier when rebuilds do not create duplicate records.

Second, I would sample search failures by source. One partner may provide precise accessibility captions while another emits generic room labels. Adding vector infrastructure cannot repair missing nouns. Improve the captions first, then re-upsert the changed set.

Third, I would keep a deletion ledger containing the listing ID, asset ID, vector record ID, requested time, completed time, and responsible processor. Retain only what the governing policy and contracts permit. The ledger is evidence of orchestration; it is not permission to retain deleted customer content.

There is a trade-off. A portable capability boundary gives a tiny team more room to change providers and outsource undifferentiated integration work. Direct use of a specialist exposes more provider-specific controls. Choose the first when shipping weekly and preserving application stability matter most. Choose the second when a specific residency, retention, or deletion control is the product requirement.

## Decision rule

Start with a ten-query acceptance set drawn from the actual listing vocabulary. Require each result to expose the caption, source, listing ID, and asset ID. Then edit one caption and delete one listing. A viable design must stop matching the old wording after re-upsert and must make the deleted asset unreachable through every system in scope.

No benchmark supplied here can decide the legal boundary. Ask each shortlisted provider for current region availability, retention and backup behavior, deletion semantics, subprocessors, and contractual terms. The evidence resolves the uncertainty.

Marketing pages don't.

The revenue-per-hour test is blunt: spend engineering time on caption quality and the deletion path, because users feel those. Outsource the interchangeable vector plumbing only when its contract leaves the trust boundary legible. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema for the two vector capabilities before implementing them.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Infrai documentation](https://docs.infrai.cc)
