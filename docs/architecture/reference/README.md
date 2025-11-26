# Architecture Reference Documentation

**Quick reference materials for the Kubernetes API Extensions API Server architecture.**

## Overview

This directory contains reference materials extracted from the comprehensive architecture analysis. These documents are designed for quick lookups, containing metrics, tables, and metadata without lengthy explanations.

**Analysis Date**: 2025-11-25
**Overall Confidence**: 90% (High)
**Codebase Size**: 119,053 LOC across 296 Go files

## Reference Documents

### [dependencies.md](./dependencies.md)
Core dependencies, integration points, dependency graph, and version constraints.

**Use when**: Understanding external dependencies, troubleshooting integration issues, planning upgrades.

### [critical-paths.md](./critical-paths.md)
Critical files table, hot paths, LOC metrics per component, and performance-sensitive code locations.

**Use when**: Optimizing performance, understanding request flows, prioritizing code reviews.

### [confidence-assessment.md](./confidence-assessment.md)
Per-domain confidence ratings, analysis methodology, and known gaps in understanding.

**Use when**: Evaluating reliability of architecture documentation, identifying areas needing deeper investigation.

### [future-considerations.md](./future-considerations.md)
Potential improvements, API evolution roadmap, feature requests, and research directions.

**Use when**: Planning enhancements, evaluating technical debt, considering architectural changes.

## Related Documentation

### Comprehensive Documentation
- **[Architecture Overview](../architecture.md)** - Complete architecture analysis with detailed explanations
- **[Domain Findings](../../.claude/codebase/codebase-dedfc2ae-20251125-171241/findings/)** - Deep dives into specific architectural domains

### Developer Guides
- **[CLAUDE.md](../../../CLAUDE.md)** - Quick reference for Claude Code with critical components summary
- **[README.md](../../../README.md)** - Project overview and getting started guide

## Using This Reference

These reference documents are optimized for:
- Quick lookups during code reviews
- Finding specific metrics and tables
- Understanding system boundaries
- Planning technical work

For detailed explanations and context, refer to the comprehensive architecture documentation.

## Maintenance

These reference documents are extracted from the architecture analysis and should be updated when:
- A new architecture analysis is performed
- Dependencies change significantly
- New critical paths are identified
- Confidence assessments are revised

**Last Updated**: 2025-11-25
**Source**: `.claude/codebase/codebase-dedfc2ae-20251125-171241/final/architecture.md`
