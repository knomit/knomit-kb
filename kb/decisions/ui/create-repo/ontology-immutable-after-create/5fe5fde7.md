---
kind: pragmatic
type: policy
domain: [ui, repos, ontology, fact, lifecycle]
confidence: 0.95
sources: 1
entities: [repoBuilder.loadOntology, Ontology.IsSubsetOf, fact.EmbeddedPresetByID, repos.OntologyPath, internal/repos/builder.go, internal/fact/ontology.go]
refs: ['src://7b4887ce51d9/internal/repos/builder.go@b3f65a99179a88c37665f99f50403fef3af05a62:18a0cc46be27a0ca611bebdf50558dcbc802ae8a', 'src://7b4887ce51d9/internal/fact/ontology.go@b3f65a99179a88c37665f99f50403fef3af05a62:331a061bdc1861dafca160146b49a6a798925a7e', 'kb://3ec012f5b4d2/kb/decisions/repos/kb-repo-layout/ontology-dot-domains-fallback/318f619a.md', 'kb://3ec012f5b4d2/kb/decisions/fact/per-repo-ontology/0be51fec.md']
---
# A repo's ontology is chosen ONCE at create time and is never editable afterwards — arbitrary edits are a schema migration, and the only sanctioned change stays the strictly-additive preset upgrade

# The decision

The create wizard is the ONLY place a repo's ontology is authored. There is no post-creation ontology editor in Settings or anywhere else.

# Rationale

Removing or renaming a topic orphans every fact filed under it. That is a schema migration — it needs a fact-rewriting plan, not a text box. Offering an editor without that machinery would let a user break their own corpus with a one-character edit.

The codebase already enforces exactly this principle in the one place it does mutate a stored ontology: the boot-time preset refresh in `repoBuilder.loadOntology` upgrades a stored ontology to a newer embedded preset ONLY when `ont.IsSubsetOf(preset)` holds — i.e. only when the change is strictly ADDITIVE and therefore cannot orphan anything. Topics are never removed or renamed under existing facts.

# What this does NOT mean

It does NOT mean the stored ontology never changes. Preset-derived ontologies continue to receive additive upgrades automatically at boot, and that behaviour is unaffected. The rule bars USER-AUTHORED edits after creation, not additive machine upgrades that are already proven safe by the subset check.

It also does not bar a future migration feature — it bars shipping the editor WITHOUT one.

# Consequence for the wizard

The review step must say plainly that the ontology cannot be changed afterwards, since this is a one-way door taken at create time. It is stated as plain text, not a warning: warning styling in this wizard is reserved for things that actually fail.
