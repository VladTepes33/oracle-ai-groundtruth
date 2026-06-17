# Getting Started

## Prerequisites

- A [Claude.ai](https://claude.ai) account (Pro or Team plan recommended — Projects feature required)
- Official Oracle 19c documentation PDFs (see [recommended-pdfs.md](recommended-pdfs.md))

---

## Step 1 — Create a Claude Project

1. Go to [claude.ai](https://claude.ai)
2. In the left sidebar, click **Projects** → **New Project**
3. Give it a name, e.g. `Oracle 19c DBA Assistant`

---

## Step 2 — Add the System Prompt

1. Inside your project, click **Edit project instructions** (or the settings icon)
2. Open `prompts/system-prompt.md` from this repo
3. Copy the entire content between the triple backticks
4. Paste it into the **Instructions** field
5. Save

---

## Step 3 — Upload the Knowledge Base

1. In your project, go to **Project knowledge** → **Add content**
2. Upload `knowledge-base/oracle19c_ha_knowledge_base.md`

---

## Step 4 — Upload Oracle Official PDFs

Upload as many relevant Oracle 19c PDFs as possible.
See [recommended-pdfs.md](recommended-pdfs.md) for the full list with download links.

Priority order:
1. Oracle Data Guard Concepts and Administration 19c
2. Oracle Grid Infrastructure Installation and Upgrade Guide 19c
3. Oracle Real Application Clusters Administration and Deployment Guide 19c
4. Oracle GoldenGate 19c documentation set
5. Oracle Database High Availability Overview 19c

> **Tip**: The more targeted your PDFs, the better the citation quality.
> Avoid uploading unrelated Oracle documentation (Forms, APEX, etc.).

---

## Step 5 — Test

Start a new conversation in your project and ask:

```
What are the mandatory steps to perform a switchover from a RAC primary 
to a physical standby using Data Guard Broker?
```

You should receive a structured response ending with a `📚 References` block.

---

## Troubleshooting

**AI doesn't cite PDFs**
→ Make sure the PDFs are uploaded as project documents, not just in the conversation.

**References show [KB] only, never PDF citations**
→ The question may not be covered by the uploaded PDFs. Check `recommended-pdfs.md`.

**Responses seem generic**
→ Verify the system prompt is correctly pasted in Project Instructions (not in the chat).
