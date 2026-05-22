---
type: observation
domain: [store, synthesize, schema]
confidence: 0.85
sources: 0
entities: [pipeline_watermarks, pipeline_sessions, pipeline_work_items, PipelineIndex, PipelineSession]
refs: ['src://knomit/.claude/plans/2026-03-21-hypothesize-design.md@0938d83']
---
# review_* tables renamed to pipeline_*; sessions carry a `tool` field for sharing infra

Pipeline infrastructure is shared across knomit_review and knomit_hypothesize. The original review_* tables were renamed to pipeline_*: pipeline_watermarks(tool, branch, commit_hash), pipeline_sessions(id, tool, branch, status, ...), pipeline_work_items(id, session_id, step_type, cluster_key, facts_json, response, priority, depth, created_at). The session's `tool` field distinguishes which tool owns it ('review' or 'hypothesize'). Go types: ReviewSession→PipelineSession, ReviewWorkItem→PipelineWorkItem, ReviewIndex→PipelineIndex. Methods scoped by tool gain a `tool` parameter (watermarks, sessions, GC). New tools share the same machinery by passing their tool name. internal/store/review.go renamed to internal/store/pipeline.go.
