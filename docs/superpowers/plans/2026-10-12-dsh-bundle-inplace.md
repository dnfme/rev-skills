# DSH 预设就地化实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 rev-skills 的 DSH 独立预设从「仓库内第二个 npm 包（`dsh/`）」改为「根包自带的 bundle 声明（`dsh.patch.yml`）」，删掉重复包壳与随之而来的同步点，预设行为不变。

**Architecture:** DSH 的 bundle = 任意 npm 包 + manifest 里的 `dsh.bundle.patch`，profile 的 `dsh.profile.bundles` 填**包名**。因此根包 `rev-skills` 自己就能当 bundle：新增一个仓库根的补丁文件，在根 `package.json` 声明它并列入 `files`，删掉 `dsh/` 整个目录，`bin/dsh.mjs` 只改两个常量。

**Tech Stack:** 零依赖 Node.js ESM（>=22）、`node --test`、DSH 0.2.0-rc.2（本机 `/home/DslsDZC/.npm-global/bin/dsh`）。

## Global Constraints

- 分支：`dsh-preset`（跟踪 `dnfme/feat/dsh-preset`，PR #5 就地更新）
- commit message 中文，前缀 `feat:` / `fix:` / `chore:` / `docs:` / `enhance:` / `经验:`
- **每个 commit 只 `git add` 本任务 Files 列出的文件，严禁 `git add -A`**
- 呈现中性（禁用「最推荐」「强烈建议」等最高级强推措辞）；不用 emoji
- 不改 `.claude/skills/` 下任何技能内容；不动 PR 中无关改动（`.gitattributes`、`lib/frontmatter.mjs`、`bin/install.mjs` 的 `--target dsh`）
- 保留 `bin/dsh.mjs` 的四个 `preset` 子命令
- 任一任务结束时 `node validate.mjs` 必须输出 `OK: 122 skills validated`，`npm test` 必须全绿
- `dsh.bundle.patch` 指向的文件**必须**出现在根 `package.json` 的 `files` 里（实测：漏列时 pnpm `file:` 安装被拒 `failed to read overlay … ENOENT` 并整体回滚）
- CLAUDE.md 的测试计数与实际顶层 `test(` 数一致（`tests/counts.test.mjs` 强制），增删测试必须在同一 commit 更新

---

### Task 1: 补丁就地化（文件迁移 + manifest 声明 + 常量 + 守卫测试）

**Files:**
- Create: `dsh.patch.yml`（由 `dsh/cordis.patch.yml` 移动并改写）
- Delete: `dsh/package.json`、`dsh/LICENSE`、`dsh/README.md`
- Modify: `package.json`（新增 `dsh` 字段，`files` 换条目）
- Modify: `bin/dsh.mjs:29-33`（两个常量）、`bin/dsh.mjs:425`（候选说明文案）、`bin/dsh.mjs:~430`（`仓库技能根` 候选的括号说明）
- Modify: `tests/dsh-preset.test.mjs:13-15`（路径常量）、`216-246`（manifest 与 LICENSE 守卫）、`293`（候选断言）、`335-350`（包解析夹具）、测试计数相关
- Modify: `tests/dsh.test.mjs:148`、`397`

**Interfaces:**
- Produces: `bin/dsh.mjs` 导出的 `BUNDLE_DIR` 语义变为「仓库根（即 bundle 包目录）」，`BUNDLE_PACKAGE` 变为 `'rev-skills'`；后续任务与文档全部按这两个值书写
- Consumes: 无

- [ ] **Step 1: 移动补丁文件到仓库根并改写头部注释**

```bash
git mv dsh/cordis.patch.yml dsh.patch.yml
```

把 `dsh.patch.yml` 开头到 `- insert:` 之前的注释块整体替换为：

