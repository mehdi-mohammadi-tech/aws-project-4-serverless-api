# aws-project-4-serverless-api
Fully serverless REST API with Lambda, API Gateway, and DynamoDB
# AWS Project 4: Serverless API (Lambda + API Gateway + DynamoDB)

## Overview

This project implements a **fully serverless REST API** on AWS. An HTTP request travels from the internet through API Gateway, triggers a Lambda function, which reads data from a DynamoDB table and returns it as JSON — all without managing a single server.

**Key Concept:** Serverless — no servers to provision, patch, or scale. Pay only per request. Automatic scaling from zero to thousands of requests.

---

## Architecture

```
   Internet (Browser / Client)
            │
            │  1. HTTPS Request
            ▼
   ┌──────────────────┐
   │  API Gateway     │   The "front door" — public URL
   │  (HTTP API)      │   Routing, security boundary
   └────────┬─────────┘
            │  2. Invoke (Lambda Proxy Integration)
            ▼
   ┌──────────────────┐
   │  AWS Lambda      │   Business logic (Node.js)
   │  projekt4-hallo  │   Stateless, runs on demand
   └────────┬─────────┘
            │  3. Read (AWS SDK v3) — via IAM Role
            ▼
   ┌──────────────────┐
   │  DynamoDB        │   Data store (from Project 3)
   │  projekt3-spieler│   NoSQL, on-demand
   └──────────────────┘
            │
            │  4. Data flows back up the chain
            ▼
       JSON Response to client
```

---

## What Was Built

### 1. Lambda Function (`projekt4-hallo`)
- Runtime: Node.js 24.x, x86_64
- Reads all items from DynamoDB via AWS SDK v3 (`@aws-sdk/lib-dynamodb`)
- Returns structured JSON with proper HTTP status codes
- Includes try/catch error handling

### 2. API Gateway (HTTP API)
- Public HTTPS endpoint
- Lambda Proxy Integration
- Security: Open (for learning — production needs auth)

### 3. IAM Execution Role
- `AmazonDynamoDBReadOnlyAccess` policy attached
- Demonstrates the principle of Least Privilege (read-only, no write/delete)

### 4. DynamoDB Integration
- Reuses the `projekt3-spieler` table from Project 3
- Scan operation to retrieve all player records

---

## Lambda Function Code

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, ScanCommand } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async (event) => {
  try {
    const command = new ScanCommand({
      TableName: "projekt3-spieler"
    });

    const data = await docClient.send(command);

    return {
      statusCode: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        message: "Spieler aus DynamoDB!",
        anzahl: data.Items.length,
        spieler: data.Items
      })
    };
  } catch (error) {
    return {
      statusCode: 500,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

---

## Request Flow Explained

1. **Client → API Gateway:** A browser sends an HTTPS GET request to the API endpoint URL.
2. **API Gateway → Lambda:** API Gateway wraps the request into an `event` object and invokes Lambda (Lambda Proxy Integration).
3. **Lambda → DynamoDB:** Lambda assumes its IAM execution role and uses the AWS SDK to Scan the table.
4. **Response chain:** Lambda returns a response object (statusCode, headers, body); API Gateway converts it to an HTTP response; the browser displays the JSON.

The first invocation after idle may incur a **Cold Start** (initialization latency).

---

## AWS Services Used

| Service | Purpose | Cost |
|---------|---------|------|
| AWS Lambda | Serverless compute (business logic) | Free Tier: 1M requests + 400k GB-sec/month |
| API Gateway (HTTP API) | Public HTTPS endpoint, routing | Free Tier: 1M requests/month (first 12 months) |
| DynamoDB | NoSQL data store | On-demand, ~$0 when idle |
| IAM | Permissions (execution role) | Free |

---

## Key Learnings

### 1. Serverless = No Server Management
There are still servers — but AWS provisions, scales, patches, and monitors them. The developer writes only functions. This is the serverless value proposition: focus on code, not infrastructure.

### 2. The Power of Integration
This project connected three services into one workflow: API Gateway (entry), Lambda (logic), DynamoDB (data). Each service does one thing well — a microservices/composable mindset.

### 3. IAM Roles Enable Secure Service-to-Service Access
Lambda accesses DynamoDB through an IAM execution role — no hardcoded credentials. The role grants only read access (Least Privilege), so even a compromised function cannot delete data.

### 4. Error Handling Matters
External calls (DynamoDB) can fail. try/catch returns meaningful errors and correct status codes instead of opaque 502s — essential for production robustness.

### 5. HTTP API vs REST API
HTTP API was chosen: simpler, faster, ~70% cheaper than REST API — ideal for straightforward Lambda proxy integrations.

---

## Common Pitfalls (and Solutions)

| Pitfall | Cause | Solution |
|---------|-------|----------|
| `AccessDeniedException` | Lambda role lacks DynamoDB permission | Attach DynamoDB policy to execution role |
| `ResourceNotFoundException` | Wrong table name in code | Verify table name matches exactly |
| Function URL vs API Gateway confusion | Two ways to expose Lambda | Function URL = simple; API Gateway = full features |
| Code saved but not active | Forgot to Deploy | Always Deploy after editing |
| 502 Bad Gateway | Unhandled exception in Lambda | Add try/catch error handling |
| Cold Start latency | New execution environment | Provisioned Concurrency (if needed) |

---

## Security Considerations

**Current state (learning):** API is Open — anyone with the URL can call it.

**Production hardening would add:**
- Authentication (API Keys, IAM, Cognito, or Lambda Authorizer)
- Rate limiting / throttling to prevent abuse
- A custom IAM policy scoped to the single table (true Least Privilege)
- Input validation
- CloudWatch alarms and logging

---

## Interview Questions Covered

This project prepared answers for interview questions including:
- What is serverless and its pros/cons
- What does "serverless" really mean (are there no servers?)
- Typical serverless architecture on AWS
- What is a Cold Start
- Supported Lambda runtimes
- Lambda Function URL vs API Gateway
- What is the handler function
- What is the event object
- HTTP API vs REST API
- "Open" security risks
- How Lambda accesses other services (IAM roles)
- Least Privilege principle
- AWS Managed vs Customer Managed policies
- Why error handling matters in Lambda
- Describing a complete serverless application

---

## Next Steps (Project 5: Containers, IaC, CI/CD)

This serverless foundation leads to:
1. **Docker** — containerizing applications
2. **ECS / EKS** — container orchestration
3. **Terraform** — Infrastructure as Code
4. **GitHub Actions** — CI/CD pipelines

Possible serverless extensions: POST endpoints (write data), path parameters (get single player), DynamoDB Streams, Step Functions for orchestration.

---

## Repository Structure
```
aws-project-4-serverless-api/
├── README.md
├── architecture-diagram.svg
├── src/
│   └── index.mjs
└── docs/
    ├── lambda-setup.md
    ├── api-gateway-setup.md
    └── iam-permissions.md
```

---

## Author

**Mehdi Mohammadi**
- AWS SAA-C03 Certified
- Portfolio: [GitHub](https://github.com/mehdi-mohammadi-tech)
- LinkedIn: [mehdi-mohammadi-347704393](https://linkedin.com/in/mehdi-mohammadi-347704393/)

---

**Status:** ✅ Complete | **Stack:** Lambda + API Gateway + DynamoDB | **Region:** eu-central-1 (Frankfurt)

**Last Updated:** June 2, 2026
