# DSH 预设就地化（取消独立 bundle 包）设计

日期：2026-10-12
状态：已获用户批准（2026-10-12 逐节确认）

## 1. 总览

**目标**：把 rev-skills 的 DSH 独立预设从「仓库内第二个 npm 包」改为「根包自带的 bundle 声明」，去掉重复的包壳与随之而来的同步点；预设行为（预设 id、技能根解析顺序、`bin/dsh.mjs` 子命令）完全不变。

**来源**：PR #5（`dnfme:feat/dsh-preset`）。本设计在该 PR 的分支上追加提交，就地更新 PR。

**产出**：新增 1 个文件（`dsh.patch.yml`），删除 `dsh/` 整个目录（4 个文件），其余为常量、断言与文档同步。

## 2. 实测依据

下列结论均在本机 dsh 0.2.0-rc.2 上实测得出，非文档推断：

| 结论 | 证据 |
|---|---|
| bundle = 任意 npm 包 + `dsh.bundle.patch`；profile 的 `dsh.profile.bundles` 填**包名** | 以根包充当 bundle 搭临时 profile，`dsh --profile t3 --dump-config` 退出 0，`preset-rev-skills` 出现在末层，`customSkillDirs` 表达式解析到该包 `.claude/skills`（122 项） |
| patch 文件名任意，路径由 bundle manifest 声明 | 由 `cordis.patch.yml` 改名 `dsh.patch.yml` 后重装并启动，行为不变 |
| **patch 文件必须列进该包的 `files`** | 漏列时 `dsh plugin add` 被拒：`failed to read overlay …/node_modules/rev-skills/cordis.patch.yml: ENOENT`，随后还原 package.json / lock / node_modules |
| `dsh plugin --profile <p> add <pkg>` 会自动追加 `dsh.profile.bundles` 行 | 实测 manifest 由 `["@deepseek-ai/dsh-base"]` 变为 `["@deepseek-ai/dsh-base", "rev-skills"]` |
| bundle 的兼容性校验对无 `@deepseek-ai/*` 依赖的包不构成障碍 | 根包 manifest（含 `bin` / `scripts` / `engines`）直接通过校验并加载 |

`dsh.bundle.patch` 用 `link:` / `file:` 本地规格与从 registry 安装都走同一套解析（`resolveBundleDir` 先查 dsh 安装锚点，再查 profile 目录）。

## 3. 新布局

```
rev-skills/
  dsh.patch.yml          ← 新增（唯一新文件，仓库根）
  package.json           ← 增加 dsh.bundle、files 换条目
  bin/dsh.mjs            ← 两个常量
  docs/dsh-integration.md← 路线二与同步点表重写
（删除）dsh/package.json、dsh/LICENSE、dsh/README.md、dsh/cordis.patch.yml
```

`dsh/README.md` 的面向 DSH 侧说明并入 `docs/dsh-integration.md` 第 2 节；`dsh/LICENSE` 不再需要（补丁文件与 `bin/` 同属工具代码一侧，由仓库根 `LICENSE` 覆盖 Apache-2.0）。

## 4. 根 `package.json`

- 新增 `"dsh": { "bundle": { "patch": "./dsh.patch.yml" } }`
- `files`：删除 `"dsh"`，新增 `"dsh.patch.yml"`（**必需**，见第 2 节第 3 行）
- 版本：沿用 PR 的 `0.1.0`，与 `.claude-plugin/marketplace.json` 成对
- 不变：`bin` 的 `rev-skills-dsh`、`scripts.dsh`、keywords、description

## 5. `dsh.patch.yml`

- 预设行与 18 个子插件配置照旧（补丁内本无包名，`config.id` 仍为 `rev-skills`）
- `customSkillDirs` 表达式：删除第 4 候选（`rev-skills-dsh-preset/package.json` 的 `..` join）；原第 3 候选「包目录」即本包自身，语义不变
- 头部注释按「本包即 bundle」重写；技能根解析顺序说明保留（`REV_SKILLS_DIR` → `<DSH_HOME>/skills` → 本包）