```yaml
# rev-skills 的 DSH bundle 补丁：本仓库根包（rev-skills）自身即 bundle，根
# package.json 用 `"dsh": { "bundle": { "patch": "./dsh.patch.yml" } }` 指向本文件。
# 顶层一条 `- insert:` 行声明独立 agent preset `rev-skills`，其子条目对齐官方
# `cordis` 预设（去掉 Cordis 编写相关的行与 plan mode）。
# `skill-filesystem` 行设置 `includeDefaultRoots: false`，该 preset 贡献的是 rev-skills
# 技能根这一层，不继承项目与用户技能根；技能目录解析仍会把该层与查看作用域的项目/
# 用户根合并：同名条目以本 preset 层为准，其他 preset 不受影响。
# 技能根部进包内副本：`customSkillDirs` 表达式按顺序探测 `REV_SKILLS_DIR`（指向仓库根
# 或直接指向技能目录）、DSH 用户技能根（`$DSH_HOME/skills`，默认 `~/.dsh/skills`）、
# 本包目录（`rev-skills/package.json` 反解），取第一个存在且至少含一个 `re-*` 条目的候选。
```

- [ ] **Step 2: 从 `customSkillDirs` 表达式删掉最后一个候选**

把该行末尾的候选数组由：

```
const c = [tj(process.env.REV_SKILLS_DIR), process.env.REV_SKILLS_DIR, hd(), tj(pj('rev-skills/package.json')), tj(pt.join(pj('rev-skills-dsh-preset/package.json'), '..'))].filter((d) => d);
```

改为（删掉 `rev-skills-dsh-preset` 那项；原第 3 候选即本包自身）：

```
const c = [tj(process.env.REV_SKILLS_DIR), process.env.REV_SKILLS_DIR, hd(), tj(pj('rev-skills/package.json'))].filter((d) => d);
```

- [ ] **Step 3: 根 `package.json` 声明 bundle 并换 `files` 条目**

`files` 数组里 `"dsh"` 换成 `"dsh.patch.yml"`；在 `"bin"` 块之后新增：

```json
  "dsh": {
    "bundle": {
      "patch": "./dsh.patch.yml"
    }
  },
```

- [ ] **Step 4: 删掉包壳文件**

```bash
git rm dsh/package.json dsh/LICENSE dsh/README.md
```

（`dsh/README.md` 的面向 DSH 侧说明在 Task 3 并入 `docs/dsh-integration.md`，本步骤先记下其内容要点：bundle 安装方式、预设清单、技能根解析顺序。）

- [ ] **Step 5: 改 `bin/dsh.mjs` 的两个常量**

```js
export const BUNDLE_DIR = REPO_ROOT;
export const BUNDLE_PACKAGE = 'rev-skills';
```

- [ ] **Step 6: 改 `bin/dsh.mjs` 的两处说明文案**

`resolveLocalSkillRoot` 里：

```js
  add('仓库技能根 <repo>/.claude/skills（补丁里由包解析得到）', join(repoRoot, '.claude', 'skills'));
  candidates.push({ source: '从 rev-skills 的 package.json 反解（本包即 bundle）', root: null, exists: null, skills: null, resolvable: false });
```

- [ ] **Step 7: 同步 `tests/dsh-preset.test.mjs` 的路径常量**

```js
const DSH_DIR = fileURLToPath(new URL('../', import.meta.url));
const PACKAGE_FILE = join(DSH_DIR, 'package.json');
const PATCH_FILE = join(DSH_DIR, 'dsh.patch.yml');
```

- [ ] **Step 8: 改写 `tests/dsh-preset.test.mjs` 的 bundle 清单守卫**

把 `test('dsh/package.json 是合法 bundle 清单', …)` 整个替换为：

```js
test('根 package.json 是合法 bundle 清单', () => {
  const pkg = JSON.parse(readFileSync(PACKAGE_FILE, 'utf8'));
  assert.equal(pkg.name, 'rev-skills');
  assert.match(pkg.version, /^\d+\.\d+\.\d+/);
  assert.match(pkg.license, /Apache-2\.0/, 'bundle 随根包发布，许可证沿用根包');
  assert.equal(pkg.type, 'module');
  const rel = pkg.dsh?.bundle?.patch;
  assert.equal(typeof rel, 'string', 'dsh.bundle.patch 必须存在');
  const abs = join(DSH_DIR, rel);
  assert.ok(existsSync(abs), `dsh.bundle.patch 指向不存在的文件：${rel}`);
  assert.ok(abs.startsWith(DSH_DIR), 'dsh.bundle.patch 必须落在仓库内');
  assert.ok(Array.isArray(pkg.files) && pkg.files.length > 0);
  for (const f of pkg.files) {
    assert.ok(existsSync(join(DSH_DIR, f)), `files 里列了未随包发布的文件：${f}`);
  }
});
```

