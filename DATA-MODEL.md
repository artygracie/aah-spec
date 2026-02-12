# Artyfacts Data Model

**Version:** 0.1.0  
**Date:** 2026-02-12

This document defines the database schema for Artyfacts, including versioning and content deduplication.

---

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        PostgreSQL                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │organizations │───▶│   projects   │───▶│   artifacts  │      │
│  └──────────────┘    └──────────────┘    └──────┬───────┘      │
│                                                  │               │
│         ┌────────────────────┬──────────────────┼───────┐       │
│         ▼                    ▼                  ▼       ▼       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   versions   │    │    tags      │    │relationships │      │
│  └──────┬───────┘    └──────────────┘    └──────────────┘      │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   blobs      │◀─── Content deduplication via hash            │
│  └──────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   S3 / R2        │
                    │   (blob storage) │
                    └──────────────────┘
```

---

## Tables

### organizations

Top-level tenant for multi-org support.

```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  
  -- Settings
  default_retention TEXT DEFAULT '30d',
  default_visibility TEXT DEFAULT 'team',
  
  -- Limits
  artifact_limit_monthly INTEGER DEFAULT 100,
  storage_limit_bytes BIGINT DEFAULT 1073741824, -- 1GB
  
  -- Timestamps
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
```

### projects

Optional grouping within an organization (maps to repos, products, etc.).

```sql
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  description TEXT,
  
  -- Settings (override org defaults)
  default_retention TEXT,
  default_visibility TEXT,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(org_id, slug)
);

CREATE INDEX idx_projects_org_id ON projects(org_id);
```

### artifacts

Core artifact metadata. Each artifact can have multiple versions.

```sql
CREATE TABLE artifacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  project_id UUID REFERENCES projects(id) ON DELETE SET NULL,
  
  -- External references
  external_id TEXT,              -- Agent-provided ID (e.g., "aah_local_123")
  
  -- Artifact identity
  type TEXT NOT NULL,            -- e.g., "document/markdown"
  title TEXT,
  summary TEXT,
  
  -- Current version pointer
  current_version_id UUID,       -- Points to latest version (set after insert)
  version_count INTEGER DEFAULT 1,
  
  -- Source provenance
  agent_id TEXT,
  agent_name TEXT,
  agent_role TEXT,
  framework TEXT,
  framework_version TEXT,
  session_id TEXT,
  task_id TEXT,
  task_description TEXT,
  model TEXT,
  
  -- Lifecycle
  retention TEXT DEFAULT '30d',
  visibility TEXT DEFAULT 'team', -- private, team, organization, public
  status TEXT DEFAULT 'draft',    -- draft, review, approved, superseded, archived
  expires_at TIMESTAMPTZ,
  
  -- Timestamps
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(org_id, external_id)
);

CREATE INDEX idx_artifacts_org_id ON artifacts(org_id);
CREATE INDEX idx_artifacts_project_id ON artifacts(project_id);
CREATE INDEX idx_artifacts_session_id ON artifacts(session_id);
CREATE INDEX idx_artifacts_task_id ON artifacts(task_id);
CREATE INDEX idx_artifacts_agent_id ON artifacts(agent_id);
CREATE INDEX idx_artifacts_type ON artifacts(type);
CREATE INDEX idx_artifacts_status ON artifacts(status);
CREATE INDEX idx_artifacts_created_at ON artifacts(created_at DESC);
CREATE INDEX idx_artifacts_expires_at ON artifacts(expires_at) WHERE expires_at IS NOT NULL;
```

### versions

Individual versions of an artifact. Content is stored via blob reference.

```sql
CREATE TABLE versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  
  -- Version info
  version_number INTEGER NOT NULL,
  
  -- Content reference (deduped via blobs table)
  blob_id UUID NOT NULL REFERENCES blobs(id),
  
  -- Content metadata
  media_type TEXT NOT NULL,       -- MIME type
  encoding TEXT DEFAULT 'utf-8',
  size_bytes INTEGER,
  token_count INTEGER,
  
  -- Version metadata
  change_summary TEXT,            -- Optional: what changed from previous version
  created_by TEXT,                -- Agent or user who created this version
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(artifact_id, version_number)
);

CREATE INDEX idx_versions_artifact_id ON versions(artifact_id);
CREATE INDEX idx_versions_blob_id ON versions(blob_id);
```

### blobs

Content-addressed blob storage. Deduplicates identical content via SHA-256 hash.

```sql
CREATE TABLE blobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Content hash for deduplication
  content_hash TEXT UNIQUE NOT NULL,  -- SHA-256 hash
  
  -- Storage location
  storage_url TEXT NOT NULL,          -- S3/R2 URL
  storage_bucket TEXT NOT NULL,
  storage_key TEXT NOT NULL,
  
  -- Metadata
  size_bytes INTEGER NOT NULL,
  media_type TEXT NOT NULL,
  
  -- Reference counting for cleanup
  reference_count INTEGER DEFAULT 1,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_blobs_content_hash ON blobs(content_hash);
