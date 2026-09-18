# Triển khai Agentic SDD lên một repo greenfield mới

Quy trình thủ công, từng bước, chạy copy-paste được. Mọi lệnh trong tài liệu này
đã được chạy thật khi dựng chính repo template này (2026-09-18) — không phải
chép lại từ guide.

**Thời gian thực tế:** Phase 1–5 khoảng 1–2 giờ. Phase 6 (baseline) là 1–5 ngày
làm việc của con người, không phải của agent. Đừng nén nó lại.

Ký hiệu dùng xuyên suốt:

```bash
export TEMPLATE=/home/tuannguyen/projects/ai-learning/sdd-framework   # repo này
export NEW=/đường/dẫn/tới/repo-mới                                    # repo đích
```

---

## PHASE 0 — Quyết định trước khi gõ lệnh nào

Ba câu hỏi này quyết định bạn cài gì. Trả lời trước, đừng vừa cài vừa nghĩ.

**1. Thật sự là greenfield?**
Greenfield = chưa có code, chưa có baseline đóng băng. Nếu đã có codebase chạy
production thì đó là brownfield → **không cài BMAD**, chạy Track C rồi Track B.
Tài liệu này dành cho greenfield.

**2. Ai là người ra quyết định làm rõ yêu cầu, và SLA trả lời bao lâu?**
Phải có tên cụ thể. Track A bị chặn ở `/speckit-clarify` nếu không ai trả lời
được. Đây là thứ làm chết quy trình nhanh nhất.

**3. Ai sở hữu việc nâng cấp từng tool?**
Ghi tên vào `docs/tooling-versions.md`. Một thay đổi format artifact ở bất kỳ
tool nào cũng âm thầm làm vỡ handoff generator và validator.

### Kiểm tra prerequisites

```bash
node -v          # ≥ 20.12
python3 -V       # ≥ 3.10
uv --version
git --version    # ≥ 2.20
python3 -c 'import yaml; print(yaml.__version__)'   # thiếu: pip install pyyaml
```

---

## PHASE 1 — Tạo repo

```bash
mkdir -p "$NEW" && cd "$NEW"
git init -b main
```

Cần commit đầu tiên trước khi cài gì, để mọi thay đổi của installer đều nằm
trong một diff xem lại được:

```bash
cp "$TEMPLATE/docs/agentic-sdd-protocol-v2.md"  docs/ 2>/dev/null || mkdir -p docs && cp "$TEMPLATE/docs/agentic-sdd-protocol-v2.md" docs/
cp "$TEMPLATE/docs/agentic-sdd-setup-guide.md"  docs/
git add -A && git commit -m "chore: import agentic SDD protocol + setup guide"
```

> **Vì sao git là bắt buộc, không phải tuỳ chọn:** `handoff.yaml` ghi git SHA
> của `spec.md`/`plan.md`/`tasks.md`. Luật `HV013b` so lại SHA đó để phát hiện
> ai đó sửa spec giữa lúc đang thi hành. Không có git thì không có gate này —
> và đây chính là luật đáng giá nhất trong validator.

---

## PHASE 2 — Cài ba framework

### 2.1 Spec Kit

```bash
cd "$NEW"
uv tool install specify-cli
export PATH="$HOME/.local/bin:$PATH"          # thêm vào ~/.bashrc cho lâu dài
specify version

specify init --here --force --non-interactive --integration claude --script sh
```

`--non-interactive` là bắt buộc khi chạy trong agent harness — không có nó lệnh
sẽ treo chờ phím mũi tên. `--force` cần vì thư mục không rỗng.

Kiểm tra:

```bash
test -f .specify/memory/constitution.md && echo OK
ls .claude/skills | grep speckit
```

### 2.2 BMAD (chỉ Track A)

```bash
npx --yes bmad-method@latest install --yes \
  --directory "$(pwd)" \
  --modules bmm \
  --tools claude-code \
  --set core.output_folder=docs/baseline
```