（删掉了原断言里对 `dependencies === undefined` 与 `main`/`exports`/`bin` 三项为空的检查：那是「包壳不得指向未发布文件」的守卫，根包本来就有 `bin`。`dependencies` 为空由 Task 2 单独守卫。）

- [ ] **Step 9: 把 LICENSE 守卫改指根 LICENSE**

把 `test('dsh/LICENSE 是仓库根 LICENSE 的 Apache-2.0 正文', …)` 整个替换为：

```js
test('根 LICENSE 是 Apache-2.0 全文，供随包发布的补丁使用', () => {
  const license = readFileSync(join(DSH_DIR, 'LICENSE'), 'utf8');
  assert.ok(license.includes('Apache License'), 'LICENSE 应含 Apache License 标题行');
  assert.ok(license.includes('Version 2.0, January 2004'), 'LICENSE 应是 Apache-2.0 全文');
  assert.ok(license.includes('END OF TERMS AND CONDITIONS'), 'LICENSE 应是全文而非节选');
});
```

- [ ] **Step 10: 改 `tests/dsh-preset.test.mjs` 的技能根表达式断言**

`assert.ok(expr.includes('rev-skills-dsh-preset/package.json'), …)` 一行删除，并把上一行改为：

```js
  assert.ok(expr.includes('rev-skills/package.json'), '表达式应支持从本包（rev-skills 即 bundle）解析');
```

- [ ] **Step 11: 改包解析夹具的假目录**

```js
  const app = mkdtempSync(join(tmpdir(), 'dsh-preset-app-'));
  const fakeRepo = join(app, 'node_modules', 'rev-skills');
  const fakeKeys = join(fakeRepo, '.claude', 'skills', 're-fake');
  mkdirSync(fakeKeys, { recursive: true });
  writeFileSync(join(fakeRepo, 'package.json'), '{}');
  writeFileSync(join(fakeKeys, 'SKILL.md'), 'x');
  try {
    const baseUrl = join(fakeRepo, 'package.json');
    // DSH_HOME 指向一个不存在的目录，确保回落到包解析候选
    const viaPkg = evalExpr(expr, baseUrl, { revSkillsDir: '', dshHome: join(app, 'empty-dsh-home') });
    assert.equal(viaPkg, join(fakeRepo, '.claude', 'skills'));
  } finally {
    rmSync(app, { recursive: true, force: true });
  }
```

（删掉了原来额外创建的 `node_modules/rev-skills-dsh-preset` 目录与它的 `package.json`——bundle 即 `rev-skills` 自己，`baseUrl` 落在同一个包内。）

- [ ] **Step 12: 改 `tests/dsh.test.mjs` 的两处**

第 148 行：

```js
  assert.equal(BUNDLE_PACKAGE, 'rev-skills');
```

第 397 行：

```js
  assert.equal(bundleExists, true, '根包即 bundle（dsh.patch.yml + dsh.bundle.patch），install 用例依赖它');
```

- [ ] **Step 13: 跑测试**

Run: `node validate.mjs && npm test`
Expected: `OK: 122 skills validated`，且 `node --test` 汇总 `fail 0`；若出现 CLAUDE.md 计数不一致的失败（`CLAUDE.md 的测试计数与实际测试数一致`），记下实际数，Task 3 会统一同步——本任务不新增/删除顶层 `test(`，正常应仍是 202。

- [ ] **Step 14: 提交**

```bash
git add dsh.patch.yml package.json bin/dsh.mjs tests/dsh-preset.test.mjs tests/dsh.test.mjs
git commit -m "feat: DSH 预设就地化——根包自带 bundle，取消独立 dsh/ 包"
```

