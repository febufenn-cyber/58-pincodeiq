# CLAUDE.md

This repo is product #58 of Febin's 50-SaaS challenge. Nothing is built yet — docs/ is
the source of truth. Read docs/LLD.md then docs/PLAN.md and execute tasks in order
with TDD. Stack conventions and the shipped reference implementation live in the
private repo febufenn-cyber/50-saas (contract-reviewer/) — same Worker+Supabase+
credits+provider patterns. The no-LLM-in-hot-path decision and the KV eventually-
consistent quota design in the LLD are verified decisions — do not re-litigate without
evidence.
