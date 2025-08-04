# Hybrid Hot-Cold Data Architecture for Billing Records

## Problem Statement  
A serverless Azure architecture stores 2M+ billing records (~300 KB each, ~600 GB raw) in Cosmos DB. Records older than 90 days are rarely accessed but remain fully provisioned in Cosmos DB, driving high storage costs. We need to:
- **Optimize storage cost** without downtime or API changes  
- **Maintain ≤ 3 s latency** for on-demand cold reads  
- **Ensure zero data loss** and **no contract breaks**  

## Constraints  
- No changes to existing API contract  
- No downtime during migration  
- Cold data (90+ days) must be retrievable within sub-2 s  
- Automated, reliable, idempotent archival  
- Secure, compliant handling of billing data  

## Design Rationale  
- **Tiered storage**: Keep hot data (0–90 days) in Cosmos DB for ultra-low RU point-reads; offload older data to cheaper cold stores.  
- **Fallback middleware**: Azure Function façade transparently reads from Cosmos DB, then Blob or Table storage if missing.  
- **Archival pipeline**: Automated with Azure Data Factory (batch) or Azure Functions + Change Feed (near-real-time) to move 90+ day records.  
- **Monitoring & resilience**: Retries, circuit breakers, dead-letter queues, metrics, Azure Monitor alerts.  
- **Security**: Encryption at rest/in transit, RBAC + Managed Identities, diagnostic logs, soft-delete, immutability as needed.

## Architecture Diagram  
![Hybrid Azure hot-cold data architecture with fallback and monitoring][C:\Users\Administrator\Desktop\Architecture diagram.png]

## Key Components  
1. **Client / API Gateway**  
   - Sends read/write requests to Azure Function façade.  

2. **Azure Function (Façade + Fallback)**  
   - HTTP-triggered.  
   - Primary: Cosmos DB point-read.  
   - Fallback: Blob or Table lookup.  

3. **Cosmos DB (Hot Store)**  
   - Stores 0–90 day records.  
   - Low-latency RU-based reads.  

4. **Cold Store**  
   - **Table Storage**: PartitionKey/RowKey for point-lookups, OData filtering.  
   - **Blob Storage (Cool)**: JSON files + metadata index in Cosmos for lookups.  

5. **Archival Pipeline**  
   - **Batch**: Azure Data Factory with watermark-based Copy Activity.  
   - **Streaming**: Azure Functions Change Feed Processor + Timer triggers.  
   - Flags archived items (`archived = true`) in Cosmos DB.  

6. **Cleanup Job**  
   - Soft-delete → batched hard-delete after verification.  

7. **Monitoring & Alerts**  
   - Application Insights metrics: `hotReads`, `coldReads`, `fallbackCount`, `archiveSuccess`, `archiveFailure`.  
   - Azure Monitor alerts on error/latency thresholds.  

## Cost Comparison (450 GB Cold Data)  
| Option                    | USD/GB-mo | Monthly Cost (USD) | Monthly Cost (INR)¹ |
|---------------------------|-----------|--------------------|---------------------|
| Cosmos DB (storage only)  | $0.25     | $112.50            | ₹9,855              |
| Blob Storage Cool tier    | $0.01     | $ 4.50             | ₹ 394               |
| Blob Storage Archive tier | $0.00099  | $ 0.45             | ₹ 39                |

¹ 1 USD = ₹87.6

## Pseudocodes

### Read Fallback Logic  

```javascript
async function handleRequest(id) {
  try {
    const doc = await cosmos.readItem(id);
    return {
      status: 200,
      body: doc
    };
  } catch (err) {
    if (err.code !== 404) throw err;

    let payload;

    if (coldStoreType === "blob") {
      const uri = (await cosmos.readMetadata(id)).blobUri;
      payload = await blobClient.downloadJSON(uri);
    } else {
      const entity = await tableClient.getEntity(pk(id), id);
      payload = JSON.parse(entity.payloadJson);
    }

    return payload
      ? { status: 200, body: payload }
      : { status: 404, body: { error: "Not found" } };
  }
}

```
### Archival Job  

```javascript
async function archiveOldRecords() {
  const cutoff = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000);
  const candidates = await cosmos.query(
    "SELECT * FROM c WHERE c.ts <= @cutoff AND c.archived != true",
    { cutoff }
  );

  for (const rec of candidates) {
    try {
      if (coldStoreType === "blob") {
        await blobClient.uploadJSON(`cold/${rec.id}.json`, rec);
        await cosmos.upsert({
          id: rec.id,
          archived: true,
          blobUri: `cold/${rec.id}.json`
        });
      } else {
        await tableClient.upsertEntity({
          PartitionKey: pk(rec.id),
          RowKey: rec.id,
          payloadJson: JSON.stringify(rec)
        });
        await cosmos.upsert({
          id: rec.id,
          archived: true
        });
      }
    } catch (err) {
      await deadLetterQueue.send(rec.id);
    }
  }
}

```
### Cleanup Job  
```javascript
async function cleanupArchived() {
  const batch = await cosmos.query(
    "SELECT c.id FROM c WHERE c.archived = true AND c.deleted != true",
    { maxItemCount: 1000 }
  );

  for (const { id } of batch) {
    try {
      await cosmos.deleteItem(id, id);
    } catch (err) {
      console.error("Delete failed", id, err);
    }
  }
}
```
---

This README summarizes a **cost-optimized**, **resilient**, and **secure** hybrid hot-cold data solution on Azure, enabling sub-2 s cold reads and transparent API behavior.  