---

### Task 2: 把「patch 不在 files 里」的坑固化成回归断言

**Files:**
- Modify: `tests/dsh-preset.test.mjs`（新增一条测试）
- Modify: `CLAUDE.md`（测试计数）

**Interfaces:**
- Consumes: Task 1 产出的根 `package.json` 的 `dsh.bundle.patch` 与 `files`
- Produces: 无

- [ ] **Step 1: 写这条测试（先写出会通过的版本）**

在 `tests/dsh-preset.test.mjs` 的根 manifest 守卫测试之后追加：

```js
test('dsh.bundle.patch 指向的文件必须随包发布（files 里漏列即安装期失败）', () => {
  const pkg = JSON.parse(readFileSync(PACKAGE_FILE, 'utf8'));
  const rel = pkg.dsh?.bundle?.patch;
  assert.equal(typeof rel, 'string', 'dsh.bundle.patch 必须存在');
  // 归一成包内相对路径：'./dsh.patch.yml' 与 files 里的 'dsh.patch.yml' 视为同一项
  const norm = (p) => p.replace(/^\.\//, '').split('\\').join('/');
  const target = norm(rel);
  const listed = (pkg.files ?? []).map(norm);
  assert.ok(
    listed.includes(target),
    `dsh.bundle.patch 指向 ${rel}，但 package.json 的 files 里没有它；`
      + 'pnpm 的 file: 安装按 files 复制，漏列会让 dsh plugin add 报 '
      + 'failed to read overlay … ENOENT 并整体回滚',
  );
  // bundle 会被整包装进 profile 的 node_modules，不得带上运行时依赖
  assert.equal(pkg.dependencies, undefined, 'bundle 包不得有运行时依赖');
});
```

- [ ] **Step 2: 跑测试确认通过**

Run: `node --test tests/dsh-preset.test.mjs`
Expected: PASS（Task 1 已把 `dsh.patch.yml` 列进 `files`）。

- [ ] **Step 3: 验证这条断言真的会红（刻意的红-绿验证）**

临时把根 `package.json` 的 `files` 里 `"dsh.patch.yml"` 改为 `"dsh.patch.yml.bak"`，运行：

Run: `node --test tests/dsh-preset.test.mjs`
Expected: FAIL，报 `files 里列了未随包发布的文件` 或本测试的 `files 里没有它`。看到失败后**改回** `"dsh.patch.yml"`，重跑确认转绿。

- [ ] **Step 4: 同步 CLAUDE.md 的测试计数**

Run: `node --test tests/*.test.mjs 2>&1 | grep '^ℹ tests'`

把 CLAUDE.md 里 `` `npm test` = validate + N 项测试 `` 的 N 改为实测值（本任务新增 1 条，预期 202 → 203）。

- [ ] **Step 5: 跑全量**

Run: `npm test`
Expected: 全绿，且 `CLAUDE.md 的测试计数与实际测试数一致` 通过。

- [ ] **Step 6: 提交**

```bash
git add tests/dsh-preset.test.mjs CLAUDE.md
git commit -m "test: 固化 dsh.bundle.patch 必须随包发布的回归断言"
```

---

### Task 3: 文档同步

**Files:**
- Modify: `docs/dsh-integration.md`（第 1 节表述、第 2 节路线二、第 4 节验证、第 5 节排错表、第 6 节同步点表）
- Modify: `README.md:66-69`、`README_EN.md:66-69`、`AGENTS.md:14`、`CLAUDE.md`（DSH 段）

**Interfaces:**
- Consumes: Task 1 的 `BUNDLE_PACKAGE = 'rev-skills'`、补丁路径 `dsh.patch.yml`
- Produces: 无

- [ ] **Step 1: 改写 `docs/dsh-integration.md` 第 2 节「路线二」**

