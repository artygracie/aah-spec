# Artyfacts Implementation Plan

**Date:** 2026-02-12  
**Goal:** Integrate Idea Factory with AAH spec → Build Artyfacts MVP

---

## Phase 1: AAH Envelope Wrapper (Idea Factory)

### Current State
Idea Factory's `OutputRouter` already has an `ArtifactHandler` interface:

```typescript
export interface ArtifactHandler {
  save(artifact: Artifact): Promise<string>;  // Returns saved path
}

export interface Artifact {
  agentId: string;
  project?: string;
  type: string;
  path: string;
  content: string;
}
```

The current `FileArtifactHandler` just dumps files to disk. We need a new `ArtyfactsHandler` that:
1. Wraps content in AAH envelope
2. Uploads to Artyfacts API
3. Returns the shareable URL

### New File: `lib/output-router/artyfacts-handler.ts`

```typescript
import crypto from 'node:crypto';
import type { ArtifactHandler, Artifact } from './index.js';

export interface ArtyfactsConfig {
  apiUrl: string;         // https://artyfacts.dev/api or localhost:3000/api
  apiKey: string;         // API key for auth
  orgId?: string;         // Organization ID (for multi-tenant)
  projectSlug?: string;   // Default project
}

export interface AAHEnvelope {
  aah_version: string;
  artifact: {
    id: string;
    type: string;
    title?: string;
    created_at: string;
  };
  source: {
    agent_id: string;
    agent_role?: string;
    framework: string;
    framework_version?: string;
    session_id?: string;
    task_id?: string;
    model?: string;
  };
  content: {
    media_type: string;
    encoding: string;
    body: string;
    body_hash: string;
    size_bytes: number;
    token_count?: number;
  };
  lifecycle?: {
    retention: string;
    visibility: string;
    status: string;
    tags?: string[];
  };
}

export class ArtyfactsHandler implements ArtifactHandler {
  private config: ArtyfactsConfig;

  constructor(config: ArtyfactsConfig) {
    this.config = config;
  }

  async save(artifact: Artifact): Promise<string> {
    // 1. Build AAH envelope
    const envelope = this.wrapInAAH(artifact);

    // 2. Upload to Artyfacts API
    const response = await fetch(`${this.config.apiUrl}/v1/artifacts`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.config.apiKey}`,
      },
      body: JSON.stringify(envelope),
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({ error: response.statusText }));
      throw new Error(`Artyfacts upload failed: ${JSON.stringify(error)}`);
    }

    const result = await response.json() as { id: string; url: string };
    
    console.log(`[ArtyfactsHandler] Uploaded: ${result.url}`);
    return result.url;  // Return shareable URL instead of local path
  }

  private wrapInAAH(artifact: Artifact): AAHEnvelope {
    const contentHash = crypto
      .createHash('sha256')
      .update(artifact.content)
      .digest('hex');

    // Infer media type from artifact type/path
    const mediaType = this.inferMediaType(artifact.type, artifact.path);

    return {
      aah_version: '0.1',
      artifact: {
        id: `aah_if_${crypto.randomUUID()}`,
        type: this.mapArtifactType(artifact.type),
        title: this.inferTitle(artifact.path),
        created_at: new Date().toISOString(),
      },
      source: {
        agent_id: artifact.agentId,
        agent_role: this.inferRole(artifact.agentId),
        framework: 'idea-factory',
        framework_version: '1.0.0',
        // These could be passed through context
        // session_id: ...,
        // task_id: ...,
      },
      content: {
        media_type: mediaType,
        encoding: 'utf-8',
        body: artifact.content,
        body_hash: `sha256:${contentHash}`,
        size_bytes: Buffer.byteLength(artifact.content, 'utf-8'),
      },
      lifecycle: {
        retention: '30d',
        visibility: 'team',
        status: 'draft',
        tags: artifact.project ? [artifact.project] : [],
      },
    };
  }

  private inferMediaType(type: string, path: string): string {
    if (type === 'markdown' || path.endsWith('.md')) return 'text/markdown';
    if (type === 'json' || path.endsWith('.json')) return 'application/json';
    if (type === 'yaml' || path.endsWith('.yaml') || path.endsWith('.yml')) return 'text/yaml';
    if (type === 'typescript' || path.endsWith('.ts')) return 'text/typescript';
    if (type === 'javascript' || path.endsWith('.js')) return 'text/javascript';
    return 'text/plain';
  }

  private mapArtifactType(type: string): string {
    const mapping: Record<string, string> = {
      markdown: 'document/markdown',
      json: 'data/json',
      yaml: 'data/yaml',
      prd: 'structured/spec',
      research: 'structured/analysis',
      typescript: 'code/typescript',
      javascript: 'code/javascript',
    };
    return mapping[type] || 'document/text';
  }

  private inferTitle(path: string): string {
    // signs-banner-placement-recommendations.md → Signs Banner Placement Recommendations
    const filename = path.split('/').pop() || path;
    return filename
      .replace(/\.[^.]+$/, '')          // Remove extension
      .replace(/[-_]/g, ' ')            // Replace dashes/underscores with spaces
      .replace(/\b\w/g, c => c.toUpperCase());  // Title case
  }

  private inferRole(agentId: string): string {
    if (agentId.includes('research')) return 'researcher';
    if (agentId.includes('engineer') || agentId.includes('dev')) return 'developer';
    if (agentId.includes('pm')) return 'pm';
    if (agentId.includes('qa')) return 'qa';
    return 'agent';
  }
}
```

### Integration Point

In Idea Factory's router initialization (wherever `OutputRouter` is configured), swap the handler:

```typescript
import { ArtyfactsHandler } from './output-router/artyfacts-handler.js';

