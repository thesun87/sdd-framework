# Glossary — Canonical Terms

Bilingual (EN / VI). Every artifact, identifier, and commit message uses the
canonical term. If a term you need is not here, **stop and ask** — do not invent
a synonym. Naming drift is the single failure mode that task-scoped review
cannot detect (setup guide Appendix B #1).

Target: at least 20 canonical terms before the baseline is frozen.
The validator (HV015) fails if this file is missing or under 500 bytes.

| Canonical term (EN) | Tiếng Việt | Definition | Do NOT use |
|---|---|---|---|
| Track A | Track A | Greenfield flow: BMAD → Spec Kit → Superpowers | "full flow", "new project mode" |
| Track B | Track B | Brownfield feature flow: Spec Kit → Superpowers | "feature mode" |
| Track C | Track C | Direct change flow: Superpowers only, for reproducible defects | "hotfix mode", "quick fix" |
| Baseline | Baseline | The frozen contents of `docs/baseline/` at a named `baseline_id` | "the docs", "spec" |
| Baseline freeze | Đóng băng baseline | The act of recording `baseline-freeze.yaml` with `status: frozen` | "lock", "release" |
| Handoff contract | Hợp đồng bàn giao | `.sdd/<feature>/handoff.yaml` — the machine-checked boundary between planning and execution | "handover", "spec bundle" |
| Feature id | Mã feature | `NNN-slug`, the directory name under `specs/` | "ticket", "story id" |
| Change record | Hồ sơ thay đổi | `.sdd/direct/<date>-<ticket>/change-record.yaml` for Track C | "bug report" |
| Task brief | Brief công việc | `.sdd/<feature>/task-<NNN>-brief.md`, the only context a task subagent receives | "prompt", "instruction" |
| Convergence | Hội tụ | The `/speckit-converge` step: reassess the codebase against the spec and append remaining work as tasks | "final check", "QA pass" |
| Escalation | Leo thang | Abandoning a change unit and restarting on a higher track. Never a downgrade. | "upgrade", "switch track" |
| Allowed scope | Phạm vi cho phép | Paths a task may modify | "affected files" |
| Forbidden scope | Phạm vi cấm | Paths a task must never modify, even to improve them | "off limits" |
| Characterization test | Test đặc tả hiện trạng | A test that pins existing (possibly undesired) behaviour before changing it — Track B only | "legacy test" |
| Regression suite | Bộ test hồi quy | The suite named by `verification.commands.regression` | "full test" |
| Impact analysis | Phân tích tác động | `specs/<id>/impact-analysis.md`, produced READ-ONLY before any code change | "investigation" |
| Codebase context | Ngữ cảnh mã nguồn | `.sdd/<id>/codebase-context.md`, ≤ 8 KB, hand-curated | "repo summary" |
| Verification contract | Hợp đồng kiểm chứng | `docs/baseline/verification.md` and its ` ```commands ` block | "test config" |
| Stale handoff | Handoff lỗi thời | A handoff whose recorded git SHA no longer matches the artifact (HV013b) | "outdated" |
| Walking skeleton | Bộ khung chạy được | Feature `000`, the thinnest end-to-end slice that exercises the whole stack | "MVP", "POC" |

<!-- Add project-specific domain terms below this line. -->