原来的三句描述与候选表：bundle 包名改为 `rev-skills`（根包自身即 bundle，`dsh.bundle.patch` → `./dsh.patch.yml`）；技能根候选表**删掉第 4 行**（`<rev-skills-dsh-preset 包目录>/../.claude/skills`），第 3 行改为「`<rev-skills 包目录>/.claude/skills`（本包即 bundle，按包名解析）」。

手工路径的 jsonc 示例改为：

```jsonc
{
  "dependencies": { "rev-skills": "link:/path/to/rev-skills" },
  "dsh": { "profile": { "bundles": [ "@deepseek-ai/dsh-base", "rev-skills" ] } }
}
```

并补一句实测事实：从 registry 或本地路径安装时，`dsh plugin --profile <name> add rev-skills` 成功后会**自动把 `rev-skills` 追加进 `dsh.profile.bundles`**，失败则还原 `package.json`、`pnpm-lock.yaml` 与 `node_modules`。

- [ ] **Step 2: 改写 `docs/dsh-integration.md` 第 6 节「维护同步点」表**

把「版本号」（根 `package.json` ↔ `dsh/package.json`）一行删除（只剩根包一处，由仓库既有的版本同步规则覆盖）；「bundle 名」一行的另一侧由 `dsh/package.json` → `name` 改为根 `package.json` → `name`；「补丁文件」一行改为：

| 项 | 一处 | 另一处 |
|---|---|---|
| 补丁文件 | 根 `package.json` → `dsh.bundle.patch` | `dsh.patch.yml` 实际路径，且必须出现在根 `package.json` 的 `files` 里 |

- [ ] **Step 3: 把 `dsh/README.md` 的要点并入文档**

`dsh/README.md` 里未被 `docs/dsh-integration.md` 覆盖的内容（bundle 的安装输入、预设清单概览）并入第 2 节路线二；重复内容不搬。原文件已在 Task 1 删除，内容以 `git show dnfme/feat/dsh-preset:dsh/README.md` 取回参考。

- [ ] **Step 4: 同步 README / README_EN / AGENTS / CLAUDE**

`README.md` 第 69 行那段改为：

```markdown
也可用 DSH 自带的插件管理器安装：`dsh plugin --profile <name> add rev-skills`（本地路径用 `link:` / `file:` 规格；成功后会自动登记进该 profile 的 `dsh.profile.bundles`）。技能目录可用环境变量 `REV_SKILLS_DIR` 覆盖。细节与排错见 [docs/dsh-integration.md](docs/dsh-integration.md)。
```

`README_EN.md` 第 69 行按同义改写（英文）。`AGENTS.md` 第 14 行把「bundle `dsh/`」改为「bundle 即根包 `rev-skills`（补丁 `dsh.patch.yml`）」。

`CLAUDE.md` 的 DSH 段（第 63-72 行）：
- 「独立预设路线」条目改为：根包自身即 bundle（根 `package.json` 的 `dsh.bundle.patch` → `dsh.patch.yml`；该文件必须在 `files` 里），补丁插入预设行 `preset-rev-skills`（`config.id` = `rev-skills`），启用方式 `dsh plugin --profile <name> add rev-skills` 或 `node bin/dsh.mjs preset install --apply`
- 「必须成对同步的项」那句删掉所有 `dsh/package.json` 相关配对，改为：`dsh.bundle.patch` ↔ `dsh.patch.yml` 实际路径 ↔ 根 `package.json` 的 `files` 条目；补丁里的 `config.id`（`rev-skills`）↔ `docs/dsh-integration.md` 的说明

- [ ] **Step 5: 全库自检残留**

Run: `grep -rn "rev-skills-dsh-preset\|dsh/package.json\|dsh/cordis.patch.yml" --include="*.md" --include="*.json" --include="*.mjs" --include="*.yml" . | grep -v "^./docs/superpowers/"`

Expected: 无输出（`docs/superpowers/specs/2026-09-30-factual-audit-wave.md` 等历史存档允许保留旧表述，已由路径过滤排除；若命中历史存档以外的文件，逐个改正）。

- [ ] **Step 6: 跑测试并提交**

