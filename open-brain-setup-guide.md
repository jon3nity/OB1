# Open Brain — Complete Setup Guide
*Based on actual setup completed May 1–2, 2026*
*Windows + Mac compatible*

> 🔒 **Security note:** These files are downloaded from the official Open Brain 
> GitHub repo by Nate B. Jones (the project creator). You can verify the contents 
> yourself before deploying by opening the URLs in your browser, or fork the repo 
> to your own GitHub account and download from your fork instead for maximum 
> control.

---

## What You're Building

A personal AI memory system made of two parts:

- **Supabase** — your private PostgreSQL database that stores all your memories as vector embeddings
- **Open Brain MCP Server** — a Supabase Edge Function that sits between your AI clients and your database, giving Claude and other AIs tools to read and write your memories

Once set up, any Claude session with Open Brain connected can:
- **Capture** thoughts, decisions, facts, names, deadlines
- **Search** your memory by meaning (not just keywords)
- **List** recent thoughts filtered by topic, person, or type
- **Stats** — see totals, types, top topics, people mentioned

---

## What You Need

- A Supabase account (free) — [supabase.com](https://supabase.com)
- An OpenRouter account (free, $10 credit recommended) — [openrouter.ai](https://openrouter.ai)
- Node.js installed — [nodejs.org](https://nodejs.org) (LTS version)
- Scoop package manager (Windows) — installed during setup
- The credential tracker spreadsheet — keep it open throughout

---

## Credential Tracker

Download the credential tracker: https://github.com/NateBJones-Projects/OB1/blob/main/docs/open-brain-credential-tracker.xlsx, from the Open Brain GitHub repo before doing anything else. Every API key, password, URL, and command you generate goes into this spreadsheet. Some credentials cannot be retrieved once you leave the page — if you don't save them immediately you'll have to start over.

The tracker has AUTO rows that populate themselves once you enter your Project Ref. Fill in every ENTER row as you go.

> ⚠️ Keep this file local only. Never upload it to Google Drive, OneDrive, or any cloud sync folder. Your Secret Key grants full database access.

---

## Step 1 — Create Your Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign up (GitHub login is fastest)
2. Click **New Project** in the dashboard
3. Set **Project name** — e.g. `open-brain`
4. Generate a strong **Database password** — paste into credential tracker immediately (row 1.5)
5. Set **Region** to West EU (Ireland) for lowest latency if you're in Ireland
6. Under **Security**:
   - ✅ Enable Data API — leave ticked
   - ✅ Automatically expose new tables and functions — leave ticked
   - ☐ Enable automatic RLS — **leave unticked** (access control is handled at the MCP layer)
7. Click **Create new project** and wait 60 seconds

Once the project loads, look at your browser URL:
```
supabase.com/dashboard/project/YOUR_PROJECT_REF
```

Copy that random string and paste it into the tracker at row 1.8 (Project Ref). All AUTO rows in the tracker will populate immediately.

---

## Step 2 — Set Up the Database

Three SQL commands, run one at a time in the Supabase SQL Editor (left sidebar → SQL Editor → New query).

### 2.1 — Enable the Vector Extension

In the left sidebar go to **Database → Extensions**, search for "vector", and flip **pgvector ON**.

### 2.2 — Create the Thoughts Table

New query → paste and Run:

```sql
-- Create the thoughts table
create table thoughts (
  id uuid default gen_random_uuid() primary key,
  content text not null,
  embedding vector(1536),
  metadata jsonb default '{}'::jsonb,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Index for fast vector similarity search
create index on thoughts
  using hnsw (embedding vector_cosine_ops);

-- Index for filtering by metadata fields
create index on thoughts using gin (metadata);

-- Index for date range queries
create index on thoughts (created_at desc);

-- Auto-update the updated_at timestamp
create or replace function update_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger thoughts_updated_at
  before update on thoughts
  for each row
  execute function update_updated_at();
```

> ⚠️ **Common error:** If you get `syntax error at or near "$"` on the trigger function, the `$$` dollar-quote delimiters got corrupted during copy-paste. Make sure both dollar signs are present: `$$` not `$`.

When Supabase warns about RLS — click **Run without RLS**. This is safe because your MCP server handles access control via the `MCP_ACCESS_KEY`.

### 2.3 — Create the Semantic Search Function

New query → paste and Run:

```sql
create or replace function match_thoughts(
  query_embedding vector(1536),
  match_threshold float,
  match_count int,
  filter jsonb default '{}'::jsonb
)
returns table (
  id uuid,
  content text,
  metadata jsonb,
  similarity float,
  created_at timestamptz
)
language plpgsql
as $$
begin
  return query
  select
    t.id,
    t.content,
    t.metadata,
    1 - (t.embedding <=> query_embedding) as similarity,
    t.created_at
  from thoughts t
  where 1 - (t.embedding <=> query_embedding) > match_threshold
    and (filter = '{}'::jsonb or t.metadata @> filter)
  order by t.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### 2.4 — Create the Upsert Function

New query → paste and Run:

```sql
create or replace function upsert_thought(
  p_content text,
  p_payload jsonb
)
returns jsonb
language plpgsql
as $$
declare
  v_id uuid;
  v_embedding vector(1536);
begin
  v_embedding := (p_payload->>'embedding')::vector;
  
  insert into thoughts (content, embedding, metadata)
  values (
    p_content,
    v_embedding,
    coalesce(p_payload->'metadata', '{}'::jsonb)
  )
  returning id into v_id;
  
  return jsonb_build_object('id', v_id, 'content', p_content);
end;
$$;
```

### 2.5 — Lock Down Security

New query → paste and Run:

```sql
alter table thoughts enable row level security;

create policy "Service role full access"
  on thoughts
  for all
  using (auth.role() = 'service_role');
```

### 2.6 — Verify

Go to the left sidebar → Table Editor. You should see the `thoughts` table with columns: `id`, `content`, `embedding`, `metadata`, `created_at`, `updated_at`.

Go to Database → Functions. You should see `match_thoughts` and `upsert_thought` listed.

---

## Step 3 — Save Your Connection Details

In Supabase left sidebar → **Settings (gear icon) → API**. Copy into your credential tracker:

- **Project URL** — already auto-populated from your Project Ref (row 3)
- **Secret key** — listed under API keys as the service role key. Click reveal and copy it into row 3. This is your master database key — treat it like a password.

---

## Step 4 — Get an OpenRouter API Key

1. Go to [openrouter.ai](https://openrouter.ai) and create an account
2. Go to [openrouter.ai/keys](https://openrouter.ai/keys)
3. Click **Create Key**, name it `open-brain`
4. Copy the key into your credential tracker immediately (row 4)
5. Add at least **$10 in credits** at [openrouter.ai/credits](https://openrouter.ai/credits)

> **Why $10 and not $5?** The guide says $5, but the actual threshold for unlocking 1,000 requests/day (vs 50/day on the free tier) is $10. At fractions of a cent per memory capture, $10 lasts many months of daily personal use.

OpenRouter is what generates the vector embeddings — converting your memories into mathematical representations that enable semantic search.

---

## Step 5 — Generate Your MCP Access Key

This is a security token that protects your Open Brain MCP server.

**Mac/Linux:**
```bash
openssl rand -hex 32
```

**Windows PowerShell:**
```powershell
-join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
```

Copy the 64-character output and paste into row 5 of your credential tracker (MCP Access Key).

Once you've entered both your Project Ref (Step 1) and Access Key (Step 5), the tracker auto-generates all your terminal commands and connection URLs in Steps 6 and 7.

---

## Step 6 — Deploy the MCP Server (Windows)

Open PowerShell. All commands below must be run from your project folder.

### 6.1 — Create a Project Folder

Create a new folder wherever you keep projects — name it `open-brain`. Open it in File Explorer, click the address bar, copy the full path.

In PowerShell:
```powershell
cd "C:\paste\your\path\here"
Get-Location
```

The output must show your `open-brain` folder path. If it shows anything else, redo the `cd` command.

> If you want to keep this in a GitHub repo, create the folder inside your repo directory and make the repo **private** before pushing anything.

### 6.2 — Install the Supabase CLI

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```
```powershell
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
```
```powershell
scoop bucket add supabase https://github.com/supabase/scoop-bucket.git
```
```powershell
scoop install supabase
```

Verify:
```powershell
supabase --version
```

You should see a version number like `2.x.x`.

### 6.3 — Log In

```powershell
supabase login
```

This opens a browser window. Log in with your Supabase account and approve. PowerShell will confirm "You are now logged in."

### 6.4 — Initialize and Link

```powershell
supabase init
```

Verify the folder was created:
```powershell
dir supabase\
```

Link to your Supabase project (replace with your actual Project Ref):
```powershell
supabase link --project-ref YOUR_PROJECT_REF
```

Enter your Database password when prompted (the one from row 1.5 of your tracker). It won't show as you type — that's normal.

### 6.5 — Set Your Secrets

```powershell
supabase secrets set MCP_ACCESS_KEY=your-access-key-from-step-5
```
```powershell
supabase secrets set OPENROUTER_API_KEY=your-openrouter-key-from-step-4
```

> `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically inside Edge Functions — do NOT set these manually.

### 6.6 — Download the Server Files

```powershell
supabase functions new open-brain-mcp
```
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/jon3nity/OB1/main/server/index.ts -OutFile supabase\functions\open-brain-mcp\index.ts
```
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/jon3nity/OB1/main/server/deno.json -OutFile supabase\functions\open-brain-mcp\deno.json
```

Verify the download worked — this must print the import line, not "Hello from Functions":
```powershell
Get-Content supabase\functions\open-brain-mcp\index.ts -Head 1
```


✅ Correct: `import "jsr:@supabase/functions-js/edge-runtime.d.ts";`
❌ Wrong: `console.log("Hello from Functions!")` — delete the folder and repeat from `supabase functions new`


### 6.7 — Deploy

```powershell
supabase functions deploy open-brain-mcp --no-verify-jwt
```

Verify it's live:
```powershell
supabase functions list
```

You should see `open-brain-mcp` with status **ACTIVE**.

Your MCP server is now live at:
```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp
```

Your MCP Connection URL (with access key):
```
https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp?key=YOUR_ACCESS_KEY
```

Both of these are auto-generated in your credential tracker if you filled in rows 1.8 and 5.

---

## Step 7 — Connect to Your AI Clients

### 7.1 — Claude.ai Web Chat

1. Go to [claude.ai](https://claude.ai) → click your profile icon → **Settings**
2. Click **Connectors** in the left sidebar
3. Click **Add custom connector**
4. Name: `Open Brain`
5. URL: paste your MCP Connection URL (ending in `?key=YOUR_ACCESS_KEY`)
6. Click **Add**

Open Brain now appears in the Connectors list. Enable it per conversation via the **+** button at the bottom of any chat.

### 7.2 — Claude Code

Run this command from any terminal (your project folder or anywhere):
```bash
claude mcp add --transport http open-brain \
  https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp \
  --header "x-brain-key: YOUR_ACCESS_KEY"
```

Config is saved to `C:\Users\YourName\.claude.json`. Open Brain is now available in every Claude Code session.

### 7.3 — Claude Desktop

Same as 7.1 — go to Settings → Connectors → Add custom connector with your MCP Connection URL.

### 7.4 — VS Code (GitHub Copilot)

VS Code uses a dedicated `mcp.json` file at:
```
C:\Users\YourName\AppData\Roaming\Code\User\mcp.json
```

Open it (or create it) and add:
```json
{
    "servers": {
        "open-brain": {
            "type": "http",
            "url": "https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp",
            "headers": {
                "x-brain-key": "YOUR_ACCESS_KEY"
            }
        }
    },
    "inputs": []
}
```

> ⚠️ **Known issue:** VS Code Copilot attempts OAuth authentication against your MCP server before sending headers, which causes 401 errors and repeated "Waiting for server to initialize" messages. This is a VS Code Copilot bug with custom MCP servers that don't support OAuth. The connection still functions for tool calls once established. Use the header approach (not `?key=` in the URL) as it gets further in the auth flow.

### 7.5 — Cursor / Windsurf / Other MCP Clients

Add to your `mcp_settings.json`:
```json
{
  "mcpServers": {
    "open-brain": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://YOUR_PROJECT_REF.supabase.co/functions/v1/open-brain-mcp",
        "--header",
        "x-brain-key:YOUR_ACCESS_KEY"
      ],
      "env": {
        "BRAIN_KEY": "YOUR_ACCESS_KEY"
      }
    }
  }
}
```

> Note: No space after the colon in `x-brain-key:YOUR_ACCESS_KEY`. Some clients mangle spaces inside args.

---

## Step 8 — Enable Open Brain in a Conversation

In claude.ai, Open Brain is not enabled by default in every conversation. To enable it:

1. Open or start any conversation
2. Click the **+** button at the bottom of the chat input
3. Click **Connectors**
4. Toggle **Open Brain** on

You only need to do this once per conversation. Claude will then have access to all four tools: `capture_thought`, `search_thoughts`, `list_thoughts`, `thought_stats`.

---

## Step 9 — Test Your Brain

With Open Brain enabled, ask Claude:

```
How many thoughts do I have in my brain?
```

Claude should call `thought_stats` and return totals. Then test capture:

```
Remember this: I successfully set up Open Brain today.
```

Claude should call `capture_thought` and confirm with extracted metadata (type, topics, people, actions).

Then test search:

```
What did I capture about my setup today?
```

Claude should call `search_thoughts` and return the thought you just saved.

If all three work — your brain is fully operational.

---

## Step 10 — Seed Your Brain

Now that capture works, spend time populating your brain with your most important context. The more you seed it, the more useful it becomes in future sessions.

Suggested categories to capture:

- **Identity** — who you are, where you are, what you're studying or working on
- **Key people** — lecturers, managers, contacts, their emails and roles
- **Projects** — current projects, their status, what approach you're taking
- **Decisions** — major choices you've made and why
- **Credentials and references** — project IDs, application numbers, reference codes
- **Deadlines** — upcoming dates and what they're for
- **Goals** — short and long-term, and what progress you've made

---

## Step 11 — Migrate Old Conversations

To backfill important context from past Claude conversations into Open Brain:

1. Open any old Claude conversation
2. Click the **+** button → **Connectors** → enable **Open Brain**
3. Paste this prompt at the bottom of the conversation:

```
I want you to migrate the key information from our conversation history 
into my Open Brain memory system.

You have access to my Open Brain MCP tools: capture_thought, search_thoughts, 
list_thoughts, thought_stats.

Please go through this entire conversation and capture every important fact, 
decision, name, date, deadline, conclusion, or piece of information that would 
be useful to remember in future sessions.

For each item worth capturing, use capture_thought with a clear standalone 
statement that would make sense to someone reading it without any conversation 
context.

Prioritise:
- Names of people (lecturers, contacts, supervisors)
- Decisions made and why
- Deadlines and dates
- Specific facts (references, IDs, URLs, codes)
- Project details and status
- Action items and next steps
- Any institutional or administrative details

After capturing everything, use thought_stats to confirm how many new thoughts 
were added, then give me a summary of what you captured.
```

Claude will read the full conversation history and capture everything relevant. Return to a new conversation and test with `search_thoughts` to verify the migration worked.

---

## Rate Limits

| OpenRouter Credit Balance | Daily Request Limit |
|---|---|
| $0 (free tier) | 50 requests/day |
| $10+ purchased | 1,000 requests/day |

Each thought capture uses roughly 1–2 requests. Each search uses 1 request. For active daily personal use, add $10 to OpenRouter once — it unlocks 1,000 requests/day and will last months at fractions of a cent per capture.

Daily limits reset at midnight UTC (1:00am Irish time).

---

## Setting Up Auto-Capture in Claude Projects

To make Claude automatically search and capture memories in every conversation, create a Claude Project and add this to the **Project Instructions**:

```
You have access to my Open Brain memory system via MCP.

At the start of every conversation, search my brain for relevant context 
using search_thoughts before responding to anything important.

At the end of every conversation, or whenever I share something important 
(a decision, a name, a fact, a deadline, a person), capture it immediately 
to my brain using capture_thought.

Never wait for me to ask — capture proactively. Treat my brain as the 
permanent record of everything we discuss.
```

This means every conversation inside that project will search your brain for context at the start and capture important details throughout — automatically.

---

## Troubleshooting

**`upsert_thought` function not found**
Run the SQL in Step 2.4. This function is required by the MCP server but not included in the base Open Brain migration script.

**`match_thoughts` function not found / wrong signature**
Drop and recreate using the SQL in Step 2.3. The MCP server expects the 4-parameter version with the `filter jsonb` argument.

**401 errors / Invalid or missing access key**
Your `MCP_ACCESS_KEY` in Supabase secrets doesn't match what you're using in the connection URL or header. Re-run `supabase secrets set MCP_ACCESS_KEY=YOUR_KEY` with the exact same value as in your credential tracker.

**Search returns no results**
Try lowering the threshold: ask Claude to "search with threshold 0.3" for a wider match. Also ensure you have captured at least one thought first.

**`$$` dollar-quote syntax error in SQL**
The dollar-quote delimiter `$$` got corrupted to `$` during copy-paste. Replace every single `$` delimiter with `$$` in the function body wrapper.

**VS Code showing repeated "Waiting for server to initialize"**
This is a known VS Code Copilot OAuth bug with custom MCP servers. Use the header-based config in `mcp.json` (not the `?key=` URL approach) and it will get further in the auth flow.

---

## What You've Built

A personal AI memory layer that:

- Lives in **your own database** — not OpenAI's cloud, not Anthropic's servers, not anyone else's infrastructure
- Works across **every Claude surface** — web chat, Claude Code, Claude Desktop, VS Code
- Retrieves by **meaning, not keywords** — "find anything about my career plans" works even if you never used those exact words
- **Compounds over time** — every thought you capture makes every future session smarter
- Costs **almost nothing** — Supabase free tier + $10 OpenRouter credit lasting months

---

*Setup completed: May 1–2, 2026*
*Supabase region: West EU (Ireland)*
*MCP server: Supabase Edge Functions (Deno)*
*Embedding model: via OpenRouter (1536 dimensions)*
*Vector search: pgvector HNSW cosine similarity*