const outputRouter = new OutputRouter({
  eventFeed,
  handlers: {
    artifacts: new ArtyfactsHandler({
      apiUrl: process.env.ARTYFACTS_API_URL || 'https://artyfacts.dev/api',
      apiKey: process.env.ARTYFACTS_API_KEY || '',
      projectSlug: 'idea-factory',
    }),
    // ... other handlers
  },
});
```

---

## Phase 2: Artyfacts API (MVP)

### Stack
- **Runtime:** Next.js API routes on Vercel
- **Database:** Supabase Postgres (you already have projects there)
- **Blob storage:** Cloudflare R2 (cheap, S3-compatible)
- **Auth:** API keys (simple for v1)

### MVP Endpoints

```
POST   /api/v1/artifacts          - Upload artifact
GET    /api/v1/artifacts/:id      - Get artifact metadata
GET    /api/v1/artifacts/:id/raw  - Get raw content
GET    /api/v1/artifacts          - List artifacts (filter by session, agent, project)
DELETE /api/v1/artifacts/:id      - Delete artifact

GET    /a/:id                     - Public viewer page (shareable URL)
```

### Database Schema (Simplified for MVP)

Use a subset of the full data model - skip versioning and relationships for v1:

```sql
-- MVP tables only
CREATE TABLE artifacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_id TEXT UNIQUE,        -- AAH envelope artifact.id
  
  -- Core fields
  type TEXT NOT NULL,
  title TEXT,
  summary TEXT,
  
  -- Source
  agent_id TEXT,
  agent_role TEXT,
  framework TEXT,
  session_id TEXT,
  task_id TEXT,
  
  -- Content (inline for MVP, move to blobs later)
  media_type TEXT NOT NULL,
  content TEXT NOT NULL,          -- Inline for v1
  content_hash TEXT,
  size_bytes INTEGER,
  token_count INTEGER,
  
  -- Lifecycle
  retention TEXT DEFAULT '30d',
  visibility TEXT DEFAULT 'team',
  status TEXT DEFAULT 'draft',
  expires_at TIMESTAMPTZ,
  
  -- Timestamps
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tags (
  artifact_id UUID REFERENCES artifacts(id) ON DELETE CASCADE,
  tag TEXT NOT NULL,
  PRIMARY KEY (artifact_id, tag)
);

