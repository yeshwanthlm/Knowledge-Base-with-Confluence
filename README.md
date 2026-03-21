# 🧠 AI-Powered Knowledge System with MCP + Confluence + AWS Bedrock

> Turn your organisation's scattered documentation into a semantic knowledge engine — queryable in natural language from your code editor.

---

## 📋 Prerequisites

Before starting, make sure you have the following installed on your machine:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) — configured with a valid profile
- A code editor with MCP support: [Cursor](https://www.cursor.com/), [VS Code](https://code.visualstudio.com/), [Kiro](https://kiro.dev/), or Antigravity

---

## 🗺️ What We're Building

<img width="2557" height="1750" alt="image" src="https://github.com/user-attachments/assets/84dc0315-4843-4d35-9bca-8f5f4d6dfd58" />

```
Confluence Pages
      │
      ▼
MCP Atlassian Server          ←─── Query live Confluence pages from your editor
      │
      ▼
AWS Bedrock Knowledge Base    ←─── Semantic vector search over ALL your docs
      │
      ▼
Bedrock KB Retrieval MCP      ←─── Connect Bedrock KB back to your editor
      │
      ▼
Your Code Editor (Kiro / Cursor / VS Code)
```

---

## Step 1 — Create an Atlassian Account

1. Go to [https://www.atlassian.com/](https://www.atlassian.com/) and sign up for a free account.
2. Create a new **Confluence** site when prompted. Choose a site name — this becomes your URL:
   ```
   https://your-site-name.atlassian.net/wiki
   ```
3. Note down your:
   - **Confluence URL** → `https://your-site-name.atlassian.net/wiki`
   - **Atlassian email** → the email you signed up with

---

## Step 2 — Create Confluence Pages

You need some documentation in Confluence for the knowledge system to index.

**Option A — Use the example files (Recommended)**

Example Confluence page files are available in the `/files` folder of this repository. Open each file, copy the content, and paste it into a new Confluence page.

**Option B — Create your own pages**

Create pages covering topics like:
- Runbooks and incident response guides
- Architecture decision records
- Onboarding documentation
- RCAs and post-mortems
- Data contracts or API specs

> **Tip:** The more structured and descriptive your pages are, the better the knowledge base retrieval quality will be. Add clear headings, bullet points, and avoid walls of unstructured text.

---

## Step 3 — Create an Atlassian API Token

An API token is required to authenticate both the MCP Atlassian server and the AWS Bedrock Confluence connector.

1. Go to: [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)
2. Click **Create API token**
3. Give it a descriptive label, e.g. `mcp-bedrock-demo`
4. Click **Create** and **copy the token immediately** — you cannot view it again after closing the dialog

> ⚠️ **Security note:** Treat this token like a password. Do not commit it to Git. Store it in your `.env` file or a secrets manager.

---

## Step 4 — Add the MCP Atlassian Server to Your Editor

The [mcp-atlassian](https://github.com/sooperset/mcp-atlassian) server lets your AI code editor query Confluence (and Jira) directly using natural language.

### 4.1 — Add MCP Server Configuration

Open your editor's MCP configuration file and add the following:

```json
{
  "mcpServers": {
    "mcp-atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://your-site-name.atlassian.net",
        "JIRA_USERNAME": "your.email@company.com",
        "JIRA_API_TOKEN": "your_api_token",
        "CONFLUENCE_URL": "https://your-site-name.atlassian.net/wiki",
        "CONFLUENCE_USERNAME": "your.email@company.com",
        "CONFLUENCE_API_TOKEN": "your_api_token"
      }
    }
  }
}
```

Replace the placeholder values:

| Placeholder | Replace with |
|-------------|-------------|
| `your-site-name` | Your Atlassian site name |
| `your.email@company.com` | Your Atlassian account email |
| `your_api_token` | The API token you created in Step 3 |

**Where is the MCP config file?**

| Editor | Config file location |
|--------|---------------------|
| Cursor | `.cursor/mcp.json` in your project root, or `~/.cursor/mcp.json` globally |
| VS Code | `.vscode/mcp.json` in your project root |
| Kiro | `.kiro/settings/mcp.json` in your project root |

> 📖 **Reference:** [https://github.com/sooperset/mcp-atlassian](https://github.com/sooperset/mcp-atlassian)

---

## Step 5 — Test the MCP Atlassian Connection

Restart your editor after saving the MCP config. Then open the AI chat panel and try a prompt like:

```
List all Confluence spaces available in my account.
```

```
Show me the content of the page titled "Kubernetes Incident Runbook" in Confluence.
```

```
Search Confluence for pages about data pipeline architecture.
```

If the MCP server is connected correctly, your editor will call the Atlassian MCP tool and return real data from your Confluence instance.

> ✅ **Success looks like:** Your editor returns actual page titles, content, or space names from your Confluence account — not a generic "I don't have access to that" response.

---

## 💡 Why Do We Need AWS Bedrock Knowledge Base — and Why Not Just Use MCP?

This is a great question, and the answer matters for anyone building at enterprise scale.

### What MCP Atlassian Does Well

The MCP Atlassian server is excellent for **live, on-demand queries** against Confluence. When you ask "show me the runbook for payments", it fetches that page in real time. It works well when:

- You know roughly what you're looking for
- Your Confluence instance has a manageable number of pages (hundreds, not tens of thousands)
- You need the absolute latest version of a document

### Where MCP Alone Falls Short

| Limitation | Why it matters |
|------------|----------------|
| **Keyword-dependent** | MCP searches Confluence using Confluence's own CQL (search syntax). If your query words don't appear in the document, it won't find it. |
| **No semantic understanding** | Asking "what do we do when payments goes down?" won't reliably find a page titled "Incident Response: Payment Infrastructure" unless those exact words match. |
| **Doesn't scale to large doc sets** | With 10,000+ pages across 30+ spaces (common after acquisitions or in large orgs), fetching and ranking results in real time becomes slow and imprecise. |
| **No cross-document synthesis** | MCP returns individual pages. It can't synthesise an answer that spans five different runbooks, three RCAs, and two architecture docs simultaneously. |
| **Acquired company docs stay siloed** | Docs from acquired companies in separate Confluence instances can't be queried together. |

### What AWS Bedrock Knowledge Base Adds

AWS Bedrock Knowledge Base solves these problems by **pre-processing your entire Confluence corpus** into a vector database:

1. **Ingestion:** Bedrock connects to Confluence via a native connector, pulls all pages, and chunks the content into segments.
2. **Embedding:** Each chunk is converted into a high-dimensional vector (a numerical representation of its *meaning*) using an embedding model like Amazon Titan Embeddings.
3. **Storage:** Vectors are stored in Amazon OpenSearch Serverless — a managed vector store.
4. **Retrieval:** When you ask a question, your query is also converted to a vector and compared against all stored vectors using cosine similarity. The most semantically similar chunks are returned — regardless of exact wording.
5. **Synthesis:** A language model (Claude, Titan, etc.) reads the retrieved chunks and synthesises a coherent, cited answer.

### The Combined Architecture is the Right Answer

```
┌─────────────────────────────────────────────────────────┐
│  Use MCP Atlassian when...          Use Bedrock KB when..│
│                                                          │
│  • You need a specific page         • You need semantic  │
│    by name or title                   search over all    │
│                                       documentation      │
│  • You want real-time content       • Your org has       │
│    (latest edits included)            10,000+ pages      │
│                                                          │
│  • Simple, direct lookups           • Cross-document     │
│                                       synthesis needed   │
│                                                          │
│                                     • Natural language   │
│                                       queries like       │
│                                       "why did payments  │
│                                       fail in Nov?"      │
└─────────────────────────────────────────────────────────┘
```

> In short: **MCP gives you a live window into Confluence. Bedrock gives you a semantic brain over everything your organisation has ever documented.**

---

## Step 6 — Create the Knowledge Base in AWS Bedrock

### 6.1 — Prerequisites

Make sure your AWS account has the following:
- Access to **Amazon Bedrock** in your chosen region (recommended: `us-east-1`)
- **Amazon OpenSearch Serverless** available in the same region
- IAM permissions for Bedrock Knowledge Bases (see [AWS docs](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-permissions.html))

### 6.2 — Open Amazon Bedrock

1. Log in to the [AWS Console](https://console.aws.amazon.com/)
2. Search for **Amazon Bedrock** in the services search bar
3. In the left sidebar, under **Builder tools**, click **Knowledge Bases**
4. Click **Create knowledge base**

### 6.3 — Configure the Knowledge Base

**Knowledge base details:**
- **Name:** `confluence-enterprise-kb` (or any descriptive name)
- **Description:** `Knowledge base ingesting Confluence documentation for semantic search`
- **IAM role:** Select **Create and use a new service role** (recommended)

Click **Next**.

### 6.4 — Configure the Data Source (Confluence Connector)

1. Under **Data source**, select **Confluence**
2. Fill in the connection details:

| Field | Value |
|-------|-------|
| Data source name | `confluence-main` |
| Confluence URL | `https://your-site-name.atlassian.net/wiki` |
| Auth type | Basic authentication |
| Username | Your Atlassian email |
| API token | The token from Step 3 (store in AWS Secrets Manager) |

3. **Scope configuration** — choose which spaces to index:
   - To index all spaces: leave the filter empty
   - To index specific spaces: add space keys (e.g. `ENG`, `DATA`, `OPS`)

> **Recommendation for first deployment:** Start with 2–3 spaces. Verify retrieval quality, then expand. Full re-indexing is automatic on the sync schedule.

4. **Sync schedule:**
   - For demo purposes: **On-demand** (manual sync)
   - For production: **Every 24 hours** (or more frequently for fast-moving docs)

Click **Next**.

### 6.5 — Configure the Embedding Model

1. Under **Embeddings model**, select:
   - **Amazon Titan Embeddings V2** — recommended for most use cases (cost-efficient, stays in AWS)
   - Or **Cohere Embed English** for potentially higher quality on English-only content

2. **Vector dimensions:** Leave as default (1024 for Titan V2)

Click **Next**.

### 6.6 — Configure the Vector Store

1. Select **Quick create a new vector store in Amazon OpenSearch Serverless** — this is the simplest option
2. Bedrock will automatically provision an OpenSearch Serverless collection
3. Leave all other settings as default

Click **Next**, review the configuration, then click **Create knowledge base**.

> ⏳ **Wait time:** Creating the knowledge base and provisioning OpenSearch Serverless takes approximately 5–10 minutes.

### 6.7 — Run the First Sync

1. Once the knowledge base is created, go to the **Data sources** tab
2. Select your `confluence-main` data source
3. Click **Sync now**
4. Wait for the sync to complete — the **Sync status** will change from `Syncing` to `Ready`

> ⏳ Sync time depends on page count. ~100 pages takes about 2–5 minutes. Larger spaces take longer.

### 6.8 — Test in the Bedrock Console

1. In your knowledge base, click **Test knowledge base**
2. In the chat panel, type a question based on your Confluence content:

```
What is the process for responding to a payments service outage?
```

```
What are the data quality SLAs for the revenue pipeline?
```

```
Who won the Q4 2024 Engineer of the Quarter award?
```

> ✅ **Success looks like:** Answers with citations showing the source Confluence page name and URL. If you see "I don't have information about that", check that the sync completed successfully and the relevant pages are in the indexed spaces.

---

## Step 7 — Add the Bedrock KB Retrieval MCP Server to Your Editor

Now connect your code editor directly to the Bedrock Knowledge Base using the [AWS Labs Bedrock KB Retrieval MCP Server](https://github.com/awslabs/mcp/tree/main/src/bedrock-kb-retrieval-mcp-server).

### 7.1 — Get Your AWS Profile Name

Run the following command in your terminal to list all configured AWS profiles:

```bash
aws configure list-profiles
```

To see details of the currently active profile:

```bash
aws configure list
```

To verify the profile has Bedrock access:

```bash
aws bedrock list-knowledge-bases --region us-east-1 --profile your-profile-name
```

This should return a list of your knowledge bases. If you see an `AccessDenied` error, check that your IAM user/role has the `bedrock:ListKnowledgeBases` and `bedrock:Retrieve` permissions.

> **Don't have a named profile?** If you use environment variables or instance roles, set `AWS_PROFILE` to `default`.

### 7.2 — Add MCP Server Configuration

Open your editor's MCP configuration file and add the Bedrock KB server alongside the existing Atlassian config:

```json
{
  "mcpServers": {
    "mcp-atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://your-site-name.atlassian.net",
        "JIRA_USERNAME": "your.email@company.com",
        "JIRA_API_TOKEN": "your_api_token",
        "CONFLUENCE_URL": "https://your-site-name.atlassian.net/wiki",
        "CONFLUENCE_USERNAME": "your.email@company.com",
        "CONFLUENCE_API_TOKEN": "your_api_token"
      }
    },
    "awslabs.bedrock-kb-retrieval-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.bedrock-kb-retrieval-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "your-profile-name",
        "AWS_REGION": "us-east-1",
        "FASTMCP_LOG_LEVEL": "ERROR",
        "KB_INCLUSION_TAG_KEY": "",
        "BEDROCK_KB_RERANKING_ENABLED": "false"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Replace the placeholder values:

| Placeholder | Replace with |
|-------------|-------------|
| `your-profile-name` | Your AWS CLI profile name from Step 7.1 |
| `us-east-1` | The AWS region where you created the knowledge base |
| `KB_INCLUSION_TAG_KEY` | Optional — leave empty to include all knowledge bases, or set a tag key to filter |

### 7.3 — Optional: Enable Reranking

For better retrieval quality on complex queries, you can enable reranking:

```json
"BEDROCK_KB_RERANKING_ENABLED": "true"
```

> **Note:** Reranking adds a small latency cost. Recommended for production use cases where answer quality matters more than speed.

Restart your editor after saving the configuration.

---

## Step 8 — Retrieve from the Knowledge Base via MCP

With both MCP servers configured and running, your editor now has two modes of accessing your Confluence knowledge:

| Mode | When to use | How it works |
|------|-------------|-------------|
| **MCP Atlassian** | Direct page lookup, real-time content | Queries Confluence CQL directly |
| **Bedrock KB MCP** | Semantic search, complex questions, cross-doc synthesis | Vector similarity search over all indexed content |

### 8.1 — Discover Your Knowledge Bases

In your editor's AI chat panel, try:

```
List all available Bedrock knowledge bases and their data sources.
```

The Bedrock KB MCP server will return the knowledge bases it can access, including their IDs, names, and configured data sources.

### 8.2 — Query the Knowledge Base Semantically

Try these example queries to test semantic retrieval:

**Incident response:**
```
Using the knowledge base, what are the steps to respond to a Kubernetes node going into NotReady state?
```

**Cross-document synthesis:**
```
Search the knowledge base: what data quality issues have we experienced and what monitoring was added to prevent recurrence?
```

**People and recognition:**
```
Search the knowledge base: who were the Q4 2024 quarterly champions and what did they achieve?
```

**Architecture questions:**
```
Search the knowledge base: what is our data platform architecture and what are the SLAs for the ML feature store?
```

**Cost and FinOps:**
```
Search the knowledge base: what AWS cost optimisation initiatives are planned for 2025 and what are the expected savings?
```

### 8.3 — Compare MCP vs Bedrock KB Retrieval

Try asking the same question using both approaches to see the difference:

**Via MCP Atlassian (direct lookup):**
```
Search Confluence for pages about the revenue pipeline outage.
```

**Via Bedrock KB (semantic search):**
```
Search the knowledge base: why did the revenue pipeline fail and what was the root cause?
```

> The Bedrock KB version should return a synthesised answer with context from the RCA document, the data platform architecture doc, and potentially the FinOps doc — even though none of them contain the exact phrase "why did the revenue pipeline fail".

---

## 🔧 Troubleshooting

### MCP server not connecting

```bash
# Test that uvx can find the package
uvx mcp-atlassian --help
uvx awslabs.bedrock-kb-retrieval-mcp-server@latest --help
```

If either fails, make sure `uv` is installed and up to date:

```bash
pip install --upgrade uv
# or
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Confluence auth failing

Verify your credentials work with a direct API call:

```bash
curl -u "your.email@company.com:your_api_token" \
  "https://your-site-name.atlassian.net/wiki/rest/api/space" \
  | python3 -m json.tool
```

A successful response returns a list of spaces. A 401 means the token or username is wrong.

### AWS Bedrock access denied

```bash
# Check your identity
aws sts get-caller-identity --profile your-profile-name

# Check Bedrock permissions
aws bedrock list-knowledge-bases --region us-east-1 --profile your-profile-name

# Check if Bedrock is available in your region
aws bedrock list-foundation-models --region us-east-1 --profile your-profile-name | head -20
```

### Knowledge base returns no results

1. Confirm the sync completed: go to AWS Console → Bedrock → Knowledge Bases → your KB → Data sources → check **Last sync status** is `Ready`
2. Confirm the pages you're querying are in the synced spaces
3. Try the query directly in the Bedrock console Test panel first to rule out MCP issues

---

## 📁 Repository Structure

```
.
├── README.md                        ← You are here
├── files/
│   ├── 01_Kubernetes_Incident_Runbook.docx
│   ├── 02_Data_Platform_Architecture.docx
│   ├── 03_ML_Model_Governance.docx
│   ├── 04_RCA_Revenue_Pipeline_Outage.docx
│   ├── 05_FinOps_Cost_Optimisation.docx
│   ├── 06_Quarterly_Champions_Q4_2024.docx
│   └── 07_GenAI_Implementation_Guide.docx
└── .kiro/
    └── settings/
        └── mcp.json                 ← MCP configuration (Kiro)
```

---

## 📚 References

| Resource | Link |
|----------|------|
| MCP Atlassian GitHub | [https://github.com/sooperset/mcp-atlassian](https://github.com/sooperset/mcp-atlassian) |
| Bedrock KB Retrieval MCP Server | [https://github.com/awslabs/mcp/tree/main/src/bedrock-kb-retrieval-mcp-server](https://github.com/awslabs/mcp/tree/main/src/bedrock-kb-retrieval-mcp-server) |
| Atlassian API Token Management | [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens) |
| AWS Bedrock Knowledge Bases | [https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) |
| Amazon OpenSearch Serverless | [https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless.html](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless.html) |
| uv Installation | [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/) |

---

## 🤝 Contributing

Found an issue or want to improve the guide? Open a PR or raise an issue in this repository.

---

*Built for the video tutorial: **Build an AI Knowledge Base with MCP Server, Confluence & AWS Bedrock***