Run: `node validate.mjs && npm test`
Expected: 全绿（若 `tests/counts.test.mjs` 的 README 站点断言因措辞变化而红，按报错补齐缺失关键词）。

```bash
git add docs/dsh-integration.md README.md README_EN.md AGENTS.md CLAUDE.md
git commit -m "docs: DSH 集成文档与清单同步根包自带 bundle 的形态"
```

---

### Task 4: 端到端实机验证

**Files:**
- 不改文件；发现缺陷则回改对应文件并 `fix:` 提交

**Interfaces:**
- Consumes: Task 1-3 的全部产物
- Produces: 供 PR 描述引用的验证结论

- [ ] **Step 1: 静态闸口**

```bash
node validate.mjs
node bin/dsh.mjs check
```
Expected: `OK: 122 skills validated`；`OK: 122 skills DSH-compatible (32 warning(s))`。

- [ ] **Step 2: 补丁里的技能根表达式实测求值**

```bash
node -e '
const fs=require("fs"),pt=require("path");
const y=fs.readFileSync("dsh.patch.yml","utf8");
const expr=y.match(/customSkillDirs:\n\s*- !!js "([\s\S]*?)"\n/)[1].replace(/\\"/g,"\"");
const baseUrl=process.argv[1];
console.log(new Function("baseUrl","return ("+expr+")")(baseUrl));
' /tmp/nowhere/package.json
```
Expected: 打印 `REV_SKILLS_DIR` 未设、`DSH_HOME` 未设时的回落路径（`$HOME/.dsh/skills`）；把 `baseUrl` 换成已安装 profile 里 `node_modules/rev-skills/package.json` 时，应打印该包的 `.claude/skills`。

- [ ] **Step 3: 真实 profile 上装一遍（宿主自带插件管理器）**

```bash
export DSHX=/tmp/dsh-verify && rm -rf $DSHX && mkdir -p $DSHX/profiles/e2e
printf '{"name":"dsh-profile-e2e","private":true,"dependencies":{},"dsh":{"profile":{"bundles":["@deepseek-ai/dsh-base"]}}}\n' > $DSHX/profiles/e2e/package.json
cd $DSHX/profiles/e2e && DSH_HOME=$DSHX dsh plugin --profile e2e add link:/home/DslsDZC/rev-skills
DSH_HOME=$DSHX dsh --profile e2e --dump-config | grep -c preset-rev-skills
```
Expected: 安装成功无告警；`dsh.profile.bundles` 自动变为 `["@deepseek-ai/dsh-base","rev-skills"]`；`grep -c` 输出 `1`（预设行在配置树末层）；stderr 无 `skipping profile bundle`。

- [ ] **Step 4: `bin/dsh.mjs preset` 往返（本仓 CLI 路径）**

```bash
node bin/dsh.mjs preset spec
node bin/dsh.mjs preset status --profile-dir $DSHX/profiles/e2e --json
node bin/dsh.mjs preset install --profile-dir /tmp/dsh-verify/other        # dry-run
node bin/dsh.mjs preset install --profile-dir /tmp/dsh-verify/other --apply
node bin/dsh.mjs preset status --profile-dir /tmp/dsh-verify/other --json  # ok: true
node bin/dsh.mjs preset uninstall --profile-dir /tmp/dsh-verify/other --apply
```
Expected: `spec` 打印 bundle 绝对路径为仓库根、`link:` 规格与 profile 编辑；`status` 对 e2e profile 报 `ok: true`；dry-run 不写文件；`--apply` 生成备份；uninstall 从备份还原。

- [ ] **Step 5: 发布面检查**

```bash
npm pack --dry-run 2>&1 | grep -E "dsh\.patch\.yml|dsh/"
```
Expected: 输出含 `dsh.patch.yml`，不含 `dsh/` 目录条目。

- [ ] **Step 6: 清理临时目录并汇报**

```bash
rm -rf /tmp/dsh-verify
```
把 Step 1-5 的实际输出整理进 PR #5 的描述更新（不新建文件）。若任何一步暴露缺陷，回改对应文件并以 `fix:` 提交后重跑该步。