`--set core.output_folder=docs/baseline` là quan trọng: mặc định BMAD trộn
artifact vào `docs/`, làm baseline không tách được và luật "ai được ghi vào đâu"
không cưỡng chế được.

Chỉ cài `bmm`. Bỏ `bmb` (agent builder) và `cis` — thêm bề mặt bạn không dùng.

Kiểm tra:

```bash
cat _bmad/_config/manifest.yaml | head -20     # ghi version vào tooling-versions.md
grep output_folder _bmad/config.toml
```

### 2.3 Superpowers

Không tự động hoá được — plugin cài tương tác từ trong Claude Code:

```text
/plugin install superpowers@claude-plugins-official
/reload-plugins
```

Kiểm tra: gõ `/help`, phải thấy các skill `superpowers:*`. Bảy skill cần có:
`using-git-worktrees`, `subagent-driven-development`, `test-driven-development`,
`requesting-code-review`, `verification-before-completion`,
`systematic-debugging`, `finishing-a-development-branch`.

### 2.4 Commit installer output

```bash
git add -A && git commit -m "chore: install spec kit + bmad + superpowers"
```

---

## PHASE 3 — Copy glue layer từ template

**Chỉ copy phần viết tay. Tuyệt đối không copy phần installer sinh ra** —
`.claude/skills/`, `.specify/scripts/`, `.specify/templates/`, `_bmad/` đều chứa
manifest gắn với máy và version; copy sang sẽ lệch với bản vừa cài ở Phase 2.

```bash
cd "$NEW"

# 1. traffic controller
cp "$TEMPLATE/CLAUDE.md" .

# 2. glue scripts
mkdir -p scripts/sdd
cp "$TEMPLATE/scripts/sdd/"*.py scripts/sdd/

# 3. slash commands
mkdir -p .claude/commands
cp "$TEMPLATE/.claude/commands/sdd-"*.md .claude/commands/

# 4. baseline templates
mkdir -p docs/baseline/{prd,architecture,adr} docs/discovery/{interviews,as-is-process,sample-data,external-contracts,compliance}
cp "$TEMPLATE/docs/baseline/README.md"            docs/baseline/
cp "$TEMPLATE/docs/baseline/glossary.md"          docs/baseline/
cp "$TEMPLATE/docs/baseline/feature-map.md"       docs/baseline/
cp "$TEMPLATE/docs/baseline/verification.md"      docs/baseline/
cp "$TEMPLATE/docs/baseline/baseline-freeze.yaml" docs/baseline/
cp "$TEMPLATE/docs/baseline/adr/0000-template.md" docs/baseline/adr/

# 5. .sdd/ và specs/
mkdir -p .sdd/direct specs
cp "$TEMPLATE/.sdd/README.md" .sdd/
touch .sdd/.gitkeep specs/.gitkeep docs/baseline/adr/.gitkeep
for d in docs/discovery/*/; do touch "$d/.gitkeep"; done

# 6. gitignore + bật plugin ở mức team
cat "$TEMPLATE/.gitignore" >> .gitignore
cp "$TEMPLATE/.claude/settings.json" .claude/

# 7. tuỳ chọn: bộ test chứng minh validator chặn thật
cp "$TEMPLATE/package.json" .
mkdir -p tests && cp "$TEMPLATE/tests/"*.mjs tests/
cp "$TEMPLATE/scripts/lint.mjs" scripts/
```

Mục 7 đáng copy kể cả khi dự án không dùng Node. 12 test đó là bằng chứng
validator hoạt động; mất chúng là mất khả năng biết gate còn sống hay đã chết.
Nếu dự án dùng stack khác, giữ `package.json` như một công cụ meta của repo và
đặt lệnh test *của sản phẩm* vào `verification.md` riêng.

```bash
git add -A && git commit -m "chore(sdd): import glue layer from template"
```

---

## PHASE 4 — Sửa cho khớp dự án (đừng bỏ qua bước nào)