```

### tags

Freeform tags for categorization and filtering.

```sql
CREATE TABLE tags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  tag TEXT NOT NULL,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(artifact_id, tag)
);

CREATE INDEX idx_tags_artifact_id ON tags(artifact_id);
CREATE INDEX idx_tags_tag ON tags(tag);
```

### relationships

Artifact lineage and relationships.

```sql
CREATE TYPE relationship_type AS ENUM (
  'derived_from',    -- This artifact was created from the parent
  'supersedes',      -- This artifact replaces the parent
  'references',      -- This artifact references the parent
  'responds_to'      -- This artifact is a response/handoff to parent
);

CREATE TABLE relationships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  parent_artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  child_artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  
  relationship relationship_type NOT NULL DEFAULT 'derived_from',
  
  -- Optional: specific versions involved
  parent_version_id UUID REFERENCES versions(id),
  child_version_id UUID REFERENCES versions(id),
  
  -- Context
  context JSONB,  -- Additional relationship metadata
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(parent_artifact_id, child_artifact_id, relationship)
);

CREATE INDEX idx_relationships_parent ON relationships(parent_artifact_id);
CREATE INDEX idx_relationships_child ON relationships(child_artifact_id);
```

### handoffs

Active handoff requests between agents.

```sql
CREATE TYPE handoff_status AS ENUM (
  'pending',      -- Waiting for target agent
  'accepted',     -- Target agent acknowledged
  'completed',    -- Target agent produced response
  'expired',      -- Deadline passed without response
  'cancelled'     -- Source agent cancelled
);

CREATE TABLE handoffs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  version_id UUID REFERENCES versions(id),
  
  -- Target
  target_agent_id TEXT,
  target_agent_role TEXT,
  
  -- Status
  status handoff_status DEFAULT 'pending',
  
  -- Expectations
  expects_response BOOLEAN DEFAULT false,
  response_deadline TIMESTAMPTZ,
  priority TEXT DEFAULT 'normal',  -- low, normal, high, urgent
  
  -- Context for receiving agent
  context JSONB,
  
  -- Response (if completed)
  response_artifact_id UUID REFERENCES artifacts(id),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_handoffs_artifact_id ON handoffs(artifact_id);
CREATE INDEX idx_handoffs_target_agent ON handoffs(target_agent_id);
CREATE INDEX idx_handoffs_target_role ON handoffs(target_agent_role);
CREATE INDEX idx_handoffs_status ON handoffs(status);
```

### api_keys

Authentication for API access.

```sql
CREATE TABLE api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  
  -- Key info
  name TEXT NOT NULL,
  key_hash TEXT UNIQUE NOT NULL,  -- SHA-256 of actual key
  key_prefix TEXT NOT NULL,       -- First 8 chars for identification
  
  -- Permissions
  scopes TEXT[] DEFAULT ARRAY['read', 'write'],
  
  -- Limits
  rate_limit_rpm INTEGER DEFAULT 60,
  
  -- Status
  is_active BOOLEAN DEFAULT true,
  expires_at TIMESTAMPTZ,
  last_used_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_api_keys_org_id ON api_keys(org_id);
CREATE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
```

### share_links

Public/time-limited shareable links.

```sql
CREATE TABLE share_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  artifact_id UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
  version_id UUID REFERENCES versions(id),  -- NULL = always latest
  
  -- Link token (unguessable)
  token TEXT UNIQUE NOT NULL,
  
  -- Access control
  requires_auth BOOLEAN DEFAULT false,
  allowed_emails TEXT[],
  
  -- Expiration
  expires_at TIMESTAMPTZ,
  max_views INTEGER,
  view_count INTEGER DEFAULT 0,
  
  -- Status
  is_active BOOLEAN DEFAULT true,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_viewed_at TIMESTAMPTZ
);

CREATE INDEX idx_share_links_artifact_id ON share_links(artifact_id);
CREATE UNIQUE INDEX idx_share_links_token ON share_links(token);
```

---

## Key Operations

### Upload Artifact (with deduplication)

```sql
-- 1. Hash the content
-- content_hash = SHA256(content)

-- 2. Check if blob exists
SELECT id FROM blobs WHERE content_hash = $content_hash;

-- 3a. If exists, increment reference count
UPDATE blobs SET reference_count = reference_count + 1 WHERE id = $blob_id;

-- 3b. If not exists, upload to S3 and create blob record
INSERT INTO blobs (content_hash, storage_url, storage_bucket, storage_key, size_bytes, media_type)
VALUES ($content_hash, $s3_url, $bucket, $key, $size, $media_type)
RETURNING id;