## 6. `bin/dsh.mjs`

- `BUNDLE_DIR`：`join(REPO_ROOT, 'dsh')` → `REPO_ROOT`
- `BUNDLE_PACKAGE`：`'rev-skills-dsh-preset'` → `'rev-skills'`
- `resolveLocalSkillRoot` 的候选说明文案：`从 rev-skills / rev-skills-dsh-preset 的 package.json 反解` → 单包
- 其余逻辑（`check` 与四个 `preset` 子命令、dry-run、备份、原子替换）一行不动

## 7. 测试

- `tests/dsh.test.mjs`、`tests/dsh-preset.test.mjs`：导入常量与断言随包名/目录改名同步；夹具里的假 `node_modules/rev-skills-dsh-preset` 改为 `rev-skills`
- `dsh/package.json 是合法 bundle 清单` 改为根 manifest 断言：`name` 为 `rev-skills`、`dsh.bundle.patch` 指向存在的文件
- **新增回归断言**：`dsh.bundle.patch` 指向的文件必须出现在根 `package.json` 的 `files` 列表中（把第 2 节的坑固化；漏列即测试失败）
- CLAUDE.md 的测试计数（`npm test = validate + 202 项测试`）按实际数同步

## 8. 文档

- `docs/dsh-integration.md`：第 2 节路线二重写（bundle 名为 `rev-skills`；安装命令 `dsh plugin --profile <p> add rev-skills`，本地路径用 `link:` / `file:`；说明 `add` 会自动登记 `bundles` 行）；技能根候选表删除第 4 行；第 6 节同步点表把「版本号 ↔ `dsh/package.json`」两行替换为「`dsh.bundle.patch` ↔ `dsh.patch.yml` 实际路径 + 该路径必须列进 `files`」
- `README.md` / `README_EN.md` / `AGENTS.md` / `CLAUDE.md` 的 DSH 段落同步（涉及 `dsh/` 目录与包名 `rev-skills-dsh-preset` 的表述全部改写）

## 9. 验证清单

1. `node validate.mjs` → `OK: 122 skills validated`
2. `npm test` → 全绿，计数与 CLAUDE.md 一致
3. `node bin/dsh.mjs check` → `OK: 122 skills DSH-compatible (32 warning(s))`
4. 临时 profile：`dsh plugin --profile <p> add <本仓绝对路径>` → manifest 自动出现 `rev-skills` bundle 行 → `dsh --profile <p> --dump-config` 退出 0 且含 `preset-rev-skills`
5. `node bin/dsh.mjs preset spec|status|install --apply|uninstall --apply` 往返（dry-run 先行、apply 后 status 为 ok、uninstall 后复原）
6. `npm pack --dry-run` 输出中包含 `dsh.patch.yml`

## 10. 不做

- 不改 `.claude/skills/` 下任何技能内容
- 不改 PR 中与该目标无关的改动：`.gitattributes`、`lib/frontmatter.mjs` 的 CRLF 容忍与键文法放宽、`bin/install.mjs` 的 `--target dsh`、新增测试文件
- 不删除 `bin/dsh.mjs` 的 `preset` 子命令（用户明确保留，尽管宿主 `dsh plugin add` 已覆盖安装路径）

## 11. 风险

| 风险 | 处置 |
|---|---|
| 根包被当作 bundle 后，兼容性校验要求声明 DSH peer 版本范围 | 实测无 `@deepseek-ai/*` 依赖即通过；若 DSH 后续收紧，退回独立包即可（本设计不改变补丁内容，两形态可互换） |
| `files` 漏列 patch 文件 → 安装期才报错 | 测试固化为回归断言（第 7 节） |
| 使用 `link:` 安装时 profile 直接引用工作区，改补丁立即生效、改后未重启则不生效 | 文档排错表保留「预设改完没生效」一行，处置改为重载/重启会话 |
