---
name: masster
description: Use when working with MASSter mass-spectrometry analyses, including Sample or Study workflows, feature detection, isotope/adduct/MS2 processing, identification, semantic schemas, validation, automation, or related repository changes. Read the repository-backed guide before choosing operations or modifying public workflow behavior.
---

# MASSter

Read [the MASSter Agent Guide](../../docs/agent/masster-agent-guide.md) before
working with MASSter objects or APIs.

Use the public `masster` namespace. Inspect `Sample` and `Study` state with
`describe_*` and `validate_*` helpers before invoking processing operations.
Follow the guide's workflow order, preserve public compatibility, and keep
`find_iso()` as the canonical isotope-detection method.