-- 4. Create artifact
INSERT INTO artifacts (org_id, external_id, type, title, agent_id, ...)
VALUES ($org_id, $external_id, $type, $title, $agent_id, ...)
RETURNING id;

-- 5. Create version
INSERT INTO versions (artifact_id, version_number, blob_id, media_type, size_bytes, token_count)
VALUES ($artifact_id, 1, $blob_id, $media_type, $size, $tokens)
RETURNING id;

-- 6. Update artifact with current version
UPDATE artifacts SET current_version_id = $version_id WHERE id = $artifact_id;
```

### Create New Version

```sql
-- 1. Hash content, find or create blob (same as upload)

-- 2. Get current version number
SELECT version_count FROM artifacts WHERE id = $artifact_id;

-- 3. Create new version
INSERT INTO versions (artifact_id, version_number, blob_id, media_type, size_bytes, token_count, change_summary)
VALUES ($artifact_id, $version_count + 1, $blob_id, $media_type, $size, $tokens, $change_summary)
RETURNING id;

-- 4. Update artifact
UPDATE artifacts 
SET current_version_id = $version_id, 
    version_count = version_count + 1,
    updated_at = NOW()
WHERE id = $artifact_id;
```

### Get Artifact with Lineage

```sql
-- Get artifact with current version content
SELECT 
  a.*,
  v.version_number,
  v.media_type,
  v.size_bytes,
  v.token_count,
  b.storage_url
FROM artifacts a
JOIN versions v ON v.id = a.current_version_id
JOIN blobs b ON b.id = v.blob_id
WHERE a.id = $artifact_id;

-- Get lineage (parents)
SELECT 
  r.relationship,
  pa.id as parent_id,
  pa.title as parent_title,
  pa.type as parent_type
FROM relationships r
JOIN artifacts pa ON pa.id = r.parent_artifact_id
WHERE r.child_artifact_id = $artifact_id;

-- Get lineage (children)
SELECT 
  r.relationship,
  ca.id as child_id,
  ca.title as child_title,
  ca.type as child_type
FROM relationships r
JOIN artifacts ca ON ca.id = r.child_artifact_id
WHERE r.parent_artifact_id = $artifact_id;
```

### Cleanup Expired Artifacts

```sql
-- Run periodically (cron job)

-- 1. Find expired artifacts
SELECT id FROM artifacts 
WHERE expires_at < NOW() 
AND status != 'archived';

-- 2. For each, decrement blob reference counts
UPDATE blobs b
SET reference_count = reference_count - 1
FROM versions v
WHERE v.blob_id = b.id
AND v.artifact_id = ANY($expired_artifact_ids);

-- 3. Delete artifacts (cascades to versions, tags, relationships)
DELETE FROM artifacts WHERE id = ANY($expired_artifact_ids);

-- 4. Delete orphaned blobs (reference_count = 0)
DELETE FROM blobs WHERE reference_count <= 0;

-- 5. Delete corresponding S3 objects
-- (handled in application code after getting storage_keys)
```

---

## Indexes Summary

| Table | Index | Purpose |
|-------|-------|---------|
| artifacts | org_id | Filter by organization |
| artifacts | session_id | Filter by agent session |
| artifacts | task_id | Filter by task/ticket |
| artifacts | type | Filter by artifact type |
| artifacts | created_at DESC | Recent artifacts |
| artifacts | expires_at | Cleanup job |
| versions | artifact_id | Get versions for artifact |
| blobs | content_hash | Deduplication lookup |
| tags | tag | Filter by tag |
| relationships | parent/child | Lineage traversal |
| share_links | token | Link resolution |

---

## Storage Estimates

| Artifacts/month | Avg size | Dedup rate | Storage/month | Storage/year |
|-----------------|----------|------------|---------------|--------------|
| 1,000 | 10KB | 30% | 7MB | 84MB |
| 10,000 | 10KB | 30% | 70MB | 840MB |
| 100,000 | 10KB | 30% | 700MB | 8.4GB |
| 1,000,000 | 10KB | 30% | 7GB | 84GB |

Deduplication significantly reduces storage for agents that iterate on similar content.

---

## Migration Strategy

```sql
-- migrations/001_initial_schema.sql
-- Contains all CREATE TABLE statements above

-- migrations/002_add_full_text_search.sql
ALTER TABLE artifacts ADD COLUMN search_vector tsvector;
CREATE INDEX idx_artifacts_search ON artifacts USING gin(search_vector);

-- Update trigger for search
CREATE FUNCTION artifacts_search_trigger() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := 
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.summary, '')), 'B');
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER artifacts_search_update
  BEFORE INSERT OR UPDATE ON artifacts
  FOR EACH ROW EXECUTE FUNCTION artifacts_search_trigger();
```