CREATE TABLE api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  key_hash TEXT UNIQUE NOT NULL,
  key_prefix TEXT NOT NULL,       -- First 8 chars for display
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_artifacts_session ON artifacts(session_id);
CREATE INDEX idx_artifacts_agent ON artifacts(agent_id);
CREATE INDEX idx_artifacts_created ON artifacts(created_at DESC);
CREATE INDEX idx_artifacts_expires ON artifacts(expires_at) WHERE expires_at IS NOT NULL;
```

### API Implementation Sketch

**`/api/v1/artifacts/route.ts`** (Next.js App Router)

```typescript
import { createClient } from '@supabase/supabase-js';
import { NextResponse } from 'next/server';
import crypto from 'crypto';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

export async function POST(request: Request) {
  // 1. Verify API key
  const apiKey = request.headers.get('Authorization')?.replace('Bearer ', '');
  if (!apiKey || !(await verifyApiKey(apiKey))) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // 2. Parse AAH envelope
  const envelope = await request.json();
  
  // 3. Validate envelope
  if (envelope.aah_version !== '0.1' || !envelope.artifact?.id) {
    return NextResponse.json({ error: 'Invalid AAH envelope' }, { status: 400 });
  }

  // 4. Calculate expiration
  const expiresAt = calculateExpiration(envelope.lifecycle?.retention || '30d');

  // 5. Insert into database
  const { data, error } = await supabase
    .from('artifacts')
    .insert({
      external_id: envelope.artifact.id,
      type: envelope.artifact.type,
      title: envelope.artifact.title,
      agent_id: envelope.source?.agent_id,
      agent_role: envelope.source?.agent_role,
      framework: envelope.source?.framework,
      session_id: envelope.source?.session_id,
      task_id: envelope.source?.task_id,
      media_type: envelope.content.media_type,
      content: envelope.content.body,
      content_hash: envelope.content.body_hash,
      size_bytes: envelope.content.size_bytes,
      token_count: envelope.content.token_count,
      retention: envelope.lifecycle?.retention,
      visibility: envelope.lifecycle?.visibility,
      status: envelope.lifecycle?.status,
      expires_at: expiresAt,
    })
    .select('id')
    .single();

  if (error) {
    console.error('Insert error:', error);
    return NextResponse.json({ error: 'Failed to save artifact' }, { status: 500 });
  }

  // 6. Insert tags
  if (envelope.lifecycle?.tags?.length) {
    await supabase.from('tags').insert(
      envelope.lifecycle.tags.map((tag: string) => ({
        artifact_id: data.id,
        tag,
      }))
    );
  }

  // 7. Return shareable URL
  const url = `${process.env.NEXT_PUBLIC_APP_URL}/a/${data.id}`;
  
  return NextResponse.json({ id: data.id, url });
}

async function verifyApiKey(key: string): Promise<boolean> {
  const keyHash = crypto.createHash('sha256').update(key).digest('hex');
  const { data } = await supabase
    .from('api_keys')
    .select('id')
    .eq('key_hash', keyHash)
    .eq('is_active', true)
    .single();
  return !!data;
}

function calculateExpiration(retention: string): Date | null {
  if (retention === 'permanent') return null;
  const match = retention.match(/^(\d+)([dhm])$/);
  if (!match) return null;
  
  const [, amount, unit] = match;
  const ms = {
    d: 24 * 60 * 60 * 1000,
    h: 60 * 60 * 1000,
    m: 60 * 1000,
  }[unit]!;
  
  return new Date(Date.now() + parseInt(amount) * ms);
}
```

### Viewer Page

**`/app/a/[id]/page.tsx`**

```typescript
import { createClient } from '@supabase/supabase-js';
import { notFound } from 'next/navigation';
import { MarkdownRenderer } from '@/components/markdown-renderer';
import { CodeRenderer } from '@/components/code-renderer';