### 4.1 `docs/baseline/verification.md` — QUAN TRỌNG NHẤT

Khối ` ```commands ` được parse thẳng vào mọi handoff. `test` và `lint` là bắt
buộc (HV009), `regression` bắt buộc với Track B (BF003).

````markdown
```commands
test: <lệnh test thật của dự án>
lint: <lệnh lint thật>
regression: <suite phải luôn xanh>
build: <lệnh build>
```
````

**Kiểm chứng ngay, đừng tin là nó đúng:**

```bash
python3 -c "
import sys; sys.path.insert(0,'scripts/sdd')
from sdd_handoff import parse_verification
from sdd_lib import BASELINE, read
print(parse_verification(read(BASELINE/'verification.md')))"
```

Phải in ra đúng dict bạn mong đợi. Rồi chạy thật từng lệnh — bài học từ chính
repo này: `npm test` viết ra ban đầu **chưa bao giờ xanh**, vì `node --test tests/`
không nhận thư mục trên Node 24. Một verification contract không chạy được thì
toàn bộ phần còn lại của quy trình là trang trí.

### 4.2 `docs/baseline/glossary.md`

20 thuật ngữ có sẵn là từ vựng **quy trình**. Thêm từ vựng **nghiệp vụ** của dự
án. Validator chặn nếu file dưới 500 byte (HV015), nhưng nó không biết bạn thiếu
từ nào — chỗ này không tự động hoá được.

Thiếu glossary không tốn gì lúc viết, nhưng tạo ra hỗn loạn đặt tên mà review
theo từng task **về mặt cấu trúc không thể phát hiện được**.

### 4.3 `CLAUDE.md` §6 — xác minh lại tên lệnh

Tên lệnh đổi giữa các release. Bản trong template đúng với Spec Kit 1.0.8 /
BMAD 6.12.0 / Superpowers 6.3.0. Xác minh với bản bạn vừa cài:

```bash
python3 -c "
import csv
for r in csv.DictReader(open('_bmad/bmm/module-help.csv')):
    if r['skill'] not in ('_meta',''): print(r['skill'], '|', r.get('output-location',''))"
ls .claude/skills | grep -E 'speckit|bmad'
```

Trong Claude Code: `/help`, xem `speckit-*` là gạch ngang hay dấu chấm. Ghi
dạng đúng vào `CLAUDE.md`. Ghi mọi chỗ lệch vào `docs/tooling-versions.md`.

### 4.4 `docs/tooling-versions.md`

```bash
cp "$TEMPLATE/docs/tooling-versions.md" docs/
```

Rồi thay toàn bộ version bằng số thật của bạn và điền tên người sở hữu việc nâng
cấp. Giữ nguyên bảng "deviations" — nó là thứ cứu bạn khi nâng cấp tool và glue
vỡ.

```bash
git add -A && git commit -m "chore(sdd): adapt glue to this project"
```

---

## PHASE 5 — Chứng minh gate hoạt động, TRƯỚC KHI tin nó

Đây là bước hay bị bỏ nhất và là bước quan trọng nhất. Một validator chưa từng
chặn cái gì thì không ai biết nó có chặn được không.

```bash
npm test        # nếu đã copy bộ test: 12/12 phải xanh
npm run lint
```

Nếu không copy bộ test, làm thủ công một lần:

```bash
mkdir -p specs/999-probe
echo "# Spec" > specs/999-probe/spec.md          # cố tình thiếu plan.md, tasks.md
git add -A && git commit -q -m "probe"
python3 scripts/sdd/sdd_handoff.py --feature 999-probe --track A
python3 scripts/sdd/sdd_validate.py --feature 999-probe
# PHẢI in FAIL HV003/HV004/... và "BLOCKED — DO NOT START SUPERPOWERS", exit 1
git rm -r --cached specs/999-probe .sdd/999-probe && rm -rf specs/999-probe .sdd/999-probe
git commit -q -m "chore: remove validator probe"
```

Thấy `BLOCKED` mới đi tiếp. Nếu nó PASS, glue của bạn hỏng.

---

## PHASE 6 — Baseline (việc của con người, 1–5 ngày)

### 6.1 Thu thập đầu vào thô — không dùng agent

Đổ vào `docs/discovery/`: biên bản phỏng vấn, quy trình as-is, dữ liệu mẫu đã
ẩn danh, hợp đồng tích hợp ngoài, ràng buộc tuân thủ. Agent đọc được thì mới
sinh được thứ dùng được.

### 6.2 Chạy BMAD theo đúng thứ tự, DỪNG đúng chỗ

```text
bmad-product-brief     → docs/baseline/planning-artifacts/
bmad-prd               → docs/baseline/planning-artifacts/
bmad-ux                → nếu có giao diện
bmad-architecture      → docs/baseline/planning-artifacts/
```

Tuỳ chọn khi yêu cầu còn mơ hồ: `bmad-brainstorming`, `bmad-deep-recon`.

**DỪNG Ở ĐÂY.** Không chạy `bmad-create-epics-and-stories` — Spec Kit sở hữu
danh sách task. Không chạy bất kỳ skill Phase 4 nào (`bmad-build`,
`bmad-agent-dev`, `bmad-sprint-planning`, ...) — Superpowers sở hữu thi hành.
`CLAUDE.md` §1 đã cấm sẵn, nhưng bạn vẫn phải biết vì sao.

### 6.3 Curate — con người làm, không phải agent

BMAD ghi vào `docs/baseline/planning-artifacts/`. Protocol cần
`docs/baseline/prd.md`, `architecture.md`. Đây **không phải** thao tác đổi tên:

- Đọc, cắt phần thừa, sửa phần sai.
- Shard PRD vào `docs/baseline/prd/`, architecture vào
  `docs/baseline/architecture/` (`tech-stack.md`, `source-tree.md`,
  `coding-standards.md`, `data-model.md`, `integration-contracts.md`).
- Đánh ID `FR-001`, `NFR-001` cho từng yêu cầu — **luật BV003 truy vết FR trong
  spec về FR trong PRD**; không có ID thì Track A không qua được gate.
- Viết ADR cho các quyết định thực sự ràng buộc implementation.
- Điền `docs/baseline/feature-map.md`: từng lát cắt dọc, ≤15 task mỗi feature,
  sắp theo phụ thuộc, `000-walking-skeleton` đứng đầu.

### 6.4 Đóng băng

```yaml
# docs/baseline/baseline-freeze.yaml
baseline_id: <project>-baseline-0001
status: frozen
frozen_at: 2026-XX-XX
frozen_by: <tên người thật>        # không phải "the team", không phải agent
```

```bash
git checkout -b baseline/0001
git add -A && git commit -m "baseline: freeze <project>-baseline-0001"
```

Từ đây `docs/baseline/**` chỉ được sửa trên nhánh `baseline/*`.

---

## PHASE 7 — Constitution và feature 000

```text
/speckit-constitution
```

Viết các nguyên tắc thực sự ràng buộc code (test-first, biên giới module, luật
xử lý lỗi). Nguyên tắc chung chung không có tác dụng. Chỉ sửa trên nhánh
`governance/*`.

Rồi làm feature `000-walking-skeleton` — lát cắt đầu-cuối mỏng nhất chạm được
toàn bộ stack. Nó tồn tại để chứng minh đường ống chạy, không phải để có tính
năng.

---

## PHASE 8 — Vòng lặp mỗi feature

```bash
# 1. Spec Kit trỏ vào feature
/speckit-specify <mô tả outcome>        # tạo specs/NNN-slug/
export SPECIFY_FEATURE_DIRECTORY=specs/NNN-slug   # cần khi dùng worktree

# 2. đặc tả — chỉ dán phần PRD đã shard liên quan, KHÔNG dán cả PRD
/speckit-clarify
/speckit-plan
/speckit-checklist
/speckit-tasks

# 3. commit để có git SHA, RỒI mới bàn giao
git add specs/NNN-slug && git commit -m "spec: NNN-slug"

# 4. gate
/sdd-handoff NNN-slug A
/sdd-validate NNN-slug          # PASS mới được đi tiếp

# 5. thi hành — Superpowers
#    dùng: using-git-worktrees, subagent-driven-development,
#          test-driven-development, requesting-code-review,
#          verification-before-completion
#    BỎ:   brainstorming, writing-plans, executing-plans

# 6. hội tụ
/speckit-converge
```

**Thứ tự ở bước 3–4 không đảo được.** `HV013` chặn nếu artifact chưa commit —
không có SHA thì không phát hiện được handoff lỗi thời.

---

## Những cái bẫy đã biết

| Triệu chứng | Nguyên nhân thật |
|---|---|
| `HV013` fail dù file có tồn tại | artifact chưa commit. Handoff ghi SHA, không ghi nội dung. |
| `HV013b` fail giữa lúc đang chạy | ai đó sửa spec sau khi sinh handoff. **Đúng như thiết kế** — quay lại phase sở hữu artifact, đừng sửa validator. |
| `HV014` fail | working tree bẩn. Commit hoặc stash. |
| Lệnh `speckit-*`/`sdd_*` báo không tìm thấy feature trong worktree | `.specify/feature.json` là machine-local và bị gitignore, không theo sang worktree. `export SPECIFY_FEATURE_DIRECTORY=specs/NNN-slug`. |
| Agent tự nhảy vào brainstorming | `superpowers:brainstorming` có description "You MUST use this before any creative work". `CLAUDE.md` §2 override nó. Nếu vẫn xảy ra thì chặn ngay, đó là lỗi. |
| BMAD đòi chạy tiếp sang epics/stories | Nó không biết Spec Kit tồn tại. Dừng nó ở `bmad-architecture`. |
| `BV003` fail: "spec FRs with no PRD origin" | Feature đang thêm yêu cầu không có trong PRD. Hoặc sửa spec, hoặc sửa PRD trên nhánh `baseline/*`. Đừng tắt luật. |
| Validator PASS nhưng code sai lung tung | Kiểm tra `verification.md` có lệnh chạy được thật không. HV009b chỉ WARN nếu binary không có trên PATH — nó không chặn. |

---

## Danh sách kiểm cuối

```text
[ ] node/python/uv/git đạt phiên bản tối thiểu
[ ] git repo khởi tạo, commit đầu tiên tồn tại
[ ] specify init chạy xong, .specify/memory/constitution.md tồn tại
[ ] BMAD cài với core.output_folder=docs/baseline (chỉ repo Track A)
[ ] Superpowers cài qua /plugin, /help thấy các skill
[ ] glue layer copy sang, KHÔNG copy .claude/skills .specify/scripts _bmad
[ ] verification.md có lệnh THẬT, đã chạy từng lệnh, tất cả xanh
[ ] parse_verification() in ra đúng dict mong đợi
[ ] glossary có từ vựng nghiệp vụ, không chỉ từ vựng quy trình
[ ] CLAUDE.md §6 khớp với /help và module-help.csv của bản vừa cài
[ ] tooling-versions.md không còn placeholder, có tên người sở hữu nâng cấp
[ ] validator đã được chạy với handoff cố tình hỏng — NÓ ĐÃ CHẶN
[ ] người ra quyết định làm rõ yêu cầu có tên và có SLA
[ ] CI guards (setup guide §7) — hoặc đã cài, hoặc đã ghi nhận là lỗ hổng
```

Dòng áp chót quan trọng nhất trước khi giao cho người khác dùng. Dòng cuối là
thứ quyết định mọi luật trong `CLAUDE.md` §3 là quy tắc hay chỉ là lời khuyên.