export default async function ArtifactPage({ params }: { params: { id: string } }) {
  const supabase = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_KEY!
  );

  const { data: artifact } = await supabase
    .from('artifacts')
    .select('*')
    .eq('id', params.id)
    .single();

  if (!artifact) notFound();

  // Check visibility (for MVP, all artifacts are viewable with link)
  // Later: add auth checks based on visibility

  return (
    <div className="max-w-4xl mx-auto p-8">
      {/* Header */}
      <div className="mb-8">
        <h1 className="text-2xl font-bold">{artifact.title || 'Untitled Artifact'}</h1>
        <div className="text-sm text-gray-500 mt-2">
          Created by <span className="font-medium">{artifact.agent_id}</span>
          {' • '}
          {new Date(artifact.created_at).toLocaleString()}
        </div>
        {artifact.agent_role && (
          <span className="inline-block mt-2 px-2 py-1 bg-blue-100 text-blue-800 text-xs rounded">
            {artifact.agent_role}
          </span>
        )}
      </div>

      {/* Content */}
      <div className="prose prose-lg max-w-none">
        {artifact.media_type === 'text/markdown' ? (
          <MarkdownRenderer content={artifact.content} />
        ) : artifact.media_type.startsWith('text/') ? (
          <CodeRenderer 
            content={artifact.content} 
            language={artifact.media_type.split('/')[1]} 
          />
        ) : (
          <pre className="bg-gray-100 p-4 rounded overflow-auto">
            {artifact.content}
          </pre>
        )}
      </div>

      {/* Footer */}
      <div className="mt-8 pt-4 border-t text-sm text-gray-500">
        <div>Framework: {artifact.framework}</div>
        <div>Type: {artifact.type}</div>
        {artifact.session_id && <div>Session: {artifact.session_id}</div>}
        {artifact.token_count && <div>Tokens: {artifact.token_count.toLocaleString()}</div>}
      </div>
    </div>
  );
}
```

---

## Phase 3: Integration & Testing

### Steps

1. **Create Supabase project** (or use existing)
   - Run MVP schema migration
   - Generate API key

2. **Create Next.js app**
   ```bash
   npx create-next-app@latest artyfacts --typescript --tailwind --app
   cd artyfacts
   npm install @supabase/supabase-js
   ```

3. **Deploy to Vercel**
   - Connect repo
   - Set env vars: `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `NEXT_PUBLIC_APP_URL`

4. **Update Idea Factory**
   - Add `ArtyfactsHandler`
   - Set `ARTYFACTS_API_URL` and `ARTYFACTS_API_KEY` env vars
   - Test with a research task

5. **Validate end-to-end**
   - Idea Factory runs task
   - Agent writes artifact
   - Artyfacts receives AAH envelope
   - Shareable URL returned
   - URL posted to Slack
   - You click and see beautiful rendered content 🎉

---

## What's Deferred to v2

- [ ] Versioning (version history, diffs)
- [ ] Relationships (lineage graph)
- [ ] Blob storage (move content to R2)
- [ ] Deduplication (hash-based)
- [ ] Organizations / multi-tenant
- [ ] Comments / reactions
- [ ] "Promote to GitHub" action
- [ ] Slack unfurling
- [ ] Full-text search

---

## Timeline Estimate

| Phase | Effort | Output |
|-------|--------|--------|
| Phase 1: AAH wrapper | 2-3 hours | `artyfacts-handler.ts` in Idea Factory |
| Phase 2: Artyfacts API | 4-6 hours | Working upload + viewer |
| Phase 3: Integration | 1-2 hours | End-to-end flow working |

**Total: ~1 day to MVP**

---

## Questions Before Starting

1. **Supabase project:** Use existing (Idea Factory's?) or create new `artyfacts` project?
2. **Domain:** artyfacts.dev? artyfacts.artygroup.com? localhost for now?
3. **API key management:** Generate one key for now, or build key management UI?
