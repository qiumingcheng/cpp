# AGENTS.md — 工作区协作说明

本文件供所有 AI 编码代理（Claude Code、Codex 等）自动加载。`CLAUDE.md` 是本文件的软链，两处内容一致，只维护本文件。

**发布硬约束（2026-11-09 最新用户要求）：禁止覆盖远端已有提交或重写已发布历史。禁止 `--force`、`--force-with-lease`、带 `+` 的 refspec 等非快进推送，也不得通过删分支重建或强制更新 API 绕过。后续发布使用新增提交和普通快进 push；远端已有新提交时先 fetch、保留并整合远端改动，未解决冲突时停止发布，不用旧版本替换远端。此前 amend 后覆盖发布的约定已撤销，只保留为历史记录。**

**跨机器接续入口更新：2026-11-09。** 先读本节，再按任务读取相关专题。§2～§7 保留了原 WSL 环境的操作方法与带日期实验记录；其中的历史 HEAD、安装路径、PID、端口占用、“未 push”及测试数字不能直接当作另一台机器或当前远端版本的状态。当前发布版本以 §0.2 为准，后续有新发布时同步更新该表与发布回执。

## 0. 快速导航

| 当前任务 | 先读章节 | 核心入口 |
|---|---|---|
| 查找图多跳、向标融合、向文融合的性能报告 | §0.8 | 三份性能主文档、远端直达链接和新环境路径 |
| 新机器准备工作区、确认仓库和版本 | §0.1～§0.4 | 保持 `cpp/各项目` 布局；按八分支映射选择源码 |
| 继续距离复用、列裁剪、UNION ALL Top-K 下推 | §0.2、§0.4 | `公共文档/UNION_ALL与DiskANN_TopK/` |
| 继续向量＋标量过滤融合 | §0.2、§0.4 | `公共文档/openGauss类Oracle向标融合/执行包/` |
| 编写或整理文档 | §1 | 默认落盘到 `公共文档/<主题>/`，不要散落到源码树 |
| 操作 Oracle 26ai | §2 | `deliverables/ora.sh` |
| 继续 PG19 + ORCA + Daft | §3 | `公共文档/多模优化与计算/执行包/` |
| 操作 PG19 + Apache AGE | §4 | 端口 55439；先 `LOAD 'age'` |
| 编译、安装或启停 openGauss | §5 | 按 Debug/Release 用途选择脚本 |
| 继续 HybridRecall 优化或对标 Milvus | §6 | 先看 §6.1 当前代码，再按 §6.5～§6.7 测试 |
| 继续 PG19 原生多跳图执行（图 B-tree / graph_adj / AGE 邻接） | §7 | `.worktrees/pg19-graph-core/`；当前安装 `pgq-integrated`，不要用历史实验目录 |

除用户明确要求外，不要跨项目启动服务、修改分支或搬迁文档。执行前先确认当前工作树、dirty 文件、运行进程和容器状态；结束后恢复不再需要的服务，保留用户数据与无关修改。

### 0.1 工作区结构与路径

用户要求各环境继续采用 **`cpp` 下放各项目** 的布局，并尽量精简外层仓库。`qiumingcheng/cpp` 首批仅收录本文件、`CLAUDE.md` 软链、`.gitignore` 和[跨机器研发工作区重建指南](公共文档/工作区维护/Codex云端迁移/跨机器研发工作区重建指南.md)。产品源码、已发布文档和测试从各自仓库恢复，不把产品目录及其 `.git` 当普通文件整体提交到外层仓库。

```text
cpp/
├── AGENTS.md
├── CLAUDE.md -> AGENTS.md
├── 公共文档/
├── integration/                  # 原机器可保留，精简新环境不必创建
├── deliverables/                 # 原机器可保留，精简包不上传
├── openGauss-server/
├── openGauss-vector-filter-p12/
├── openGauss-hybrid-recall-consolidate/
├── postgres19/
├── pg_orca/
├── daft/
├── apache-age-pg19-port/          # 原 AGE 属性统计环境按需保留
└── .worktrees/pg19-graph-core/
    ├── postgres19/
    └── apache-age-pg19-port/      # 原生图配套 AGE，与上面的旧环境隔离
```

- 本文件所在的 `cpp/` 为工作区根。其他机器可以使用不同用户名和磁盘位置，保持根下相对布局即可；不要为适配文档强行创建 `/home/qiumc`。
- 在需要执行下文命令的新 shell 中，先进入本文件所在目录，再设置：

  ```bash
  export CPP_WORKSPACE="$(pwd -P)"
  ```

- 文档默认写到 `$CPP_WORKSPACE/公共文档/`。构建脚本继续使用它们已有的 `PGD_WORKSPACE`、`CODE_BASE`、`BINARYLIBS`、`GAUSSHOME`、`PGDATA`、`PGPORT` 等配置入口，不能只改文档路径便假定脚本已经可移植。
- 新机器的源码目录可以通过 clone 或独立 worktree 准备；已有目录先检查所属仓库、分支及 dirty 状态，不自动覆盖。使用 Git worktree 时由 Git 在该环境创建，不复制另一台机器的 `.git` 链接文件。
- `.build/`、`.install/`、`.runtime/`、容器和数据库数据目录是各机器独立的环境资产。文中的默认端口、库名和容器名只是定位线索，不表示新机器已经存在或正在运行这些服务。

### 0.2 五个产品仓库、八个特性分支与接续版本

下表是 **2026-11-09 更新后的发布快照**（八分支迁移于 2026-11-08 发布，原生图 PG 与 ORCA 测试修复于 2026-11-09 发布），用于新机器恢复；不是要求当前 WSL 工作树立即切换或 reset。先确认目标特性，不能用仓库默认分支替代对应特性分支，也不能把同仓的不同特性分支直接混合。

| 特性／组件 | 个人仓库 | 分支 | 完整提交 SHA | 工作区内目录 |
|---|---|---|---|---|
| 距离复用＋Top-K 下推＋列裁剪 | [openGauss-server](https://github.com/qiumingcheng/openGauss-server) | `codex/migrate-ann-distance-topk-column-pruning` | `533c4bde563f6a221f176bd41d2bffc78223c2a3` | `openGauss-server/` |
| 向量＋标量过滤融合 | [openGauss-server](https://github.com/qiumingcheng/openGauss-server) | `codex/migrate-vector-scalar-filter` | `22d4918f5e042a992d4c67a7de7957a297f27bb5` | `openGauss-vector-filter-p12/` |
| 文本＋向量混合召回 | [openGauss-server](https://github.com/qiumingcheng/openGauss-server) | `codex/migrate-hybrid-recall` | `bdda19c2a613c3584ac40099f1e1ae9921d9830f` | `openGauss-hybrid-recall-consolidate/` |
| 原生多跳图／PG 内核 | [postgres](https://github.com/qiumingcheng/postgres) | `codex/migrate-pg19-native-graph` | `54b31bef68d6874ae10403c6ef5ff966c93cbe4d` | `.worktrees/pg19-graph-core/postgres19/` |
| 原生多跳图／AGE 适配 | [age](https://github.com/qiumingcheng/age) | `codex/migrate-pg19-native-graph` | `dc9f36cb93c3687fceef140707cd72303b62934b` | `.worktrees/pg19-graph-core/apache-age-pg19-port/` |
| PG19＋ORCA＋Daft／PG 基线 | [postgres](https://github.com/qiumingcheng/postgres) | `codex/migrate-pg19-orca-daft` | `7873db5369b967dd53984df34fd800003410d04f` | `postgres19/` |
| PG19＋ORCA＋Daft／优化器与 PG 执行适配 | [pgorca](https://github.com/qiumingcheng/pgorca) | `codex/migrate-pg19-orca-daft` | `3e19ab2a5729f6418c8fa382072664fd632c601e` | `pg_orca/` |
| PG19＋ORCA＋Daft／Rust Bridge 与执行引擎 | [Daft](https://github.com/qiumingcheng/Daft) | `codex/migrate-pg19-orca-daft` | `285faf468114b0d7b839c1bcf611c7e606b3dcff` | `daft/` |

- 注意远端与本地目录名不同：`postgres` → `postgres19`，`pgorca` → `pg_orca`，`Daft` → `daft`，`age` → `apache-age-pg19-port`。同一特性涉及不同仓库时分支名相同；这不表示各仓 SHA 或基线相同。
- 首组三特性的共同基线为 `7b4cd7e77b41e8254704e6d8c02bf08be9412b67`，基线上保留三个特性提交。向标融合基线为 `ba1e4ad2f99295b1b0f65e3880a05cb75a1121d4`；其他基线与提交父链见下面的发布回执。PG 协同计算分支仅保留 PG19beta2 契约基线，没有新增 PG 内核提交。
- 2026-11-08 的八项审查修复在云端完成，四个受影响分支当时 amend 后各仍是一条迁移提交。该次迁移仅静态复核、未编译或运行测试；不能用历史通过记录替代实际版本验收。
- 2026-11-09 在新 openEuler 22.03 WSL 环境，原生图 PG 新增后续提交 `4e99b55`（采集器私有快照命令号）和 `54b31be`（并发 expected 末尾空行），正常 fast-forward push，未重写迁移历史。图模块 31/31、PG 核心回归 245/245、graph-adj-info 两种交错 1/1、配套 AGE 43/43 与 AGE 事务私有邻接补测 11/11 通过。距离复用＋Top-K＋列裁剪方向在同机 Debug/O0 构建安装成功，原生8/8通过；向标融合方向也已完成Debug/O0构建安装、原生8/8、普通取消和模式/缓存补测40/40；HybridRecall同机Debug/O0构建安装成功、仓内hybrid_recall回归1/1通过。五个方向均完成本轮可取得提交用例的验证，具体范围及限制见下文；不据此宣称生产准入完成。本轮实际日志与问题记录位于本机 `公共文档/工作区维护/特性构建与测试/`（未作为公开仓库上传材料）。
- 2026-11-09 ORCA 后续提交 `0d9d271` 已正常 push，补齐 `daft_loader` 遗漏的仓内 ABI 夹具和 preload 检查，取消对未发布旧任务脚本与固定 `/tmp` 目录的依赖。ON 构建安装、loader 1/1、IR 9项、成本41项及最终计划校验13项通过；OFF 独立构建及无 loader/Bridge/Arrow 动态依赖与 dpg 符号检查也已通过；随后 3e19ab2 修复新增 F01 用例未恢复 trace 导致测试成本配置被拒绝及 expected 空行；Release Bridge 构建安装、Daft SQL 16/16、Rust 155/155＋故障注入5/5、PG核心245/245、正确性14/14、Join压力/恢复及打包升级回滚矩阵均通过。完整ORCA schedule首轮33/34通过，唯一失败为修改前运行的Arrow expected空行；修正后定向1/1及完整Daft子集16/16通过，34个用例均有通过结果，首轮非零退出码保留。72小时门因缺少run_mixed_workload.sh未运行，未发布的历史1439项向标runner和Oracle原始导出也未虚构复测。所有本轮实例已停止、数据库保留。
- 原生图最新修复的行为边界：跨角色 `REASSIGN OWNED` 会使角色绑定的派生图统计失效，新角色需要重新创建并收集；同角色迁移不变，不删除业务图或表数据。
- 原迁移源 WSL 中的产品源码当时未同步上述云端修复，且可能存在用户修改；不得从该旧 HEAD 重建提交覆盖远端，也不得擅自 reset、clean、切分支。2026-11-09 的 openEuler WSL 是按指南重建的独立新环境，实际恢复和本轮测试修复见上文，不能将旧源机状态套用到当前目录。
- 核实与更新本表时，结合用户当前指令与实际远端分支；恢复方法见[重建指南](公共文档/工作区维护/Codex云端迁移/跨机器研发工作区重建指南.md)。原机器的 `公共文档/工作区维护/Codex云端迁移/` 仍保留迁移状态、审查报告和 amend 回执，精简包不附带这些审计附件。远端已变化时先识别新旧差异，不以本表快照强行回退。

### 0.3 新机器首次接手顺序

1. 确认工作区根和用户要继续的特性，按 §0.2 定位仓库、分支和目录；读取目标项目自己的 `AGENTS.md` 等约定。
2. 只读检查已有工作树、HEAD、dirty 文件、进程、端口和容器。已有安装和数据库先确认构建身份、版本及数据用途；不因目录存在就认定它们与选定源码匹配。
3. 按 §0.4 读取专题交接、任务状态和未闭合事项。新机器优先阅读各产品仓内的已发布文档。完整任务包存在时，`task_state.json` 只能通过对应 `taskctl.py` 管理；缺少历史状态或依赖时不运行不完整的任务工具、不生成虚假的完成记录，可按重建指南使用仓内原生构建与测试继续源码研发。
4. 配置本机工具链、依赖来源、路径和连接参数。数据用版本化生成脚本恢复，或按来源与校验值获取归档。图种子库、失败现场和用户业务数据不是缓存，不能清理或原地重建。
5. 按本次授权执行必要工作。文中的 build、start、gate 等命令是任务入口，不是读取本文件后必须立即执行的操作；沿用用户当前对编译、测试、服务及发布的限制。只做文档、盘点或迁移时，不自动拉起服务或运行长时间稳定性测试。
6. 交接时记录实际源码 SHA、改动范围、执行过的命令、测试结果及未验证项；停止本次不再需要的服务，保留用户数据与无关修改。

**当前入口可用性**：本仓库的最小资料集为本文件、`.gitignore` 和重建指南，`CLAUDE.md` 为软链；提交和 push 在云环境完成。为减少文件，版本清单直接使用 §0.2，恢复命令内嵌在指南中，不再另建 README、lock 文件或独立 bootstrap 脚本。命令尚未在新机器实测，不能宣称一键恢复已经验证。

### 0.4 特性文档、任务状态与执行资产

下表保留原完整工作区的资料定位，并非首批上传目录清单。精简环境优先在对应产品仓的 `公共文档/`、`docs/` 与测试目录中读取已发布内容；未发布的历史任务附件不由文档虚构重建。

| 特性 | 先读入口 | 接续时必须关注 |
|---|---|---|
| 距离复用＋列裁剪＋Top-K 下推 | `公共文档/UNION_ALL与DiskANN_TopK/设计/`、`源码解读/`、`测试/`、`迁移/` | 某些 README 仍描述旧实验版本；先对照 §0.2 与迁移记录。已在最终分支中的补丁不能重复应用 |
| 向量＋标量过滤融合 | `公共文档/openGauss类Oracle向标融合/执行包/MODEL_HANDOFF_PROMPT.md`、`继续开发交接.md`、`已知未闭合事项.md` | 同包 `scripts/`、`specs/`、`sql/`、`state/`、`templates/` 与必要证据属于研发资产，不能只迁 Markdown |
| HybridRecall | `公共文档/两路召回对比Milvus/交接文档_继续优化.md`、`HybridRecall与Milvus测试数据、有效优化及结果总结.md` | 历史性能报告绑定当时的源码、构建、数据与参数；新源码不能沿用旧性能结论。基准入口见 `deliverables/hybrid_vs_milvus_bench/` |
| 原生图＋AGE | `公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/MODEL_HANDOFF_PROMPT.md`、`继续开发交接提示词.md` | 活动任务仍为 `G1-080`；保留 `scripts/checks/specs/include/decisions/state` 及校验器引用的 `golden/evidence`。不得与 ORCA/Daft 的 PG 实例混用 |
| PG19＋ORCA＋Daft | `公共文档/多模优化与计算/执行包/MODEL_HANDOFF_PROMPT.md`、`继续开发交接提示词.md`、从零构建指南 | 活动任务仍为 `P2-001-D`；72 小时稳定性门未完成。根任务 YAML/JSON、执行包与 `integration/pg19-pgorca-daft/` 共同组成任务入口 |

当前精简发布不单独上传这些专题目录和执行包。源码仓内已有的 SQL、fixtures、预期结果与测试随源码取得；原机器的历史资料继续保留。如果以后需要恢复原任务控制流程，必须同时取得它引用的真实规格、状态与必要证据，不能一边省略 `golden/evidence` 一边声称校验器或旧任务包可完整运行。大体积数据另行归档，不能据此删除原资产。

### 0.5 发布和公开资料约定

- 默认只向用户指定的个人仓库和特性分支提交、push。除用户明确要求创建 PR 外，不调用创建 PR 工具、执行 `gh pr create` 或点击网页的 PR 创建流程，也不向 fork 的官方上游默认发起 PR。`Compare & pull request` 是可选提示，不是已存在 PR。
- 清理 PR 仅限授权范围，关闭时保留分支和代码。`cpp` 辅助资料与产品源码分别发布，不能把一个仓库的授权当作任意其他仓库的写入授权。
- 当前用户约定（2026-11-09 本机测试任务更新）：本环境允许对测试发现的必要问题直接修复、commit 并 push 到个人仓库对应特性分支，记录问题和验证；不创建 PR，不默认 amend 或 force push。此前“仅云端提交、WSL 只读”的约定属于 2026-11-08 迁移的历史边界，不覆盖本次明确授权。`qiumingcheng/cpp` 仍按上述四个条目发布辅助资料，不因此整体上传本机测试日志、产品树或数据。
- 历史迁移修复曾按当时要求 amend 并覆盖发布。用户现已禁止覆盖：后续不得 amend、rebase 或 reset 已发布提交后重写远端历史；采用新增提交和普通 push，保留远端已有提交。若普通 push 因远端推进而被拒绝，先 fetch 并整合，不能改用任何 force 方式。
- `qiumingcheng/cpp` 当前为公开仓库。文档和示例不写真实密码、token、cookie 或私钥；认证由本机私有配置提供。发布脚本时也要处理其机器专属认证和路径，不因本文件已整理就假定整个工作区可以原样上传。
- 外层仓库按明确清单收录文件，不整体上传产品源码目录、`.git`、编译/安装产物、运行数据库、工具链缓存、日志、运输分片和重复源码快照。旧归档若有独立恢复价值，保留在适当存储及索引中，不作为缓存删除。

### 0.6 本文件的维护方式

新版本发布后先更新 §0.2 和对应回执；新机器只更新其实际环境配置，不篡改其他机器的历史测量。历史章节中的 SHA、性能数字和二进制 hash 必须保留原归属，不能简单全局替换成新 SHA。遇到说明与实际工作树不一致时，先核实并明确记录差异；本文件不是覆盖用户最新指令或删除现有资产的理由。

### 0.7 向标融合完整功能回归入口（2026-11-09 补包）

向标融合分支新增仓内 [P1/P2 完整功能回归包](openGauss-vector-filter-p12/src/test/regress/diskann_filter_p12/README.md)。收到“编译、安装并测试向标融合”的任务时，先检查源码目录与用户修改；新环境缺少源码时按 §0.2 和重建指南获取该特性版本，再按此 README 准备构建依赖、Release 安装、独立实例并运行测试。只准备本任务所需项目，已有工作树版本有差异时先识别原因，不擅自覆盖。

该包包含 15 组验证及辅助脚本、2 份 Oracle 原始数据、4 份固定预期和校验清单，可离线重放历史 1439 项核心检查；原生 8 项及取消复用单列。普通 fixture 由脚本创建，Oracle 固定输入和预期不得由当前产品输出替换。新环境无需为重放启动 Oracle；任务控制状态、性能和长稳测试仍按原边界。

本次补包只做静态核对，未执行完整 1439 项。实际通过数量、失败和续跑范围以目标环境新输出为准，不能沿用历史结果。该测试包随产品仓获取，外层 cpp 不复制脚本或数据。该次补包与索引更新曾按当时约定在云端 amend/push，属于历史操作；最新禁止覆盖的约束优先，后续改用新增提交和普通 push。其他环境的既有任务授权与历史记录保留原归属。

### 0.8 三项性能报告快速定位（2026-11-09 核对）

查询“最后性能报告”“测试场景简表”或准备性能汇报时，优先读取下表。原完整工作区的三份正文位于各自 `公共文档/` 主题路径；精简新环境从对应产品仓取得，按下列工作区相对路径定位。远端链接固定到已核对的提交，三份正文均与原工作区文件内容一致；不要求在 cpp 外层再复制报告。

| 特性 | 性能主文档（远端直达） | 新环境中相对 cpp 根的路径 |
|---|---|---|
| 向标融合 | [DiskANN前两阶段最终测试报告.md](https://github.com/qiumingcheng/openGauss-server/blob/22d4918f5e042a992d4c67a7de7957a297f27bb5/%E5%85%AC%E5%85%B1%E6%96%87%E6%A1%A3/openGauss%E7%B1%BBOracle%E5%90%91%E6%A0%87%E8%9E%8D%E5%90%88/DiskANN%E5%89%8D%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%9C%80%E7%BB%88%E6%B5%8B%E8%AF%95%E6%8A%A5%E5%91%8A.md) | `openGauss-vector-filter-p12/公共文档/openGauss类Oracle向标融合/DiskANN前两阶段最终测试报告.md` |
| 图多跳 | [主要测试场景简表.md](https://github.com/qiumingcheng/postgres/blob/54b31bef68d6874ae10403c6ef5ff966c93cbe4d/%E5%85%AC%E5%85%B1%E6%96%87%E6%A1%A3/PostgreSQL19%E5%8E%9F%E7%94%9F%E5%A4%9A%E8%B7%B3%E5%9B%BE%E6%89%A7%E8%A1%8C%E4%B8%8EAGE%E9%80%82%E9%85%8D/%E6%8A%A5%E5%91%8A/%E4%B8%BB%E8%A6%81%E6%B5%8B%E8%AF%95%E5%9C%BA%E6%99%AF%E7%AE%80%E8%A1%A8.md) | `.worktrees/pg19-graph-core/postgres19/公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/主要测试场景简表.md` |
| 向文融合（HybridRecall） | [HybridRecall与Milvus测试数据、有效优化及结果总结.md](https://github.com/qiumingcheng/openGauss-server/blob/bdda19c2a613c3584ac40099f1e1ae9921d9830f/%E5%85%AC%E5%85%B1%E6%96%87%E6%A1%A3/%E4%B8%A4%E8%B7%AF%E5%8F%AC%E5%9B%9E%E5%AF%B9%E6%AF%94Milvus/HybridRecall%E4%B8%8EMilvus%E6%B5%8B%E8%AF%95%E6%95%B0%E6%8D%AE%E3%80%81%E6%9C%89%E6%95%88%E4%BC%98%E5%8C%96%E5%8F%8A%E7%BB%93%E6%9E%9C%E6%80%BB%E7%BB%93.md) | `openGauss-hybrid-recall-consolidate/公共文档/两路召回对比Milvus/HybridRecall与Milvus测试数据、有效优化及结果总结.md` |

- 向标融合看《DiskANN前两阶段最终测试报告》§5.1 的 OFF/自然 AUTO 收益与劣化主对照；测量属于 2026-10-11～12 的具体版本，不能给新提交直接套用旧性能。
- 图多跳看《主要测试场景简表》§2.6 的 2026-10-25 复测汇总（强制路线对照，非 AUTO 选路）；完整研究记录仍按其引用定位。保留连续键等计时门未通过的限制，不称全部性能验收完成。
- 向文融合看《HybridRecall与Milvus测试数据、有效优化及结果总结》§4.5.1～§4.5.3 的最近正式对比（2026-10-13）；后续诊断更新到 2026-10-19，诊断观察不替代正式同口径性能结果。

本导航更新不代表重新运行性能测试。报告正文、历史版本及原始数字保持不变，未发布的附件不能因为正文已发布就视为齐备。

## 1. 文档类交付物的落盘与目录分层（强制）

### 1.1 路径优先级

1. 用户明确指定了文件路径时，严格使用该路径；父目录不存在就直接创建，不得因为当前工作目录、工具默认目录或操作方便而改存别处。
2. 用户要求修改已有文档时，保持原文件路径，不另建一份同名或近似副本。
3. 源码项目自身约定位置的文档（如项目内 `README.md`、`docs/`、API 文档）遵循该项目约定。
4. 除上述情况外，凡“编写、生成、整理、输出、补充”方案、设计、调研、报告、指南、汇报材料等面向用户的文档类交付物，默认根目录一律为：

   ```text
   <本文件所在的 cpp 工作区>/公共文档
   ```

   本机当前对应 `/home/qiumc/data/code/cpp/公共文档`；其他机器以实际 `CPP_WORKSPACE` 为准。不得默认落到当前目录、工作区根目录、某个源码仓、`deliverables/`、`.codex_tmp/` 或 `/tmp`；也不得只在回复中给出一个并未实际创建的“建议路径”。

### 1.2 写入前必须先判断归属

- 先查看 `公共文档/` 已有目录和同主题文件，并用标题、项目名、技术名等关键词检索；有合适的主题目录就复用，有应当更新的权威文档就更新原文，避免重复造文档。
- 不要把 `公共文档/` 当成所有文件平铺的目录。文档明显属于某个项目、技术或专题时，必须放入 `公共文档/<项目或主题>/`；目录不存在时主动创建，无需等用户逐字要求创建子目录。
- 满足以下任一情况时，必须创建或复用主题子目录：同一主题预计有两份及以上文档；包含设计、测试、实施、报告、交接等多个阶段；带有图片、数据、脚本或其他附件；从名称即可识别出独立项目或专题。
- 复杂主题可继续按用途分层，例如 `公共文档/<主题>/设计/`、`测试/`、`报告/`、`执行包/`、`图示/`；优先沿用该主题已有结构，不机械制造只有一个文件的无意义深层目录。
- `公共文档/` 根目录只放跨主题索引、工作区级规范，以及确实无法归入任何主题的一次性独立文档。不要用超长文件名代替应有的目录层级。
- 图片、图表和附件应与正文放在同一主题树中（如 `<主题>/图示/` 或 `<主题>/assets/`），正文使用相对链接，移动或新建后检查链接是否仍有效。

### 1.3 写入与交付检查

- 生成文档时应先确定最终目录，再一次性写入目标位置；不要先在别处生成副本再遗留两份。临时中间文件可以放 `/tmp`，但最终交付物必须归档到上述最终路径。
- 文件名应准确表达主题和用途，避免新增 `新建文档.md`、`report.md`、`最终版2.md` 等含糊名称；除非文档本身是时间快照，否则不要随意把日期或“最终版”写入文件名。
- 完成前必须核验目标文件真实存在、路径正确、层级合理，并检查本次新增文档是否误落在其他目录。最终回复中给出实际落盘的绝对路径；若因权限或工具限制无法写入目标路径，应明确报告失败原因，不得静默改存其他位置或声称已经保存。
- 上述规则约束后续新增或本次涉及的文档，不要求代理擅自批量搬迁历史文件；需要重组已有目录时，必须先确认引用关系，并仅在用户要求的范围内进行。

## 2. Oracle 26ai（podman 容器，实测 2026-09-05）

### 2.1 推荐入口

**推荐入口：`deliverables/ora.sh` 包装脚本**（实测），容器自愈 + 宽表智能排版 + 中文显示，免去下面所有前置坑：

```bash
O="$CPP_WORKSPACE/deliverables/ora.sh"
$O 'select ...'            # 智能表格（默认）：CSV 取数 + 按终端宽度自适应列宽/截断
$O -v 'select ...'         # 竖排：宽表神器（91 列的 ALL_TABLES 也能一行一条可读）
$O -c 'select ...'         # 原始 CSV（脚本/管道处理）
$O --raw 'select ...'      # 原生 sqlplus 输出
$O --sys "select ... from v\$instance;"   # SYSDBA（可叠加 -v/-c）
$O                         # gsql 风格交互（持久会话 + 宽表左右滚动 + ↑↓ 历史）
$O --native                # 原生 SQL*Plus 交互（兼容 ACCEPT/EDIT 等特殊命令）
```

- 交互默认是 gsql 风格横排 ` | ` / `-+-`；表格超出终端宽度时自动进入 `less -S -X`，每条记录保持一行，用 `←`/`→` 横向滚动，`q` 返回 SQL 提示符，并保留退出前已显示的内容。
- 交互会话用 `\x off` / `\x on` / `\x auto` 切换横排滚动（默认）、强制竖排、自动竖排；`\q` 退出。一次性查询过宽仍自动竖排，也可用 `-v` 强制竖排；程序处理用 `-c`。
- pager 默认为 `less`，可用 `ORA_PAGER=/path/to/less` 替换；设为 `off` 则禁用。
- 它自动做：修复 `/run/user/1000` → 停着的容器自动 `podman start` 并等就绪 → `podman exec -i`（stdin 透传）→ `NLS_LANG=AL32UTF8`（中文不问号）→ 中文按双宽字符对齐
- 默认交互是同一个持久 SQL*Plus 子进程；`ALTER SESSION`、事务和临时会话状态不会因排版而丢失。返回数据以 CSV 传给宿主机 Python 排版，不再修改 `glogin.sql`。
- SQL 参数含 `$`/单引号时：外层用双引号包 SQL，字符串内用单引号（Oracle 双引号是标识符，会报 ORA-00904）

### 2.2 原生命令（备用）

以下命令是原 WSL 环境的备用操作，按实际环境使用；不是新机器必须执行的初始化步骤。业务连接串由本机私有配置通过 `ORA_CONNECT` 提供，不将实际认证值写入文档或提交仓库：

```bash
# 前置修复：每次 WSL 重启后必须先跑（无 systemd，/run/user/1000 缺失会让 podman 报
# "error creating tmpdir: mkdir /run/user/1000: permission denied"）
ORACLE_RUNTIME_DIR="/run/user/$(id -u)"
sudo mkdir -p "$ORACLE_RUNTIME_DIR"
sudo chown "$(id -u):$(id -g)" "$ORACLE_RUNTIME_DIR"
sudo chmod 700 "$ORACLE_RUNTIME_DIR"

podman start oracle26ai    # 启动：约 1~2 分钟后可连接（以连接测试为准）
podman stop oracle26ai     # 停止

# 连接测试（业务账号，一次性脚本——必须 -i，见下方坑）
podman exec -i oracle26ai sqlplus -s "${ORA_CONNECT:?请先在本机私有配置中设置业务连接串}" <<< 'select count(*) from documents;'
# SYSDBA（容器内本地认证，无需密码）
podman exec -i oracle26ai sqlplus -s / as sysdba <<< "select instance_name from v\$instance;"
# 交互式（用户习惯用法，rlwrap -H ~/.oracle_sql_history 可加命令历史）
podman exec -it oracle26ai sqlplus "${ORA_CONNECT:?请先在本机私有配置中设置业务连接串}"
```

### 2.3 连接信息与风险

**连接信息汇总**：业务账号 `由本机私有配置提供，备用命令读取 ORA_CONNECT`；SYSDBA 走容器内 `/ as sysdba`；宿主机端口 **1521 已映射可达**（JDBC/外部工具连 `localhost:1521/FREEPDB1`；本机无 sqlplus 客户端，一律 podman exec）。

⚠️ **一次性脚本必须 `podman exec -i`**（实测踩过）：不带 `-i` 时 stdin 不透传，`<<<` 里的 SQL **静默不执行**——sqlplus 正常连接、正常退出、无结果无报错，极易误判成"查询成功"。交互式才用 `-it`。

**原生 SQL*Plus 的宽表显示**（仅 `--native`/绕过 ora 默认排版时）：默认 `linesize=80`，宽表一条记录会折成多行。两种处理方式：

```sql
set linesize 300    -- 人看：一行一条记录（⚠️ SET 一次只能设一个变量，
set pagesize 100    --   合并成一条 SET 报 SP2-0268）
set markup csv on   -- 脚本/管道：直接输出 CSV，列再多不错位，null 也清晰
```

- 实例：SID=FREE，PDB=**FREEPDB1**，容器端口 1521 映射到宿主机；版本 banner = Oracle AI Database 26ai Free 23.26.2.0.0
- rootless podman，存储已搬到数据盘：`~/data/programs/oracle/containers`（`~/.config/containers/storage.conf`）
- ⚠️ **容器无数据卷挂载，数据全在容器可写层：严禁 `podman rm oracle26ai`（删容器=数据全丢）；start/stop 安全**
- `podman ps` 显示 `(starting)` 属正常：无 systemd 连接，healthcheck 跑不起来，不影响使用
- 启动初期 alert 日志可能出现 ORA-01555（自动统计任务扫向量表），非致命
- 业务测试对象：`VECTOR_TEST.DOCUMENTS` / `DOCUMENTS_PF`、`HNSW_REL_TEST.DOCUMENTS`（向量索引测试表）；scifact 数据集在 `.codex_tmp/oracle26ai_hybrid_bench_20260831/`（Codex 2026-08-31 的混合检索基准）

## 3. PG19 + pg_orca + Daft 多模优化与计算（实测 2026-09-05）

### 3.1 状态、边界与权威入口

- **真实状态**：67 个原子任务中 63 个已完成；`P2-001-D` 仍 `in_progress`，正式 72 小时稳定性门尚未开始，所以是“P0/P1 功能链已打通、未生产准入”。`P2-002` AI 算子、`P3-001` sidecar、`P3-002` 外部直扫均为 deferred，不能宣称已支持。
- **主要行为**：ORCA 在同一 Memo/成本模型中全局比较 PG 与 Daft 物理方案，只有胜出的连续 Daft 节点才机械翻译成 `Custom Scan (DaftExec)`；现已覆盖 Source、Project、Limit、Filter、HashJoin、HashAggregate、Sort、TopN。PG heap/index 仍由 PG Executor 按 MVCC/RLS/权限读取，经 Arrow C Data + 同 backend 内惰性加载的 Rust Bridge 流式送入 Daft；Bridge 不二次优化，执行开始后失败直接 ERROR、不透明换计划。
- **边界与开关**：`pg_orca.daft_enable=off` 是默认值，`auto` 缺 Bridge/能力不匹配时保留 PG 方案，`force` 只用于测试且不能绕过能力/语义门；`daft_prefer_plan=on` 仅用于证明链路，不代表自然 CBO 性能胜出。P0/P1 禁止 `DaftExec` 进入 PG parallel/background worker，也不与 `Gather/GatherMerge` 混用。
- **本机迁移源与代码入口**：`postgres19@7873db5`、`daft@55f7bc2`、`pg_orca@5277154` 是原 WSL 的源码快照，不是当前远端发布 HEAD。跨机器接续使用 §0.2 中已发布并修复的版本；本机未同步云端修复，必须保留现有差异和用户资产，禁止擅自 reset/checkout 到 `origin/main`。优化器看 `pg_orca/gpopt/daft/`，PG 执行/Arrow/loader 看 `pg_orca/compat/executor/`，Rust ABI/执行看 `daft/src/daft-pg-bridge/` 与 `daft/src/daft-local-execution/`，测试编排看 `integration/pg19-pgorca-daft/`。
- **继续开发协议**：文档唯一权威目录是 `公共文档/多模优化与计算/`；首次接手先完整执行 `执行包/MODEL_HANDOFF_PROMPT.md`，并查看 `执行包/继续开发交接提示词.md` 最新断点。先跑 `validate_package.py`、`preflight.sh report`、`taskctl.py list-ready`；当前须 `taskctl.py show P2-001-D` 恢复活动任务。`task_state.json` 只能经 `taskctl.py` 修改；不要用早期“最终报告”判断进度。

### 3.2 现成测试栈

默认端口 **55419**（实测可拉起；Codex 受限沙箱连接本地 socket 时需授权）：

```bash
I="$CPP_WORKSPACE/integration/pg19-pgorca-daft"
P="$CPP_WORKSPACE/.install/pg19-pgorca-daft"
export PGD_WORKSPACE="$CPP_WORKSPACE"
bash "$I/cluster.sh" start
"$P/bin/psql" -X -h 127.0.0.1 -p 55419 -d postgres
bash "$I/cluster.sh" stop
```

### 3.3 构建与验证

- 干净全栈只用 `$I/build_clean.sh`；单个 Bug 优先用 `pg_orca/test/test.sh --orca-tests daft_<name>`（所需 `PG_CONFIG`/`PG_REGRESS_SQL`/隔离目录见 `执行包/新人或新模型从零构建与基础测试指南.md`），当前基线为 Daft 16/16、pg_orca schedule 34/34。
- 阶段门用 `$I/run_gate.sh P0|P1-correctness|P1-performance|observability|P2-faults|P2-packaging`；**不要运行 `P2` 冒充快速回归**，它要求 259200 秒且当前 `run_mixed_workload.sh` 尚未交付。
- 选中 Daft 的实测 SQL/EXPLAIN 见 `Daft优先计划_EXPLAIN_ANALYZE实测.md`。

## 4. PG19 + Apache AGE（实测 2026-09-04）

### 4.1 两套图能力

本节记录的是原 AGE 属性统计环境；本次开发的原生多跳图执行及 AGE 邻接适配使用 **§7 的独立源码、安装与实例**，不要混用。

环境里 PG19 有**两套图能力，互相独立**，别混淆：
- **Apache AGE 1.8.0**：第三方扩展（openCypher 语法），编译安装在 `.install/pg19-pgorca-daft`（PG19beta2）——2026-09-04 当时检查的四套安装（pg19-debug / pg19-asan / **pg19-pgorca-daft** / postgis-deps）**只有这套有 AGE**，ORCA 回归用的 pg19-debug 没有；当前 §7 的 `pgq-integrated` 也有配套 AGE
- **SQL/PGQ**：PG19beta2 内核自带（`CREATE PROPERTY GRAPH` / `GRAPH_TABLE()`，见 postgres19 源码 gram.y），SQL:2023 标准语法，与 AGE 无关

AGE 源码/移植：`apache-age-pg19-port/`（PGXS 方式，`make install PG_CONFIG=.install/pg19-pgorca-daft/bin/pg_config`）。

### 4.2 现成实例

默认停着：`.runtime/age-properties-stats-bench-20260902/`，端口 55439，AGE 装在 **`age_stats_bench`** 库，图：`stats_demo`、`age_prop_stats_bench`。

```bash
R="$CPP_WORKSPACE/.runtime/age-properties-stats-bench-20260902"
G="$CPP_WORKSPACE/.install/pg19-pgorca-daft"
$G/bin/pg_ctl  -D $R/data -o "-p 55439 -k $R/socket" -l $R/postgresql.log -w start   # 启动
$G/bin/pg_ctl  -D $R/data -m fast -w stop                                            # 停止
```

### 4.3 会话初始化

新会话里跑 cypher 前必须先执行前两句：

```sql
LOAD 'age';                                  -- 不加 → ERROR: unhandled cypher(cstring) function call
SET search_path = ag_catalog, public;        -- 不加 → ERROR: type "agtype" does not exist
SELECT count(*) FROM cypher('stats_demo', $$ MATCH (n) RETURN n $$) AS (v agtype);
```

关联：`deliverables/age_properties_stats/`（AGE 属性统计基准，2026-09-02 跑过）；另有一份 PG16 的 age.so 在 `postgres16/install/`。

## 5. openGauss 通用构建与实例管理

### 5.1 按用途选择构建脚本

DiskANN、HybridRecall 等 openGauss 相关项目**不要手工 configure/make**。日常功能开发、正确性和调试使用下面的 Debug/O0 脚本：

```bash
BUILD="$CPP_WORKSPACE/deliverables/multimodal_optimizer/opengauss_build_install_restart.sh"

$BUILD incremental   # 增量 make → 停库(fast) → install → 重启 → SQL 检查（日常默认，约 8 分钟）
$BUILD full          # make clean + 重新 configure + 全量编译 + 安装（约 38 分钟）
$BUILD restart       # 不编译，仅停库重启
```

HybridRecall 与 Milvus 做任何性能 PK 时，必须改用 Release/O2 脚本；**Debug/O0 数字不得进入性能结论**：

```bash
RELEASE_BUILD="$CPP_WORKSPACE/deliverables/hybrid_vs_milvus_bench/build_release.sh"

$RELEASE_BUILD incremental   # 已有 Release 配置且只改 .cpp
$RELEASE_BUILD full          # 首次构建或改过头文件
$RELEASE_BUILD restart       # 不编译，仅重启
```

Release 脚本会校验 `version()` 不含 `debug on`；其 configure 固定 `-O2`、`enable_debug=no`、`enable_cassert=no`。不要仅凭 ELF 含 `debug_info` 判断成 Debug 构建，保留符号是为了 perf 解栈。

配套进度条（另开终端）：`opengauss_build_progress.sh`（同目录，`-w` 实时重绘）。

### 5.2 脚本默认配置

下表记录原机器默认值；新机器必须按实际工作区显式配置路径，不能假定脚本的硬编码默认值随 `CPP_WORKSPACE` 自动变化。现成实例不等于新机器已经建库，禁止为套用示例覆盖已有数据。

可用环境变量覆盖：`CODE_BASE`、`BINARYLIBS`、`GAUSSHOME`、`PGDATA`、`PGPORT`、`JOBS`、`DB_USER`、`LOG_DIR`。

| 项 | 默认值 |
|---|---|
| 源码 CODE_BASE | `/home/qiumc/data/code/cpp/openGauss-server` |
| 三方库 BINARYLIBS | `/home/qiumc/data/code/cpp/binarylibs` |
| GAUSSHOME | `$CODE_BASE/mppdb_temp_install` |
| PGDATA | `$CODE_BASE/topk_test_data`（单机实例，端口 **55440**，DB 用户 `$USER`） |
| JOBS | 4 |
| 日志 | `$CODE_BASE/build_logs/<mode>_<时间戳>.log` |
| 编译器 | binarylibs 自带 GCC 10.3（`buildtools/gcc10.3`） |

日常脚本 configure 参数以脚本为准（`--enable-debug --enable-cassert --enable-thread-safety`，O0）；性能脚本为 `-O2` 且无 debug/cassert。
**不要**采用源码树 `OPENGAUSS_BUILD_INSTALL_RUN_GUIDE.md` 或其他文档里的 configure 参数（`--enable-mot --enable-bbox --enable-htap` 为陈旧变体，与当前树不符）。

### 5.3 原 WSL 连接方式与运行记录（实测 2026-10-19）


实例数据目录为`topk_test_data`，端口55440，本地trust免密。**2026-10-19 14:41同线程重新搜索诊断收尾，OG/Milvus/Oracle/私有55450/原生25632均停止；HEAD及产品仍`e84f354971`，见§6.38**。三个.cpp临时改动已撤回并Release重编，无新产品优化；§6.37的Local Stream提前EOF修复保留，无搜索/成本/索引变化。

权威安装仍为`.install/hybrid-recall-pure-validation/bin/gaussdb`，14:35:34恢复构建后安装及实际验证PID3076055 SHA256=`efe0796ebbca2cf4f9ecc16372aba28fe2b8b77a1e313f5acccca6f0304dda8e`；**e420是临时诊断版、0b2是旧产品，均不是当前安装**。本轮恢复版原生5/5、85种交错、12872次四模式有序ID、并发400/20错误/80超时恢复通过；定向10000及批次/普通Stream的前轮范围见§6.37。banner仍为configure时4db9d8d4/2026-10-15 16:56:55，不能代替源码/构建/运行hash。配置精确恢复8c83e997…、默认池OFF，临时perf探针/亲和性已清理。Milvus未启动；此前stop退出134、OOM=false，下次启动检查恢复日志，严禁删除容器或基准数据。默认源码树mppdb_temp_install不是本测试权威安装。

```bash
G="$CPP_WORKSPACE/.install/hybrid-recall-pure-validation"
export LD_LIBRARY_PATH=$G/lib        # 必须！裸调 gsql 报 libssl.so.3 找不到
$G/bin/gsql -X -d postgres -p 55440 -c "select version();"
```

- 性能实例启停必须为 Release 脚本显式传入正确的 `CODE_BASE/GAUSSHOME/PGDATA/PGPORT`，完整命令见下一节；不要让默认值误启另一工作树的二进制。
- 若系统 libstdc++ 版本不匹配，`LD_LIBRARY_PATH` 还需追加 `/home/qiumc/data/code/cpp/binarylibs/buildtools/gcc10.3/gcc/lib64`；本次测试客户端使用了该路径。
- 正常停止用对应安装树的 `gs_ctl stop -D <PGDATA> -m fast -w`；仅在实例挂起时使用 `-m immediate`。
- 现有库：`postgres`、`test`（回归用）、`hybrid_recall_p31/p32/p33_20260826`、`vf_iso_20260831`、`diskann_bypass_bench_20260902`。

### 5.4 编译策略

- **能增量就不要全量**：仅修改 `.cpp` 时优先 `incremental`，一轮约 8 分钟。
- **修改头文件必须 full**：当前源码树未启用 `--enable-depend`，增量构建不会可靠重编依赖对象，可能形成 ABI 混合二进制。
- 首次构建、切换 Debug/Release 配置或怀疑 `config.status` 口径不符时，也必须 `full`。

### 5.5 已知构建与测量陷阱

1. **JRE 只读权限**：`src/gausskernel/Makefile:690` 使用无 `-f` 的 `cp -r` 安装 JRE，第二次安装可能因系统 JRE 文件为 444 而失败。修复：`find mppdb_temp_install/jre -type f ! -writable -exec chmod u+w {} +`。
2. **LLVM 链接错乱**：系统 LLVM 可能遮蔽 binarylibs 归档并引发 `llvm::EnableABIBreakingChecks` 等 undefined symbol。已在 gausskernel Makefile 中让 `-L$(LIBLLVM_LIB_PATH)` 前置；复发时用 `ld --trace-symbol` 诊断。
3. **数据目录不会自动初始化**：脚本要求 `PGDATA/PG_VERSION` 已存在；SQL 连通失败只会告警，不能只看脚本退出过程判断实例可用。
4. **环境变量泄漏**：错误的 `GAUSSHOME` 等变量会让脚本找不到实际 `gs_ctl`；执行前核对或清除外部遗留变量。
5. **实例挂起**：并行查询卡死、`gs_ctl stop -m fast` 无法停止时，才使用 `gs_ctl stop -D <PGDATA> -m immediate`。
6. **安装阶段较慢**：不要用过短超时杀死链接或安装，避免留下半写产物；必要时按统一脚本的阶段拆开观察，但不要另造 configure 参数。
7. **Debug/Release 不能交叉增量**：`incremental` 沿用当前工作树的 `config.status`。若该工作树曾被 Debug 脚本重新 configure，性能测试前必须先跑 Release `full`，并核验 `config.log` 和 `version()`。
8. **构建会污染缓存**：全量构建与同时启动 Milvus 在约 8 GB 主机上容易触发换页。构建完成后再启动对照服务并充分预热；正式窗口出现 swap-in/out 或 major fault 必须作废。
9. **EXPLAIN 计时使用 TEXT**：本环境的 `EXPLAIN (FORMAT JSON)` 曾出现约 20 倍计时伪影。perf 图只定位 openGauss 内部 CPU 去向；没有 Milvus 服务端火焰图时，不能据此拆解两端内部差额。

## 6. HybridRecall 文本+向量两路召回（本机性能记录截至 2026-10-19）

接手顺序：确认分支和 dirty 状态（§6.1）→定位代码与既有优化（§6.2）→用 Release/O2 编译并跑回归（§6.3）→复用现有数据和索引（§6.4）→按等召回口径验收（§6.5～§6.6）→依据火焰图决定下一改动（§6.7）。

### 6.1 本机历史测试基线、工作树与文档

跨机开发使用 §0.2 的 `codex/migrate-hybrid-recall@bdda19c2`。下表保留 2026-10-19 性能记录对应的工作树和版本，不能把其中的旧 HEAD 或二进制直接当作云端修复版。

| 用途 | 工作树/分支 | HEAD | 说明 |
|---|---|---|---|
| 纯 HybridRecall 开发与性能测试 | `/home/qiumc/data/code/cpp/openGauss-hybrid-recall-consolidate`，`feature/hybrid-recall` | `e84f354971` | **2026-10-19 性能基线**；EOF修复见§6.37，同线程重新搜索诊断/恢复见§6.38；HNSW差额未定因，无新产品优化/Milvus PK |
| DiskANN 等多模联调 | `/home/qiumc/data/code/cpp/openGauss-server`，`integration/multimodal-stacked` | `bf59a66d8b` | 堆叠集成分支，不作为纯 HybridRecall 性能归因基线 |

- “两分支均未 push”是当时的本机记录；HybridRecall 已于 2026-11-08 迁移并修复发布，最新远端见 §0.2，原集成工作树不等于该独立特性分支。旧 `fix/hybrid-recall-worker-guc-sync` 已删除。恢复点为 `backup/feature-hybrid-recall-pre-optimization-20260907` 和 `backup/hybrid-recall-optimized-stacked-20260907`，**不要把 backup 当开发主线，也不要 reset 到 `origin/main`**。
- 主工作树现有用户资产 dirty，纯工作树全量编译也会生成大量未跟踪中间文件。每次先看 `git status --short`，只 add 自己的目标文件；不要 `git clean`、reset 或处理无关文件。
- 历史性能交接入口：`公共文档/两路召回对比Milvus/交接文档_继续优化.md` §0.54→§0.53；总报告`HybridRecall与Milvus测试数据、有效优化及结果总结.md` §4.5.33（同线程重新搜索，未定因），产品EOF修复见§4.5.32，BM25重叠因果见§4.5.31，首次触碰组件见§4.5.30，完整执行三轮见§4.5.25。仓内`docs/hybrid_recall/BASELINE.md`§31对应该历史性能版本；云端修复说明见最新分支的`docs/hybrid_recall/REVIEW_FIXES.md`，不把旧性能数字归给新 SHA。最近Milvus单档仍为旧产品§4.5.18，不能冒充本版新PK。新诊断不能据第二遍worker单窗较快称反超；缺完整干净双向配对，不无限重跑或丢弃异常。

### 6.2 架构、代码入口与有效提交

- 阻塞算子 `Hybrid Recall` 组织 `Limit→BM25 Index Scan` 和 `Limit→Ann Index Scan(HNSW)`，按稳定非空 ID 去重并做加权 RRF Top-K。支路 task worker/`LOCAL GATHER` 并行不是 HNSW 扫描内并行；本项目只验收 BM25+HNSW，不要与 DiskANN 或 UNION ALL Top-K 混淆。
- `hybrid_recall_parallel_mode=off|auto|force|vector`：**最新性能主档须用`auto`，不得再固定off来替代代价验收**（用户2026-10-14要求，见§6.21）。历史OFF结果保留；AUTO可以选择串行，但须同版本、同质量强制OFF/FORCE/VECTOR实测验证选择。`force`是双支路worker，`vector`是仅向量支路worker的实验能力，当前AUTO只枚举串行与双worker。
- 业务优先 `ogai.hybrid_search_v2`；`hybrid_rrf_marker(...)` 只是优化器 marker，不能逐行执行。`enable_hybrid_recall` 默认 `off`；V2 验收用 `on_unsupported=error`，过滤放 JSON `filters`，`p_where_clause` 必须为空，可用 `query_vector` 跳过外部 embedding。
- 同一普通表须有 BM25+HNSW 索引；顶层只读单表 SELECT、唯一 `ORDER BY marker DESC`、正有限 `LIMIT`、无 `OFFSET`，不支持 JOIN/子查询/聚合/窗口/DISTINCT/集合操作/行锁。安全 filter 同时下推并在融合层 recheck；支路 `ORDER BY` **不能加 tie-break**，否则破坏索引有序路径。
- Planner：`src/gausskernel/optimizer/path/hybridRecallPath.cpp`；Executor：`src/gausskernel/runtime/executor/nodeHybridRecall.cpp`；API：`contrib/ogai/ogai--1.0.sql`；HNSW 搜索热路径：`src/gausskernel/storage/access/datavec/hnswscan.cpp`、`hnswutils.cpp`。

当前纯分支的重要优化提交：

| commit | 内容 | 不可忘记的结论 |
|---|---|---|
| `aa39527a48` | stream worker 同步 datavec GUC | 修复并行支路静默使用默认 ef |
| `39c47295d8` | HNSW 上层下降 ef 1→8 | recall@100 约 0.75→0.9995，是质量根因修复 |
| `5417434c2e` | auto stream startup 成本 150→5000 | 防止默认 auto 选择 2.6x 次优计划 |
| `891941e164` | task stream 批 16 + `vector` 模式 | 交付成本下降，但端到端仍不胜串行 |
| `2dd1883b2a` | 内置 vector 距离直达、discarded 延迟物化、同页邻接 TID 预取 | HNSW 性能主代码；未改页格式，不需要重建 HNSW |
| `7546c32f1e` | RRF 直接有序索引支路的无用原始分数投影替换为同类型 NULL | 结果 ID/payload/融合分数不变，SciFact 高档诊断 CPU/请求下降 9.45%；端到端同期新差距尚未验收 |
| `c7d69eb4d7` | RRF 无过滤本线程文本路跳过无人使用的 BM25 原始分数查找表 | 排序/扩批/正常 MVCC 不变；BM25 子树估算 CPU/请求 0.60242→0.21809 ms；四档 ID 不变，正式同期时延/QPS 尚未验收 |
| `1fed7772a6` | 会话级有校验buffer编号提示＋最多4页有界预取 | Release full、原生2/2、驱逐/线程池迁移/并发/672语义通过；SciFact采样整请求CPU观测下降23.17%，不是Milvus正式胜出 |
| `4c6383e5d5` | 单个有界、逻辑清空的会话HNSW扫描工作区 | `hnsw_scan_cache_size=2048`显式开启，默认0；不缓存结果。高档minor fault约362→0.34/请求，四档同二进制改善；原生2/2及24生命周期、线程池并发通过，不是正式Milvus PK |
| `f17b450809` | 原内积循环返回前清理AVX/SSE边界 | 不改计算顺序/候选/索引；Release、672语义、当时四档质量、原生2/2通过；旧口径同期样本见历史§6.16 |
| `dfe48f9706` | 可选、冻结Unicode16的BM25 `standard_v1` analyzer | 统一共同参考质量语义，不是独立速度收益；默认Jieba/旧索引不变，见§6.19 |
| `292303d8ea` | 小索引visited初始容量参考正reltuples，仍按需扩容 | SciFact M64/ef350固定4MiB工作区时CPU/请求观察下降约4.3%～4.8%；原生4/4、四档ID不变；不改搜索/索引，见§6.20 |
| `b47f4fb9a5` | task worker同步BM25的topk/k1/b | 修复并行忽略评分和初始批参数；Release/O2、原生5/5、四模式全量有序ID一致。AUTO四档选择方向与实测一致，正式环境未过，见§6.21 |
| `099656b971` | task Stream按正值常量候选上限取min(limit,100)交付批次 | 替代固定16；未知形态仍16、普通Stream不变，100候选发送等待0；完整执行D2观察领先、SciFact仍慢，端到端仍不胜串行，见§6.27 |
| `e84f354971` | Local Stream先读完成标记，再读队列状态；配对读/写屏障 | 修复旧EMPTY＋新结束导致最后一批候选丢失；原版可控交错实证、Release与正确性回归通过；不是HNSW性能优化，见§6.37 |

### 6.3 纯分支 Release/O2 构建、启动与回归

```bash
W="$CPP_WORKSPACE/openGauss-hybrid-recall-consolidate"
R="$CPP_WORKSPACE/deliverables/hybrid_vs_milvus_bench/build_release.sh"
I="$CPP_WORKSPACE/.install/hybrid-recall-pure-validation"
D="$CPP_WORKSPACE/openGauss-server/topk_test_data"

# 只改 .cpp；如果改了任何头文件则把 incremental 换成 full
CODE_BASE=$W GAUSSHOME=$I PGDATA=$D PGPORT=55440 JOBS=6 bash "$R" incremental

# 已构建时只启动；脚本会检查实际版本不是 debug
CODE_BASE=$W GAUSSHOME=$I PGDATA=$D PGPORT=55440 bash "$R" restart

# 连接与二进制核验
env LD_LIBRARY_PATH=$I/lib $I/bin/gsql -X -h /tmp -p 55440 -d postgres -c 'select version();'
```

- 首次接手先确认 `ss -ltnp | rg ':55440|:55441|:55450'`；openGauss 单机还会占相邻内部端口，曾因 55440 实例让隔离 55441 启动失败，隔离实例应使用 55450。
- 定向回归target在 `src/test/regress/GNUmakefile`；当前Release/O2基线为 **5/5**（bm25_standard、hybrid_recall、hybrid_bm25_worker_guc、hnsw_buffer_hints、hnsw_visited_sizing）：

  ```bash
  GCCLIB="$CPP_WORKSPACE/binarylibs/buildtools/gcc10.3/gcc/lib64"
  env GAUSSHOME=$I SSL_LIB_PATH=$I/lib LD_LIBRARY_PATH=$I/lib:$GCCLIB \
    make -C "$W/src/test/regress" bm25_standardcheck \
      EXTRA_TESTS='hybrid_recall hybrid_bm25_worker_guc hnsw_buffer_hints hnsw_visited_sizing'
  ```

- 仅 `.cpp` 改动使用 incremental；改头文件必须 full（未开 `--enable-depend`，否则可能形成 ABI 混合二进制）。构建后先验证结果 ID/召回，再测性能；HNSW 热路径 `.cpp` 改动通常不需要重建索引。
- Local Stream接收改动还须运行`python3 "$W/src/test/ut/local_stream_eof/check.py"`（85种确定性交错）；当前产品应85/85通过，`--git-ref 5bde98afc0`用于负向对照，预期18失败/退出1。此单测不替代ARM弱内存实机验收。

### 6.4 现成数据与 Milvus 环境

- openGauss：`postgres.hybrid_bench_d2_all_plain`，10 万行/768 维，正文与向量均为 PLAIN，主键/BM25/HNSW 索引已经存在。**不要重新运行建表或建索引脚本**，先核验后复用。
- Milvus：容器 `milvus-standalone`，官方 2.6.22，collection `hybrid_bench_d2`，10 万行，dense HNSW `M=16/efConstruction=200` 和 sparse BM25 索引均已完成。数据卷保留，`podman start milvus-standalone` 即可复用；严禁 `podman rm`。
- **启动容器不保证 collection 已 load。** ABBA 客户端不会自动加载；换数据集时先检查 `get_load_state`，显式 `load_collection('hybrid_bench_d1'/'hybrid_bench_d2')` 并确认 Loaded，再预热。可以 `release_collection` 释放闲置集的内存，不是删除数据，也不要因此重建索引。
- 测试前记录各服务初始状态，只启动必要资源；性能测试时 Oracle 必须停止。测试结束恢复初始状态，默认不留 openGauss/Milvus/Oracle 常驻影响后续测试。
- openGauss 客户端走 Unix socket `/tmp:55440`；Milvus 走 `http://127.0.0.1:19530`，Python 已设置 `no_proxy=127.0.0.1,localhost`。两服务必须同期在线，不能拿不同日期的孤立数字拼 PK。

### 6.5 公平 PK 口径与当前基线

- 外层用户契约固定最终 Top-10、等权 RRF k=60。双方内部候选和 ef 应各自寻优，并在 recall 相等后比较性能；不要强行统一内部 100+100。
- openGauss 的 `bm25_topk` 是 BM25 **初始批量**，不是最终 Top-K。保持默认 5 会多次扩批并显著变慢；95%/99% 主档必须显式设为 55/100。
- 旧 `gt_fusion.npy` 在 BM25 同分截断处受 `argpartition` 任意选择影响，旧 `0.8802` 不再是公平主口径。权威质量参考目录为 `公共文档/两路召回对比Milvus/验证/Pareto寻优/高召回共同基准_D2/`：分别用两端实际 BM25 与共同暴力余弦构造两份参考，对两份 recall 取较低值。

严格等双参考召回配置与 2026-09-07 历史同期结果如下；10 月合格正式结果见 §6.8，当前代码验收状态见 §6.9，不要把下表当新提交的实测：

| 档位 | openGauss 参数（text/vector/ef/bm25_topk） | Milvus 参数（sparse/dense/ef） | OG P50/P95/P99 | MV P50/P95/P99 | OG/MV QPS |
|---|---|---|---|---|---|
| recall@10=0.9502 | 55/75/75/55 | 90/70/70 | **1.872/2.569/2.968 ms** | 2.480/3.247/3.687 ms | **513.01/388.78** |
| recall@10=0.9904 | 100/100/106/100 | 100/100/142 | **2.297/3.162/3.721 ms** | 2.657/3.438/3.888 ms | **417.73/364.14** |

这只是 D2、热态、单连接、无额外过滤、`hnsw_iterative_scan=relaxed_order` 的结论；D1 长正文、并发 4/8、冷态、更大规模、过滤矩阵和并发更新尚不能外推。99% P99 领先仅 4.30%，应表述为本次样本中的小幅领先，而非 SLA。

### 6.6 历史双参考 95%/99% OFF 同期 ABBA 复跑

本节只复现9月历史OFF基线；当前共同参考AUTO主档见§6.21，不能用本节替代AUTO代价验收。

输出目录必须是新目录。脚本每阶段至少预热 10 秒、测量 30 秒，并在完整 500 查询周期边界结束；保存逐查询时延、结果 ID、环境和分页计数。

```bash
P="$CPP_WORKSPACE/公共文档/两路召回对比Milvus/验证/PLAIN对照/同期稳态ABBA.py"
GT="$CPP_WORKSPACE/公共文档/两路召回对比Milvus/验证/Pareto寻优/高召回共同基准_D2"
E="$CPP_WORKSPACE/公共文档/两路召回对比Milvus/验证/Pareto寻优"
RUN_ID="$(date '+%Y%m%d_%H%M%S')"

podman start milvus-standalone
# 等 Milvus 19530 就绪后再开始；openGauss 应已由上一节的 Release restart 启动

# 95% 等召回
python3 "$P" --out "$E/纯分支复测_召回95_$RUN_ID" --seconds 30 --warmup 10 --rounds 2 \
  --iterative-scan relaxed_order --bm25-topk 55 --og-parallel-mode off \
  --og-text-candidates 55 --og-vector-candidates 75 --og-ef 75 \
  --mv-dense-candidates 70 --mv-sparse-candidates 90 --mv-ef 70 \
  --reference-dir "$GT"

# 99% 等召回；主机换页偶发，建议直接跑 4 轮再选干净对称轮次
python3 "$P" --out "$E/纯分支复测_召回99_$RUN_ID" --seconds 30 --warmup 10 --rounds 4 \
  --iterative-scan relaxed_order --bm25-topk 100 --og-parallel-mode off \
  --og-text-candidates 100 --og-vector-candidates 100 --og-ef 106 \
  --mv-dense-candidates 100 --mv-sparse-candidates 100 --mv-ef 142 \
  --reference-dir "$GT"
```

测量纪律：

1. 构建/重启会驱逐缓存，先强预热；第一次极冷窗口不能混入正式结论。
2. 检查每个 `round*_measured.json` 的 `host_vmstat_delta`。任一正式窗口 `pswpin`、`pswpout` 或 `pgmajfault` 非零，就排除相应完整轮次并补跑；必须最终选到一轮 OG→MV 和一轮 MV→OG 的干净对称组合。不能只发布含污染窗口的聚合 `summary.json`，也不能按性能好坏挑轮次。
3. 新旧结果先比较 recall 和 `ordered_ids` SHA-256，再比较 P50/P95/P99/QPS。绝对值随主机状态变化；两端同时变快不能算成 openGauss 新代码收益。
4. 9 月原始证据（10 月最新入口见 §6.8）：95% 为 `公共文档/两路召回对比Milvus/验证/Pareto寻优/feature分支复测_召回95_OG55_75_ef75_MV70_90_ef70_hot_rerun/summary_clean_symmetric_hot_cross_run.json`；99% 为 `公共文档/两路召回对比Milvus/验证/Pareto寻优/feature分支复测_召回99_OG100_100_ef106_MV100_100_ef142_balanced4/summary_clean_symmetric_rounds0_1.json`。两端 recall 与有序 ID hash 均完全复现历史正式结果。
5. 收尾停止本轮服务但不删除数据：`env LD_LIBRARY_PATH=$I/lib $I/bin/gs_ctl stop -D "$D" -m fast -w`，随后 `podman stop --time 30 milvus-standalone`；再用 `ss`/`podman ps` 核验。

### 6.7 D2 历史火焰图结论（SciFact 最新断点见 §6.9）

- 最新有效图：`公共文档/两路召回对比Milvus/验证/火焰图/D2_all_plain_slow153_135_同页邻接预取_v2/cpu.svg`，机器汇总同目录 `bottleneck_breakdown.json`。它针对困难查询 q153/q135，5,047 样本，无质量 warning。
- 完整请求 CPU 样本：HNSW 64.65%，其中 buffer lookup/管理 19.04%、`VectorInnerProduct` 13.35%、`HnswLoadUnvisitedFromDisk` 含子调用 5.77%；BM25 13.57%，SQL 生命周期 11.71%。火焰图宽度是 CPU 样本占比，不是墙钟，也不能把包含关系的函数占比相加。
- 距离直达后 HNSW 子树已不再经过 `FunctionCall2Coll`、`vector_negative_inner_product` 或 `pg_detoast_datum`。完整图中同名调用来自 BM25/其他 SQL 路径，不能再把 HNSW 瓶颈写成 TOAST 解压；SQL Bypass 也已确认执行，但不会消除 HNSW 图遍历本体。
- 当前第一剩余瓶颈是每个新 element 的首次随机页访问：`ReadBuffer → BufferAlloc → BufTableLookup/buf_hash_operate → PinBuffer/LockBuffer`，其次是 768 维内积。困难查询 shared hit 约 1700～1800，普通查询约 800～900，是 P99 查询相关性的主要来源。
- 已否决且不要重复：仅内联 detoast 快检（只移动栈归属、无端到端收益）、四累加器 AVX（无收益且排序漂移）、扫描级小 LRU（命中仅约 4%～10%）、合并候选/pairingheap 分配（候选管理占比太小）、全局关闭迭代续扫（过滤/MVCC 会返回不足）。
- 下一步先补 `2dd1883b2a` 的并发 VACUUM/UPDATE/DELETE、槽位复用、非同页和 PQ/RabitQ/LSG/mmap 回退语义门；再研究元素页首次访问的安全布局/不可变读路径。任何绕过 buffer 的设计必须先定义 pin/锁、更新失效、回收和内存上限。
- 采样入口为 `验证/火焰图/采样融合CPU.py`，先跑 `--help`；`/usr/bin/perf` 已存在，需 `--sudo`，还要传入包含 `stackcollapse-perf.pl`/`flamegraph.pl` 的 FlameGraph 目录。`perf.data` 含原始用户栈字节，必须私有保存。新采样输出目录不得覆盖既有证据，采样期间的延迟只作诊断，正式性能仍以未采样 ABBA 为准。

### 6.8 SciFact 投影优化及正式基线（2026-10-13 第一阶段，最新断点见 §6.9）

- 第一阶段提交 `7546c32f1e`，改动在 `createplan.cpp:hybrid_elide_unused_score()`：只对 RRF 的直接 `Limit → IndexScan/AnnIndexScan` 消除未使用的原始分数投影，保留排序、qual、MVCC、候选、ef 与 payload 列位置；不修改 HNSW 页格式。Release/O2 增量构建、原生 `hybrid_recallcheck` 1/1 与 96 次过滤/payload/并行模式/计划缓存语义对照通过，四档全部查询有序 ID 不变。
- 新证据目录 `公共文档/两路召回对比Milvus/验证/SciFact投影优化_20261013/`。SciFact 高档 `text100/vector80/ef400` 的新图 `火焰图_ef400/cpu.svg` / `bottleneck_breakdown.json`：重复余弦/BM25 输出分数子树消失，user CPU/请求 5.3454→4.8404 ms（9.45%）；HNSW 75.89%，其内积 19.11%、buffer lookup 14.73%，均为完整请求 CPU 样本占比且存在包含关系。向量 EXTERNAL 未压缩，剩余 PGLZ 来自 BM25；不能将其说成 HNSW 解压。
- **该阶段正式时延/QPS 未验收。** D2 95% 的四次完整 ABBA、SciFact 低档的两次完整 ABBA 均未得到干净对称轮；第五/第三次分别取消，原始文件保留。高档 SciFact 和 D2 99% 在该阶段未完成，已在 §6.9 的新版本上补测。后台换页即使来自编辑器等非数据库进程，仍不放宽 §6.6 的零换页门；不停止其他用户工作。不把采样 CPU 降幅当成 Milvus 差距缩小百分比。
- 最近合格的正式结果在 `验证/双数据集复测_20261013/*验收.json`，均为投影优化前代码：D2 95.02%/99.04% 的 OG P50 低 22.26%/4.68%、QPS 高 28.73%/6.07%；SciFact 80.0721%/81.6411% 的 OG P50 高 55.76%/92.60%、QPS 低 34.73%/47.14%。SciFact 两参考差异使最低 recall 理论上限约 92.36%，不能照搬 95%/99% 档，也不是 ANN 本身的召回上限；人工 nDCG/qrels Recall 另算。
- SciFact 数据为 `hybrid_bench_d1`，5,183 文档、768 维词袋随机投影、1,109 查询（300 有 qrels），复用现有索引。低档 OG `text100/vector80/ef200/bm25_topk100`、MV `sparse117/dense80/ef400`；高档 OG `100/80/400/100`、MV `99/100/560`。参考目录 `验证/双数据集复测_20261013/SciFact_共同基准/`。改成 OG vector100/ef252 虽匹配高档双参考 recall，但人工 nDCG 降低，已否决为本轮有效优化。
- 正式复测使用原 `同期稳态ABBA.py`（SciFact 显式 `--dataset d1 --queries 1109`）及 `双数据集复测_20261013/验收汇总.py`。`SciFact投影优化_20261013/四档同期复测.py` 已支持 `--out-root <新目录>`，仍拒绝覆盖已有路径，不能直接重复旧命令；当前四档运行情况和资源状态以下节为准。

### 6.9 BM25 名次扫描与最新复测断点（2026-10-13 18:35）

- 当前代码 `c7d69eb4d7`：`nodeHybridRecall.cpp:hybrid_enable_rank_only_bm25()` 只为无过滤、本线程直接 `Limit → IndexScan`、普通 heap/BM25、原始分数投影为 typed NULL 且无其他表达式消费者的 RRF 设置 `BM25ScanOpaqueData::skipScoreKeys`。`bm25scan.cpp` 仅跳过 `ConstructScanScoreKeys()` 的重复回表、正文提取/解压与原始分数哈希构建；打分/排序/扩批和正常 IndexScan MVCC 不变，其他计划形态与普通 `<&>` 回退原路径。
- 原型陷阱已修复：Limit 的 NULL 会被 setrefs 改为 OUTER_VAR，必须在实际 IndexScan 上验证 typed NULL；跳过分数表后，倍增和最终 TAAT 扩批的两处 `DELETE_EX` 必须判 NULL。首版未启用、v2 会在扩批时崩溃，**只使用最终 v3 证据与 §5.3 的二进制 hash**。头文件改动已 full，最终增量日志 `build_logs/release_incremental_20261013_175628.log`。未改索引格式、未重建基准索引。
- 证据根 `公共文档/两路召回对比Milvus/验证/SciFact名次扫描优化_20261013/`。`修改后v3语义.json` 的 672 次对照/96 个 hash 全等；`MVCC验证_v3.json` 和 `TAAT扩批与MVCC验证_v3.json` 共 20 组通过，含 text2800/bm25_topk5 的 TAAT 扩展；原生 `hybrid_recallcheck` 1/1 通过。四档全部有序 ID 与历史正式结果一致，双参考 recall 与人工 qrels 指标不变。小表检查不替代 HNSW 完整并发语义门；旧版 `WHERE 原始BM25分数 > 0` 的融合错误仍是未扩大支持的既有边界。
- 最新图 `火焰图_名次扫描_v3/cpu.svg`，对照 `火焰图_快路径未启用/cpu.svg`，各有 `bottleneck_breakdown.json`：BM25 子树估算 user CPU/请求 0.60242→0.21809 ms，占比 11.8664%→4.9988%，PGLZ 样本 89→0。两图 4222/4281 样本、完整图无 warning。整请求采样 CPU 观测下降 14.06%，但 HNSW 耗时也波动，不能全归因于代码、不能与上一阶段 9.45% 相加或当成端到端收益。
- **最新四档同期都已运行，但无正式合格结果。** SciFact 低档两次、高档一次、D2 95%/99% 各一次，共 5 次完整 ABBA/10 个双端配对轮次/20 个测量窗口，全部配对轮次受换页污染。审计为 `v3同期质量与环境审计.json`，原始目录为 `同期v3/`（SciFact）及 `同期v3_补测/`（D2）。首次 D2 collection 未 load 的中止日志保留；不能用其 OG 单端数据拼 PK。§6.8 所列旧版合格性能差距仍是最新正式口径。
- 接手先核对源码/二进制，再 load 目标 Milvus collection，在新目录复测；若主机仍持续后台换页，不要无限重试或放宽门槛。HNSW 最新占完整请求 CPU 80.8456%，内积 21.9108%、BufferAlloc 15.8374% 包含在其中；下一项代码改动前先补并发 VACUUM/UPDATE/DELETE、槽位复用与特殊编码回退验证，绕过 buffer 的方案须先定义 pin/锁、失效、回收和内存上限。
- 两服务已正常停止，Oracle 未启动；停止前 Milvus D1 已 release、D2 已 load，下次仍须查实际 load 状态。独立验证表已清理，基准数据、索引、容器及无关用户资产保留。产品改动已本地提交，未 push。

### 6.10 HNSW buffer提示已提交、正式PK继续（2026-10-13 22:15）

- 用户要求持续优化到优于Milvus；目标尚未完成。证据根 `公共文档/两路召回对比Milvus/验证/HNSW缓存预取_20261013/`，详细设计/失败记录/任务见交接文档§0.24。`1fed7772a6` 已提交7个产品/回归/仓内基线文件，跟踪文件干净，未push，未迁入集成分支。
- 整页预取 v1 已被 CPU 退化否决。v2 改为最多4页pin、只提示tuple前512字节，保留正常内容锁；v3 另加每线程约32 KiB的单索引页→buffer编号提示。v3命中仍通过原PinBuffer、完整BufferTag/BM_VALID校验和ResourceOwner，失效回退ReadBuffer，不持久pin、不缓存页内容或查询结果；特殊存储/恢复/量化/mmap保留原路径。**不能使用 `rd_amcache` 存通用HNSW缓存，GSC复制代码按BTMetaPageData处理它。**
- v3 Release/O2：672次语义/96 hash、32/2000维各100轮并发通过；新增16MB buffer/约64MB索引、44组低缓存生命周期及5次超时取消后恢复通过，8形态×3查询的特殊/普通路径通过。PQ初始化和mmap在当前x86受限，均未实测通过，不能绕过架构门。相邻同二进制火焰图 `火焰图_编号提示v3{关闭}_独占/`：整请求采样user CPU 4.81966→3.71957 ms、HNSW 3.90156→2.78184，HNSW内哈希样本330→0；不是正式Milvus PK。
- 原生门拒绝v3新增独立THR_LOCAL（247→248）；未改门槛。最终v4移入 `knl_u_datavec_context`，SessionCacheMemoryContext按需分配/结束释放；full日志 `build_logs/release_full_20261013_211240.log`，21:23完成。`提示v4*`证据：原生2/2、低缓存44组、8形态24组、672语义/96 hash和新参数1109条全部通过；线程池2线程下读会话实际在两个worker间迁移，两维度各300读/100写/100VACUUM通过。私有表清理，基准索引未重建。
- 最终v4同二进制火焰图 `火焰图_会话提示v4{关闭}_独占/`：3932/3574样本，无质量warning；整请求user CPU4.73940→3.64142 ms（23.17%），HNSW3.83419→2.73972 ms（28.55%），仅是CPU诊断。`提示v4同二进制ABBA/` 的SciFact反向顺序CPU收益复现，D2近持平至小幅改善，但均无干净对称性能窗口，不写成正式PK。
- 参数寻优已验证实际融合SQL：SciFact低档 text88/vector94/ef180/batch88 为8880命中（80.0721%），人工nDCG=0.4830780、人工Recall=0.6603889；高档82/104/350/batch82为9054命中（81.6411%），人工nDCG=0.4962584、人工Recall=0.6570556。各自旧两档人工指标不退化，见最终 `提示v4等召回参数实查.json`。Milvus亦独立粗扫1989/细扫5865组（重叠不相加），9组真实RPC验证后选低档94/100/330、高档100/96/530；严格同最低双参考recall、各自旧人工指标不退化，不声称两端人工指标完全相等。
- 新 `同期v4参数.json` 用于 `四档同期复测.py --case-config`；SciFact两档各2次完整ABBA已完成，`同期v4质量与环境审计.json`验证质量与预期ID hash全过，但8个双端配对均受换页污染。**全窗口探索性汇总**：低档OG/MV P50=2.861/3.167 ms、QPS=338.43/305.15（OG P50低9.67%、QPS高10.91%）；高档3.999/3.472 ms、243.83/278.86（OG P50高15.18%、QPS低12.56%）。这不是正式验收，不替换旧表、不按速度挑轮、不无限重试；D2最终v4新版正式复测仍待补。
- 当前正式55440、Milvus、Oracle及私有55450均停止，数据/容器保留。下次须Release restart并显式load目标collection；不要默认容器启动即已load。私有 `.runtime/hnsw-buffer-hint.2gir3yCr/data` 只作功能QA（16MB buffer、2线程池），严禁拿来性能PK。下一产品改动尚未实施：先查预取重复内容锁，或补目标TID的minor fault/system CPU归因；原图仅采用户态，宿主机pgfault不能直接归给OG，细节见交接§0.25.3。目标仍未完成，不得因低档探索性领先标记完成。

### 6.11 HNSW空闲工作区已提交、同期验收继续（2026-10-13 23:33）

- 本轮重新启动正式55440，Milvus随后启动并完成高档同期复测，2026-10-14 00:03已正常停止；Oracle/私有55450停止。目标未完成，最新断点见交接§0.27。
- 新图/计数在 `验证/HNSW锁与内核_20261013/`：高档82/104/350/batch82全1109查询的目标线程约362 minor faults/请求、0.53～0.55 ms system CPU/请求；含内核CPU图缺页处理占9.39%，minor-fault事件图99.94%在HNSW临时对象首次触页。注意事件图不是CPU比例，minor fault不是swap/major fault。
- 96KiB原型已编译安装，672语义/新两档全部ID通过，但缺页仅降至337/请求、system CPU和延迟未改善，已否决并源码撤回。22:47统一Release incremental恢复完成，安装及实际进程二进制SHA256重新等于基线 `254c9725ef8be1d110d90d121e3601f686fe371656a28d38fc8cf20c8bafc7cc`，HEAD仍 `1fed7772a6`。失败原型 `72a48c9f…` 已不在运行。下一轮工作区原型状态以交接§0.26追加为准。
- 新增 `hnsw_scan_cache_size`（kB，0～16384，默认0），每会话至多一个逻辑清空的空闲工作区；本轮显式2048。借出前清槽并移到查询父上下文，正常结束ResetAndDeleteChildren后才可归还；活跃cursor不共享，额外块正常释放，不改变候选/ef/续扫/索引格式。详细MemoryTracking及非普通AllocSet回退原路径。
- 产品9文件已提交 `4c6383e5d5`，未push。头文件改动full日志 `release_full_20261013_225526.log`，安全回退增量 `release_incremental_20261013_231826.log`，该历史版本最终hash为`dbfe20b5c0fd3160ae826de65da519865942f90981a9877d466d9f9d808817e6`。`工作区v2*`证据：原生2/2、672语义/96 hash、24阶段生命周期、新两档1109条质量、2线程池32/2000维各300读/100写/100VACUUM及真实worker迁移通过；普通/特殊类型24组通过，PQ/mmap不可用不计通过。临时fixture清理，私有55450停止。
- `工作区v2四档ABBA/` 同二进制0→2048→2048→0、全窗口汇总：SciFact低/高档P50观测下降16.85%/18.55%，D2两档9.76%/9.42%；有主机换页，**仅为代码收益诊断**。高档minor fault361.63→0.34/请求，system CPU0.525→0.108ms。v1含内核图缺页处理占比9.37%→0.052%；最终v2开启图3662样本、缺页5样本/0.137%、无unknown/DSO-only叶节点，未见丢样warning。
- 正式同期用原 `四档同期复测.py --og-scan-cache-kb 2048 --case-config <验证/HNSW缓存预取_20261013/同期v4参数.json>`，仍20s预热/30s测量、整查询周期、零换页和干净对称轮；参数实际值写入environment。正式窗口内不编译/分析/运行其他shell任务，仅轮询该运行句柄，避免工具进程本身换页污染。目标仍未完成。

### 6.12 工作区同期结果与页内预取地址原型（2026-10-14 00:03）

- 交接先读§0.27。已提交`4c6383e5d5`完成SciFact高档2次完整ABBA：全窗口P50/P95/P99为OG3.398/4.556/5.166、MV3.563/4.582/5.088ms，QPS287.32/274.15；全部质量hash与人工指标通过，但4个配对均有换页，无正式验收，P99仍慢1.53%。证据 `验证/HNSW锁与内核_20261013/同期工作区v2高档审计.json`，低档/D2新版同期仍待补。
- 无查询40秒，9次major fault均来自编辑器/Codex，两数据库均0；这是空闲观察，不豁免正式计时环境门，也不擅停无关任务。Milvus已正常停止，正式55440运行、Oracle/私有55450停止。
- 当前仅`hnswutils.cpp`有新原型dirty：在既有buffer提示中保存每页两个页内位置，总量约64KiB。hit只在pin的页内形成CPU预取地址、绝不读取页字节，实际加载仍持锁重新定位；失效位置只影响预取效果。增量构建正在运行 `release_incremental_20261014_000143.log`，未提交/未验收；不得当成有效优化，先测后决定保留或撤回。后续仍需语义/并发/低缓存/特殊路径/原生门。

### 6.13 页内位置原型撤回、协议客户端已提交（2026-10-14 00:48）

- §6.12的原型已否决：锁占比下降但整请求无稳定收益且部分档变慢，未提交；00:17:52统一增量恢复完成，`release_incremental_20261014_001408.log`，运行及安装hash重新为`dbfe20b5…`。不再重复该64KiB位置提示方案。正式55440 PID728010运行；Milvus重启且仅D1加载，Oracle/私有55450停止。
- 新HEAD`21367bc089`只含`contrib/ogai/bench/libpq_bind.py`、17项只读边界测试及BASELINE§19，C++仍为4c6383e5d5，跟踪文件干净、未push。四档全部vector_send字节与结果ID验证通过。与同设置SQL EXECUTE图比较，CPU/请求3.37370→3.16244ms（6.26%观测），lexer/parser及vector_in消失；HNSW估算CPU相近，不是内核算法提速或正式Milvus胜出。
- **强制前提**：现有服务端session_timeout会覆盖Bind的语句定时器。客户端拒绝非零空闲计时器，调用方必须显式设置会话session_timeout=0和idle_in_transaction_session_timeout=0，保留statement_timeout=10s；17项测试确认超时和恢复有效。未修服务端定时器、未改持久GUC，不是任意生产配置的通用客户端。原先无约束的协议初筛数据不可作为最终配置结果。
- 正式入口新增`--og-protocol binary_bind --og-session-timeout 0 --og-scan-cache-kb 2048`，原默认不变；环境保存实际协议/超时/helper源码hash。采样参数为`--protocol binary_bind --session-timeout 0`。仍必须等双参考recall/人工不退化、完整周期、零换页/对称轮。
- 01:01追加：新协议SciFact高档完整ABBA已完成，`验证/HNSW锁与内核_20261013/同期协议绑定v1高档审计.json`。OG/MV P50/P95/P99=3.154/4.584/5.243与3.444/4.677/5.151ms，QPS304.61/279.29（OG高9.07%），P99仍高1.79%；质量/ID通过、两个配对均有换页，不是正式胜出。其余三档同版本待补，目标未完成；当前没有基准进程运行，两服务保持运行继续测试。

### 6.14 四档协议复测与同序内积原型（2026-10-14 01:34）

- `验证/HNSW锁与内核_20261013/同期协议绑定v1四档审计.json`：同一CPP4c6383e5d5/客户端21367bc089四档完整ABBA质量全部通过，8配对均有换页，仅为探索性；D2两档、SciFact低档所有指标领先，高档QPS高9.07%但P99慢1.79%。正式旧表不能被替换，不无限重跑污染窗口。
- `协议绑定_高档逐请求CPU{,编号}.json`：各6654次，慢尾主要是目标线程CPU执行，没有major fault，第二组前后均CPU19无观测迁移；`sched_schedstats=0`，不要把runqueue字段当完整等待证明。原生内积占完整图约27%，不是detoast问题。
- 新原型只改`src/common/backend/utils/adt/vector.cpp`：同8-lane、同水平归约、独立禁止FMA的AVX helper，其他维度/平台回退原路径。独立568000组逐位验证通过；禁止FMA作用到整个函数的早期版本dim4失败，未进入产品。此次不同于已否决的四累加器重排。
- HEAD仍21367bc089，原型未提交/未数据库验收，不能算有效优化。统一Release incremental日志`release_incremental_20261014_013413.log`正在运行，构建前hash dbfe20b5…；Milvus正在正常停止，Oracle/私有55450停止。基线`同序内积_修改前/results.json`已收齐；构建后先语义/ID/原生门，再对照性能及新图，无收益则撤回并重编。后续实际状态先看交接§0.29追加。

### 6.15 AVX/SSE边界清理原型（2026-10-14 01:53）

- 先读交接§0.30。§6.14手写同序内核已编译成e03577bf…并通过672语义/四档全ID/原生2/2/8形态24组；单端四档样本有改善，新图HnswSearchLayer自身样本305→39。但该原型未提交，收益不能全归给加载简化。
- 反汇编确认旧内积函数缺vzeroupper、HNSW接着执行传统SSE。`AVX_SSE边界独立对照.json`在传统SSE调用方下，旧核275～286ns、仅补清理132～141ns；每种候选568000组逐位一致。不是整请求2倍，见文档中的Intel原文与本机控制证据。
- 当前手写helper已移除，vector.cpp只比HEAD多5行：原循环后AVX条件下`_mm256_zeroupper()`，无计算顺序/FMA/尾部改写。HEAD21367bc089，未提交。`release_incremental_20261014_015303.log`构建中，完成后须重新验收该最小版，不能用手写版结果替代。Milvus/Oracle/私有55450已停止，原生25632已自动停止；正式实例由构建脚本管理。

### 6.16 最小AVX/SSE清理已提交，四档同期样本均领先（2026-10-14 02:26）

- 最新断点交接§0.31，HEAD`f17b450809`，只改vector.cpp与BASELINE§20，未push。5行补丁已统一Release增量成功，日志`release_incremental_20261014_015303.log`；安装/实际进程hash`fd233e19ff83fbcbd7f86533a6a87d97c82a1542984bbcd37e79c8f9ea7067aa`，banner仍1fed7772，不能混用旧hash。原手写helper未保留。
- `AVX边界_*`证据：672语义/96hash、四档全部ID和输入字节、原生2/2、8形态24组通过；PQ/mmap不可用不计通过，fixture清理。CPU图3882样本、unknown0、DSO-only1，无warning；整请求CPU观测3.16244→2.61597ms，HnswSearchLayer自身估算0.26397→0.02965ms。顺序时延诊断有换页，不是正式Milvus反超。
- 四档同期已完成：`验证/HNSW锁与内核_20261013/同期AVX边界v1四档审计.json`，原始数据`同期AVX边界v1/`。全部质量/ID通过，样本P50/P95/P99/QPS全领先；D2两档QPS高110.48%/91.80%，SciFact两档高73.41%/33.58%，高档P99低14.60%。**8配对均有换页，正式验收未完成，目标仍active**；完整表见报告§4.5.11/交接§0.31，不再引用旧版“高档P99慢1.79%”当当前差距。
- 02:26已核验正式55440/Milvus/Oracle/私有55450/原生25632全部停止，数据和基准索引保留。Milvus停前D2已load、D1已release，下次仍须实查。产品源码无未提交变更，`contrib/shark/smartmatch.pl`仅权限100644→100755 dirty保留。下一步获取安静对称窗口验收，不无限重跑污染窗口、不放宽门或擅停其他用户工作。

### 6.17 剩余验收条件为安静测试窗口（2026-10-14 02:37）

本节为旧双参考口径的历史断点。共同参考首轮见§6.19（当时SciFact99%仍落后）；最新§6.20已观察到新口径四档领先，但环境门仍未过，不能沿用旧口径结果作为新正式验收。

- 目标未完成，详见交接§0.32。最新提交/安装/质量证据未变，四档样本均领先，但16个measured窗口均不满足零换页。已独立核对每窗≥30秒、完整500/1109查询周期和两种顺序；不是计时或周期不完整。
- `验证/HNSW锁与内核_20261013/空闲换页归因_AVX边界停库后_20261014.json`：两库停止、约5.8GiB可用内存，40秒仍18次swap-in/16次major；进程major合计对应Codex13、编辑器3、观察脚本0。不能豁免后台换页或改成仅数据库计数，不能擅停无关用户进程或全局swapoff。
- 当前无活动测试/构建/监视句柄，所有本轮服务仍停止。已请求用户提供独占环境或其他工作结束时间；环境变化前不无限重跑，不盲目继续改产品。恢复时保留完整四档目标、原参数/质量/环境门、现有数据索引、新输出目录与正式验收标准。

### 6.18 SciFact 统一 95%/99% 的文本语义前提（2026-10-14）

本节保留只读诊断阶段事实；后续实现、共同参考及等召回复测已经完成，当前入口为§6.19。

- 用户要求 SciFact 与 D2 同档测试；旧 SciFact 80%/81% 是“双参考融合一致率”，**未完成共同参考下融合 95%/99% 要求**。两份 SciFact 参考只重合84.7160%，双参考最低值上界92.36%，不能靠增加ef解决，也不能把ANN单路99%当融合99%。详细入口为交接§0.33、总报告§4.5.12。
- 只读审计 `验证/SciFact统一召回口径_20261014/` 已确认：OG Jieba关键词提取过滤停用词/单字符后转小写；Milvus实际standard analyzer不同。OG `contrib/jieba_tokenizer/tokenizer.cpp:210` 的覆盖赋值导致大小写变体词频少计，影响4384/5183篇文档（47599次）及27/1109查询（29次）；不是HNSW优化引入。独立重算5908个现场BM25分数，最大相对误差约3.06e-7，OG仅两查询Top10同分截断差异；D2全部22000人工词分词一致。
- 仅修大小写累加的离线反事实，精确融合对Milvus参考一致率约84.72%→88.30%，仍未到95%，且人工融合Recall略降；**未改产品、未编译安装新产品、未重建索引、未替换GT，不算有效性能优化**。诊断程序已独立编译运行。
- 分词影响已存储的词频/文档长度/df，不能只改查询端或用旧BM25索引验证新语义。后续应先明确共同analyzer及同分规则、隔离文本索引验证、独立共同融合参考，再重定标两数据集；保持内部候选/ef各自寻优。HNSW不因BM25分词改变而需要重建；保护旧基准及中文语义。新口径可达性与性能尚未验证。
- OG有两篇COPY反斜杠转义差异（4634/5146），当前词项不变；统一新基准须核验全文字节，不能静默覆盖原数据。2026-10-14 08:21后本轮openGauss/Milvus已恢复停止，Oracle/私有实例未启动；产品HEAD/hash仍f17/fd233e19…，仅原`smartmatch.pl`权限dirty。旧环境门仍有效。

### 6.19 统一分词与共同参考四档复测（2026-10-14 10:05）

- 最新入口交接§0.34、总报告§4.5.13，纯分支HEAD `cad28864a7`（产品`dfe48f9706`，后续仅文档），新增索引级 `analyzer='standard_v1'`；默认Jieba不变。冻结Unicode16，UTF8限定，词项99字节上限报错，禁止已建索引SET/RESET，新magic阻止旧版本误用。不要同列混建不同analyzer指望规划器选择；仓内BASELINE§21说明完整边界。
- 首轮Release full于08:54完成，hash `81d2014b…`；原生3/3、28项新断言、旧四档全部ID/输入字节通过。SciFact全部6292条文本词频与Milvus一致，110900原生BM25分数重算通过，Top10一致100%。精确向量离线融合OG100%/MV99.99098%，不是原生HNSW 95%/99%或性能验收。
- 提交收尾只改生成头文件注释空白，第二次full `build_logs/release_full_20261014_090552.log` 已于09:16:53完成，最终安装/运行SHA256 `24ae5253a5719bcfb48e89192975b87392a75851062b1bd0fa051dfe93d121b4`。banner仍f17b4508/09:05:55，不代表缺少已编译的dfe提交。09:26最终版本3项原生回归输出与golden逐字节一致。原smartmatch.pl权限dirty保留、未push。
- 隔离QA库 `bm25_std_qa_20261014_085455`；融合表 `hybrid_bench_d1_standard_v1`、`hybrid_bench_d2_standard_v1` 为BM25 standard_v1与HNSW M16/efC200；新 `hybrid_bench_d1_standard_m64` 为M64/efC800，现用于SciFact PK。全文/原始float32已验证；D1保持正文EXTENDED/向量EXTERNAL，D2保持PLAIN。**不要重复创建**；旧基准表与HNSW未重建。证据根 `验证/SciFact统一召回口径_20261014/`，各fixture manifest已存；首次脚本失败空库 `_085414` 确认public无表/函数后已删除，诊断文件保留。
- `build_common_reference.py` 已完成 `common_reference_scifact_v1/` 和 `common_reference_d2_v1/`。最终Top10、参考预算100+100、RRF k60、数值ID同分、BM25及精确cosine用float64；候选/ef各自寻优，不强制同参数。SciFact全部1109查询；D2参考有1000查询，当前定标/历史对比用前500。新OG向量不再用6位小数TSV；Milvus两个旧collection全部正文/float32已回读核验相同。
- Milvus旧SciFact ef6000/32768均97.88097%，自身向量Top1有196/5183篇不命中；隔离新M16/200也有208篇。根因未证明为损坏或断图。另建 `hybrid_bench_d1_control_m32_efc400_20261014`（ef6000为98.9360%）及 `hybrid_bench_d1_control_m64_efc800_20261014`（已达99%以上），后者用于SciFact PK；三张控制collection均保留、勿重复创建。大k会触发Knowhere暴力回退，当前候选≤100、5183行单封存段，不满足该大k条件。
- 当前严格等召回配置为下表。OG的bm25_topk等于text候选；不是穷尽寻优。SciFact两端M64/800，D2两端M16/200；这是统一质量/索引配置工作，不能直接记成C++加速。

| 数据集/档位 | OG text/vector/ef | Milvus text/dense/ef | 双方共同参考命中 |
|---|---|---|---|
| SciFact 95% | 94/91/91 | 100/100/199 | 10558/11090 = 95.2029% |
| SciFact 99% | 100/100/350 | 100/100/1125 | 10981/11090 = 99.0171% |
| D2 95% | 60/70/70 | 91/70/70 | 4753/5000 = 95.06% |
| D2 99% | 100/100/106 | 100/100/144 | 4960/5000 = 99.20% |

- `matched_common_cases_m64.json`保存配对和原生ID源，`common_abba_m64_01/all_cases_audit.json`为四档最新完整测量；使用原ABBA脚本的`--common-reference-dir`、`--query-vector-source float32_npy`、指定QA库/表/collection模式。每档20秒预热、30秒测量、2个反序配对；16窗完整、所有ID/hash/整数命中均复现。
- **探索性、非正式：**8个配对全部换页。D2 95/99档QPS分别高96.15%/83.71%，SciFact95档高44.72%；**SciFact99档QPS低3.60%，P99高11.91%**。不能沿用旧80%/81%“四档全胜”，也不能宣布新正式通过。共同参考召回不是人工相关性召回；SciFact99档人工Recall为OG64.7889%/MV66.1222%，不等人工质量。
- 下一步对新SciFact99配置采目标线程CPU/minor fault与火焰图。其OG窗口宿主机pgfault约179万/6654请求，其他窗口约4～6万，仅为线索；先检验2048KiB keeper外的溢出块分配/释放，不能直接把宿主机缺页归因OG，也不要凭旧图改新代码。零swap-in/out/major-fault门不变，不无限重跑受污染窗口。
- 复跑：在本证据根执行 `python3 run_common_bench.py --config matched_common_cases_m64.json --out <新目录>`；先按§6.3显式启动正确Release和Milvus，脚本只load当档collection，末尾需手工停库。10:05本轮已恢复全部服务停止，无活动构建/测试；Milvus stop超时后exit137、OOM=false，数据卷保留，下次先查健康及索引加载。未停止无关用户进程，原smartmatch.pl权限dirty保留，代码/文档未push。

### 6.20 visited初始容量优化与共同参考四档新测量（2026-10-14 12:06）

- **当前产品`292303d8ea`，HEAD`90ff92a059`（后者仅文档），未push。** `hnswutils.cpp:InitVisited`只在磁盘查询且正reltuples小于原容量时缩小初始提示；visited仍自动扩容，未知/非正统计、插入和内存构建保留旧策略。SciFact M64/ef350数组512→64KiB；D2主档不触发该缩减。不是访问/候选上限，不修改距离/续扫/MVCC或页格式，不需重建基准索引。
- 统一Release/O2 incremental日志`release_incremental_20261014_113436.log`，11:38:18完成；该历史版本安装/实际进程hash为`a1ee51773854ebbb662a5d9428eee8ed95ca8296cf2284648619fdb5f3de298d`，不是§5.3的最新hash。原生4/4通过（新用例覆盖统计4行而实际256行时扩容、过滤、DELETE/VACUUM/TRUNCATE/未知统计）；672旧语义/96hash、新四档全量ID、32/2000维并发和快照续扫、8类回退×3查询均通过。PQ动态库未加载、mmap未启用，不计通过。
- 所有本轮证据在`公共文档/两路召回对比Milvus/验证/SciFact统一召回口径_20261014/`。新CPU图`sf99_m64_cache4096_cpu_after/cpu.svg`为4462样本、unknown0%、DSO-only0.0672%，无告警。固定4MiB、每端7763请求的前后诊断CPU/请求4.056→3.861ms（-4.83%），minor fault1001550→16、major均0；非perf两窗平均CPU下降4.31%。事件图与CPU图比例不可混淆，不把单端诊断当正式PK。
- `matched_common_cases_m64_workspace4096.json`是**当前复跑配置**：所有候选/ef/整数命中沿用§6.19，仅SciFact99%的会话`hnsw_scan_cache_size=4096`，其余三档2048，默认仍0。2/4/8MiB正反序诊断证明4MiB减少溢出分配、8MiB无进一步收益；代码与参数收益分开。
- `common_abba_visited_sizing_01/all_cases_audit.json`已完成四档/16窗/双向配对，原生/Bind ID与整数命中全复现。全部完整窗口的**探索性**QPS提升：D2 95/99档+129.73%/+95.75%，SciFact95/99档+56.39%/+14.58%；四档P50/P95/P99均观察领先，SciFact99%分别低12.84%/8.71%/7.01%。完整表见总报告§4.5.14/交接§0.35，不再把§6.19的SciFact99%落后写成最新差距。
- **正式环境未过：**15/16窗发生swap-in或major fault，完整干净配对0/8。不能把全部观测变化归因本次C++，尤其D2容量未变；也不代表人工召回95%/99%或严格等人工质量。停库后40秒`idle_paging_after_visited_sizing_01.json`仍有18次swap-in/15次major，进程快照对应Codex；只归因该空闲窗口，不反推全部计时污染或豁免门。
- 12:06正式55440/Milvus/Oracle/私有55450/原生25632均停止，无活动测试/构建；基准和控制数据全部保留。Milvus stop超时后exit137、OOM=false，下次检查健康并显式load；不删除容器。产品源码无未提交改动，原`contrib/shark/smartmatch.pl`权限dirty保留。
- 安静/独占环境就绪后，先按§6.3启动正确Release与Milvus，再在上述证据根运行`python3 run_common_bench.py --config matched_common_cases_m64_workspace4096.json --out <新目录>`，末尾手工停库。不要误用旧2MiB配置，不无限重跑污染窗口，不擅停其他用户工作或全局swapoff。若继续产品优化，先看新图中的内积、visited查重和邻接/缓冲访问；本轮已无依据把主瓶颈归为HNSW detoast。

### 6.21 AUTO计划验收与BM25 worker参数同步（2026-10-14 22:50收尾）

- **10月15日只读诊断更新：** 产品/二进制仍为下述版本，无新优化提交。新证据`验证/并行瓶颈分析_20261015/diagnosis_audit.json`确认每请求完整worker初始化、`StreamNodeGroup::deInit()`无条件末次sleep（1698次追踪均两次1000µs，阶段2.24～2.28ms）、会话级HNSW复用丢失、BM25 rank-only未传入Stream worker。详见交接§0.37；这是并行实现瓶颈，不应靠降低cost强行选并行。仅开线程池仍重建session，不承诺直接解决。08:28恢复OG停止，Milvus/Oracle未启动，39个本轮探针及临时挂载已清理；源码未改，不需重编。
- 用户最新要求以AUTO测试，并验证同一场景低代价对应低延迟/高QPS；不得把固定OFF的旧样本称为AUTO结果。代价不是毫秒，不要求数值精确成倒数。当前AUTO未枚举VECTOR，该模式只作诊断。
- 产品及HEAD为`b47f4fb9a5`，未push。原生FORCE发现BM25 worker忽略`bm25_topk/k1/b`；D2查询36有序ID不同，非默认评分时还改变集合。已在`guc.cpp`同步白名单补齐三项，未改成本系数/候选/ef/索引；Release/O2 incremental `release_incremental_20261014_151117.log`完成，hash见§5.3。banner仍f17b4508，不能据此否认新修复，也无需仅因提交Git再重编未改变的C++。
- 新原生`hybrid_bm25_worker_guc`旧二进制失败、新二进制通过，custom/generic及非默认参数24组有序ID比较；完整原生5/5通过。证据根`验证/SciFact统一召回口径_20261014/`，`auto_bm25_worker_before.json`/`auto_bm25_worker_after.json`为四档各60查询对照，修复后2160请求的同参数串并行ID全部一致；`auto_bm25_native_regression_audit.json`保存5项golden/actual hash。
- `auto_plan_validation_01/`四模式正反序已完成：预热10秒/测量30秒/整轮查询。`per_query_choice_audit.json`覆盖3218个查询档位组合、25744次custom/generic计划检查；AUTO全部选串行，每条查询跨周期和两轮各自的中位数均未发现并行更快。实际worker为OFF/AUTO/VECTOR/FORCE=0/0/1/2，无fallback；所有有序ID均复现。该控制阶段只有OG在线，不是Milvus PK。
- 控制实验32/32窗换页（swap-in5281/major3399）；仅观察到选对方向，不是全场景成本精确。模型不直接含ef/维数/M，VECTOR与FORCE共用cost，VECTOR planned字段仍标2但actual为1；不应把这些局限隐藏为“成本全面校准”。复跑控制用`validate_auto_plans.py --out <新目录> --seconds 30 --warmup 10 --rounds 2`，随后`audit_auto_plan_choices.py --run <控制目录> --out <新JSON>`。
- `common_abba_auto_01/all_cases_audit.json`为22:49完成的**最新同期AUTO四档表**，同候选/ef/工作区/共同参考，双方各自ID及整数命中全复现。D2 95/99档QPS观察高126.45%/108.57%，SciFact95/99档高51.70%/11.08%，四档P50/P95/P99也均观察领先。完整表在总报告§4.5.15、交接§0.36，不能再把12:04 OFF表写成最新AUTO结果。
- **正式环境未过：16/16窗换页，干净配对0/8**，swap-in5656/major2355/swap-out0。不能宣布正式胜出或把OFF/AUTO同物理策略的窗口涨跌算成此次补丁收益。共同参考复现率不是人工召回；不无限重试污染窗口，不擅停用户任务或全局swapoff。
- ABBA和`run_common_bench.py`新增`--og-parallel-mode`，CLI默认AUTO并保存/校验实际SHOW。当前复跑：按§6.3启动正确Release和Milvus，在证据根运行`python3 run_common_bench.py --config matched_common_cases_m64_workspace4096.json --og-parallel-mode auto --out <新目录>`。历史OFF需显式传off；底层OG类保留历史默认，主CLI显式传AUTO。
- 22:50正式OG55440/Milvus恢复停止，Oracle/私有55450/原生25632也停止，无活动构建或测试。Milvus stop超时后exit137、OOM=false，容器/数据/基准索引保留。产品及原生回归已本地提交，无未提交产品源码；原`smartmatch.pl`权限dirty保留。后续补非默认TAAT/max_score_ratio的worker同步/语义检查（本轮off/1.05默认），不声称所有BM25参数已覆盖；并行进一步优化先采worker火焰图，不拿旧HNSW图归因全部固定开销。

### 6.22 Worker生命周期、成本修订与最新性能断点（2026-10-15）

- **入口和代码：** 本节记录§0.38.7、总报告§4.5.17、BASELINE§24的阶段结果，后续核查见§6.23。4个.cpp修订、原生SQL/golden及BASELINE已提交`183eeda3de`：deInit末次条件sleep、轮询EOF时传递worker ERROR、池化Port无效socket、Hybrid根节点成本纳入ef/M/维数/池化与worker数。未push、未迁入集成分支，原smartmatch.pl权限dirty保留。
- **构建与原生门：** Release/O2增量日志`release_incremental_20261015_093851.log`，正式/实际测量进程/原生临时安装hash均为`f05b9fd863436c03ac80906a14bc00b19bd86bb17fff116550e2d80a5cab9dc4`。原生五项使用`bm25_standardcheck EXTRA_TESTS='hybrid_recall hybrid_bm25_worker_guc hnsw_buffer_hints hnsw_visited_sizing'`，日志`cost_model_native5_final.log`；普通hybrid_recallcheck只有4项。显式GCC PATH须加引号；LD_LIBRARY_PATH须包含GCC的gcc/lib64、isl/lib、mpc/lib、mpfr/lib、gmp/lib，否则libisl缺失。无头文件变更，不需full；未重建基准索引。
- **正确性和线程池：** 池ON/OFF最终版本各12872正常请求、20错误恢复、4连接400请求通过；ON另有80短超时/恢复通过。旧FORCE吞ERROR问题已修。非池化取消FATAL关连接在精确旧版400次中14次复现，仍未解决，OFF跳过取消门，不能宣称生产稳定性全部通过。576请求/1152任务确认2个Stream TID复用，但每任务仍重建session；全局线程池也影响普通连接，不是只针对Hybrid worker的开关。
- **成本验证：** 独立60查询、ef200/500、正反序：ef变化比例误差OFF20.18%→5.96%、ON23.30%→7.14%；同ef模式比误差OFF29.65%→25.58%、ON33.96%→15.48%。部分窗口换页，仅第一阶段修订，未覆盖跨数据集标尺、BM25词频/扩批、缓存/内存边界。过滤/多ANN/分区/特殊编码回退旧模型；底层独立ANN成本未一并修复。最终四个正式参数档3218查询×档位未发现并行中位数更快，AUTO全选串行，实际worker为0/0/1/2；VECTOR planned仍显示2是既有标注局限，不代表实际启动2个。
- **最新同期表不能写成全面领先：** `auto_pool_on_common_abba_01/`四档ON＋AUTO，QPS相对Milvus为D2 95/99 +112.52%/+73.17%、SciFact95 +22.55%、SciFact99 **−2.43%**。随后`auto_pool_off_scifact99_abba_01/`恢复OFF、同一新二进制AUTO：SciFact99 QPS +3.51%、P50低3.65%，但P95/P99慢3.58%/5.64%。两配置单列，不拼成统一胜出表。全部20窗换页、合格配对0/10，正式性能目标未完成。
- **证据和无效数据：** 根目录`公共文档/两路召回对比Milvus/验证/并行瓶颈分析_20261015/`。最终模式选择`final_four_modes_pool_on/choice_audit.json`；成本用`cost_model_pool_{off,on}_comparison.json`。池化有效独立集为`cost_model_pool_on_holdout_isolated/`，早一轮`cost_model_pool_on_holdout/`与正确性并发门重叠，整轮计时作废并有INVALID_TIMING.json，禁止引用。
- **收尾状态：** OG55440、Milvus、Oracle、私有55450、原生25632均停止。实例配置恢复初始SHA256=`8c83e997dd9e428e738aa6a9f2c74023ab0dd264806603771dc87a72b15bd7e7`，线程池默认OFF，产品默认未改。所有数据/索引/容器保留，自己的uprobes、debugfs/tracefs临时挂载清理。下一步先安静同版本对称窗口＋新火焰图定位SciFact99尾延迟，再补剩余成本模型；不要降低并行cost强行选慢计划，不无限重跑污染窗口。

### 6.23 精确新旧串行核查与最新SciFact99（2026-10-15收尾）

- **没有新产品补丁：** 运行产品仍`183eeda3de`，HEAD `7d49dcae18`仅记录文档。本轮临时复原旧4个.cpp、标准Release incremental，精确复现旧hash a74feea4…；随后恢复源码并重编回f05b9fd8…，均与原测量逐字节一致。没改数据/索引/候选/ef/召回。
- **预先固定旧→新→新→旧：** 每阶段重启、池OFF、AUTO实际串行0worker、20秒预热/30秒测量/完整查询周期；Milvus停止。SciFact99旧/新P99=5.525/5.590ms（新高1.18%），CPU/请求3.682/3.731ms；D2旧/新P99=2.212/2.210ms。全量冻结ID一致，8/8窗口仍host换页，目标线程major0；未复现约12%持续版本回退，但约1%差异未定论。
- **新旧图：** 13566/13405样本，unknown0%，HNSW占82.25%/82.02%，内积/visited热点接近，未采到额外规划热点/HNSW detoast。不把采样时延当PK，不把无样本当零调用。调度等待统计开关0，汇总显式null，不能解释为无等待。
- **最新单档同期：** 同一新版、池OFF＋AUTO，SciFact99共同参考10981/11090（99.0171%），双方有序ID hash全复现。OG P50/P95/P99=3.971/5.197/5.831ms，Milvus=4.553/5.657/6.214ms；OG低12.79%/8.12%/6.16%，QPS245.68/214.76（+14.40%）。4/4窗口换页、合格配对0/2，仍非正式胜出；保留上午慢样本，不与ON其他三档拼表。
- **证据根：** `公共文档/两路召回对比Milvus/验证/串行回退核查_20261015/`；关键为`protocol.json`、`revision_abba_01/audit.json`、`profile_pair_audit.json`和`milvus_same_new_01/all_cases_audit.json`。可执行快照`.runtime/hybrid-serial-abba-20261015.pTNEcsKK/{old,new}/bin`共用当前lib/share/etc软链，不是完整依赖备份。复跑前核验依赖，所有输出目录必须新建。
- **收尾及下一步：** 服务全停，配置hash仍8c83e997…、线程池默认OFF，无活动测试/构建/探针。Milvus stop退出134、OOM=false，下次启动检查恢复；数据保留。先稳定隔离窗口并追查跨窗口慢尾，不能将host换页直接断言为全部根因；不盲目回退正确性修复、不无限重跑污染PK。并行session重建/rank-only和成本跨场景标尺仍未完成。


### 6.24 长连接、事件等待与worker名次快路径（2026-10-15 12:27收尾）

- **代码及构建：** `4db9d8d4fd61f7c9d53508a08a006d64e8c0b35d`已本地提交，未push/未迁入集成分支。streamCore的deInit从1ms轮询改为同mutex/cond生命周期等待和末worker通知；streamMain在PortalStart后对Hybrid文本task调用worker局部BM25名次证明。保持原计数和真实清理等待；无过滤、直接Limit→IndexScan、typed NULL原始分数列才设skipScoreKeys，过滤/分数消费者/普通Stream保留原路径。头文件变化已Release **full**，日志`release_full_20261015_114648.log`，最终hash见§5.3；不要增量混编头文件。
- **长连接与继承边界：** 客户端原来已长连接，新审计在计时窗口外核对socket inode/连接对象/libpq启动标识/application_name，所有正反序、预热和完整查询周期零重连。池OFF前/后各8连接、57616/59007请求；池ON16连接、84924请求。**内部worker session仍逐任务重建**，不是已有线程池不生效；ON探针3个Stream TID执行1656次任务初始化。不要宣称已实现持久worker session或跨线程共享leader descriptor。
- **机制与正确性：** 原生5/5；池ON/OFF各12872正常、20错误恢复、4连接400请求通过，ON另80短超时恢复通过。普通SMP两种池配置各90正常＋10除零恢复，实际GATHER/REDISTRIBUTE。OFF/ON的deInit各666/828请求内sleep均0，均值0.192/0.123ms；历史条件轮询1.114ms仅诊断对照。BM25池ON/OFF无过滤/过滤各120请求，重复分数表构建分别0/520次，ID相同。函数被内联，计数来自RelationGetIndexScan＋bm25gettuple_internal调用栈识别对应点。
- **性能边界：** 新旧池OFF FORCE P50观察降低8.79%～16.15%，32窗仅3窗干净且非版本随机化对照，不能作为严格净收益。首轮D2串行变慢已保留，追加精确旧→新→新→旧：SciFact99平均时延+0.57%、D2 99 −5.19%，8窗污染，未重现持续7.5%D2回退也未严格证明无回退。池ON四模式32窗8窗干净；3218查询×档位逐查询中位数未发现并行更快，保存计划实际worker0/0/1/2，支持AUTO串行，不调低cost强行并行。本轮没有新Milvus PK。
- **未解决：** 该轮**线程池OFF**的FORCE CPU图13054样本，互斥归因初始化39.44%、清理10.20%、HNSW29.91%、BM25 2.34%；不是墙钟比例，不能用于解释ON，ON新证据见§6.25。下一步安全复用session/只读初始化状态，先定义权限/GUC/快照/资源回收/失效与内存上限。非池化取消160次中5次FORCE断连接、无实例重启，旧版本已复现14/400，不能宣称取消门/生产稳定性通过。
- **证据与收尾：** 根`公共文档/两路召回对比Milvus/验证/并行事件等待与快路径_20261015/`，重点`before_after_pool_off_audit.json`、`serial_revision_abba/audit.json`、`after_pool_on_four_modes/choice_audit.json`、`force_cpu_pool_off/attributed_cpu.svg`及`completion.json`。OG已fast停止，其他服务未留运行；配置hash恢复8c83e997…、线程池默认OFF，仅原smartmatch.pl权限dirty。5个本轮探针已移除，原有`/sys/kernel/tracing`挂载保留；数据/索引/容器不删除。

### 6.25 线程池ON分阶段诊断（2026-10-15，15:04补齐串行）

- **串行补表：** 总报告§4.5.20新增同e954、池ON、长连接AUTO实际0worker的四档数据，总计同为服务端Bind→Sync均值，SciFact95/99=2.193/3.978ms，D2 95/99=1.069/1.400ms。原并行行未改；串行执行区间是leader ExecutorRun含融合，不叫worker初始化。17045追踪请求、41635含预热/控制请求ID均一致；12窗5窗干净，属于稍后补测诊断，不是随机化净收益。`验证/线程池ON分阶段诊断_20261015/serial_comparison_audit.json`及`serial_completion.json`为新增入口；15:04:06再次fast停库、32探针移除、配置恢复，产品源码未改。
- **口径纠正：** 用户质疑OFF数据解释ON，本轮专门补齐ON＋长连接四档CPU/时间线。产品与安装仍4db9d8d4fd/e954f9db…，无新产品改动/编译、无Milvus PK。证据`验证/线程池ON分阶段诊断_20261015/diagnosis_audit.json`；详细表在交接§0.41、总报告§4.5.20。
- **真实复用与质量：** 9组有效采集47507请求（含预热），持久连接零重连、冻结有序ID全一致；FORCE计划实际2worker/无fallback。四档追踪7827请求/15654任务仅3个TPLstream TID，客户端跨8个TPLworker TID；必须逐任务配对InitStream/CleanUp，不能用旧固定leader/单任务TID分析器。线程复用不等于session/工作区复用。
- **ON热点：** SciFact95/99初始化占CPU31.34%/23.96%，HNSW38.75%/51.74%；D2 95/99初始化45.98%/43.11%，HNSW18.22%/19.70%。单worker初始化约1.25～1.46ms，GUC默认初始化＋配置恢复约占三分之二；线程级local syscache部分已经复用，不应宣称全部缓存重建。deInit约0.104～0.126ms已非大头。CPU占比不是墙钟，两个worker时间不能相加。
- **下一最小候选未实施：** `guc.cpp:add_guc_variable`每次追加变量全表qsort，ON配置恢复子树约47%～48%样本在排序；先验证同比较器有序插入，保留hooks/占位变量/来源语义。另研究有上限的HNSW空存储复用：SciFact99 FORCE minor fault约915～920/请求，AUTO约2.1～2.2；session工作区随任务释放。minor fault不是磁盘I/O/TOAST，不得绕过session-id/权限/错误回收安全门。改代码后按§5编译及回归，再测收益，不提前压低cost。
- **采集陷阱：** 首次`SciFact_95_force_trace`的HNSW包装入口被ANN bypass绕过，角色门失败已标记，正式诊断用`_v2`实际内部入口；探针清单`probe_manifest_v2.json`只适用核验的e954二进制。CPU、时间戳、无探针控制分开，无并发测试；全部输出目录必须新建。27个非预热窗10个干净、目标进程major0，但host仍换页，不能作正式PK。
- **收尾：** OG55440于14:39:26 fast停止，其他服务未启动；原配置hash精确恢复8c83e997…、线程池默认OFF，32个自有探针已清理，预存tracefs保留。产品源码无新增dirty，原smartmatch.pl权限与旧构建资产保留，数据/索引/容器未删。`completion.json`保存核验；非池化取消旧风险仍未解决。

### 6.26 池化HNSW复用与执行区间验收（2026-10-15 17:57收尾）

- **代码/构建：** `32d36d041f`已本地提交6个产品文件＋BASELINE，未push/未迁入集成分支，只有原smartmatch.pl权限dirty。knl线程context新增懒指针，未增加独立THR_LOCAL/未抬门槛；头文件Release full `release_full_20261015_165652.log`，接收端.cpp增量 `release_incremental_20261015_173123.log`，最终hash见§5.3。没有改成本/候选/ef/索引格式或重建基准索引。
- **复用边界：** 仅非迁移THREADPOOL_STREAM保留有界空工作区（现有GUC、每线程最多16MiB）及约32KiB的buffer编号提示；普通/可迁移session沿用旧路径。不重挂session上下文、不改session_id，不缓存活跃扫描、权限、快照或pin。query reset callback负责错误/取消，换父session/尺寸/详细跟踪时丢弃，嵌套回退；原buffer完整tag/BM_VALID和资源记账保留。不是完整worker session复用。cache_size=0仅禁用工作区，不禁用独立编号提示。
- **等待修正：** 两路活跃仍公平轮询，一路真实EOF后对剩余一路使用既有阻塞Stream接收，保留rank/cap/EOF/错误/取消。99档HNSW worker执行内发送等待SciFact/D2观察0.640/0.671→0.214/0.348ms；这是执行区间子集，不得重复相加或归入初始化。
- **验证：** 原生5/5；四模式12872有序ID、4连接400请求、20错误恢复/80超时尝试通过；工作区65阶段含12错误/40取消通过。私有fixture 376次完整ID/分数比较覆盖更新/VACUUM/REINDEX/TRUNCATE/换索引和四种空支路，自建schema清理，基准对象未动。非池化取消旧风险、全面OOM、PQ/mmap和长期稳定性仍不宣称通过。
- **最新阶段数值：** 同一5e5、池ON、长连接，SciFact99串行/并行执行4.036/4.313ms，D2 99为1.240/1.449ms，并行仍慢6.85%/16.81%；Bind→Sync为4.307/6.559ms、1.446/3.474ms。CPU图确认HNSW缺页已小，但图搜索层并行仍约慢10%，具体工作量/缓存/调度根因未闭环。最终Top10/质量与ID不变；没新Milvus PK，不能称已达并行更快。
- **无探针对照和局限：** 旧/新FORCE均值SciFact9.244→6.713、D2 4.355→3.563ms（观察降27.38%/18.17%）；AUTO4.449→4.426、1.551→1.612ms，D2观察慢3.96%未排除回退。新16窗仅1干净，追踪相关12窗5干净，非随机化版本ABBA；只报观察，不无限重试污染窗口或擅停用户任务。
- **入口/下一步：** `验证/worker执行复用对照_20261015/{final_audit.json,work_state.json,run_final_diagnostics.sh}`；最终为`*_drain`，`*_final`是中间33d17工作区版本。先验证16行批次是否适合阻塞融合，同时看初始化/最终消费/端到端，不能挪等待到ExecutorRun之外造阶段胜出；再核对HNSW展开/距离次数与缓存/调度。GUC重复排序是另一个初始化课题，不用它解释执行内部。
- **采集/收尾：** 探针需按新ELF解析，GetScanItems静态入口未命中不能记零，实际层搜索用HnswSearchLayer。其他TPLworker简单SQL按Bind/Sync真实TID排除归属，原trace/全部时延保留。OG已fast停止，其他相关服务均停，配置hash恢复8c83e997…、默认线程池OFF，36个自有probe移除，预存tracefs保留。旧e954只能配`.runtime/hybrid-worker-reuse-20261015.MHBn2xyY/baseline_full_install`完整快照，不能混用本轮改变头布局后的lib；数据/日志/快照均保留。

### 6.27 有界交付批次与完整执行计时（2026-10-16 09:05收尾）

- **代码/构建：** `099656b971`本地提交2个.cpp＋BASELINE，未push/未迁入集成分支；仅原smartmatch.pl权限dirty。无头文件改动，Release incremental `release_incremental_20261016_084342.log`于08:47:31完成，最终安装/进程hash见§5.3。实验参数钩子已移除，不需要仅因Git提交再编译。
- **实现：** `execStream.cpp:buildLocalStreamContext`仅对task Stream直接子节点的正值INT8常量LIMIT取`min(limit,100)`，未知任务保留16、普通Stream仍100。大于100仍分批，尾部保留EOF刷新，接收容量既有100；候选/ef/rank/元组所有权/错误取消/成本/索引不变。100候选时BM25和HNSW发送配额等待均0，不能写成所有同步无开销。
- **当前结果：** 同最终ELF、池ON长连接，完整执行串行/并行为SciFact99 4.164/4.328ms（慢3.94%）、D2 99为1.190/1.111ms（观察快6.68%）。完整执行含最后leader接收/融合/输出，旧§6.26仅worker包络的数字不可直接同列。无探针AUTO/FORCE均值4.556/6.898、1.477/2.989ms，端到端均串行更快，不改cost强推并行，无新Milvus PK。
- **验证/局限：** 原生5/5；四模式12872有序ID、并发400、错误20、超时80尝试/恢复通过；候选1～400与有无过滤的200组完整ID/payload/RRF/行数、376组更新/失效/空支路、普通实际Stream90查询＋10错误恢复通过。基准索引未重建，只清理自建fixture。最终追踪12窗仅1干净、无探针8窗仅2干净，全部保留，不称正式收益/SLA；非池化取消旧风险、全面OOM/多平台/长期稳定性未验收。
- **证据/复跑：** `验证/交付批次对照_20261016/{final_audit.json,experiment_audit.json,work_state.json,completion.json}`；`run_final.sh`先过门，`run_final_diagnostics.sh`检查门再采集。`experiment_*`是临时76322ce7 ELF的16/32/64/100同连接实验，`final_*`才是0b2；复跑使用新输出目录并重新解析ELF探针。`complete_execution.py`补leader尾部；`analyze_trace.py`单列后台事件后按Bind/Sync真实TID归属，原trace和全部客户端时延保留。首轮边界脚本误设bm25_topk=1的失败及最后实验AUTO的分析失败均保留，不计通过。
- **下一步：** SciFact图搜索层墙钟3.541/3.968ms，独立HNSW CPU估算3.644/4.316ms/请求。先核对worker有效ef与逐查询展开/距离/页访问次数，再判断同工作量下的缓存/调度；相同ID不证明工作量相同，不能预先归因NUMA/频率。GUC重复排序属于另一个初始化优化。固定16交付问题已处理，不再重复同一试验。
- **资源/恢复点：** OG已fast停止，55440/55450/25632/Milvus/Oracle均停，配置hash精确恢复8c83e997…、线程池默认OFF；36自有探针清理、原tracefs保留。`.runtime/hybrid-stream-batch-20261016.09pD6W/before/bin`只有旧5e5三个执行文件，lib/share链接当前安装，不是完整快照；本轮头布局未变可配当前lib，旧e954跨上轮布局仍须§6.26完整快照。用户dirty、数据、索引和全部失败证据保留。

### 6.28 HNSW工作量与反向单worker诊断（2026-10-16）

- **当前结论：** 见交接§0.44、总报告§4.5.23、BASELINE§29。没有新产品优化或Milvus PK；功能代码仍099656b971。两个.cpp的反向单worker原型59c3412b已撤回，Release incremental `release_incremental_20261016_100531.log`于10:09:25完成，安装/实际进程精确恢复0b2 hash。不要自动重施实验patch或改cost强推并行。
- **工作量已核对：** 0b2产品、SciFact1109查询×AUTO/FORCE×正反序=4436请求；底层ef350/上层8/M64、3层搜索和逐查询展开一致。平均展开359.970243，距离入口4028.412985/4028.412083；约893万次/模式中有2次孤立串行多计，保留而不称逐位全相等，不能解释worker慢。仅按HnswSearchLayer归属；leader生命周期内的BM25/SQL、2218个只初始化未搜索的scan另算。两数据集各32查询也核对预取/pin/ReadBuffer，冷hint与热态分开，重探针计时不作性能。
- **拒绝原型的依据：** 同一原型、池ON、每数据集同一长连接切换四计划，正反序/预热测量各至少10秒/完整查询周期；AUTO实际0、FORCE2、VECTOR1、反向BM25 worker1，结果不变。完整执行AUTO/FORCE/反向：SF4.333/4.502/4.278ms，D2 1.242/1.112/1.824ms；无探针均值4.919/6.857/5.111、1.631/3.272/2.447ms。SF反向阶段只观察快1.27%，端到端慢3.92%；D2端到端慢50.10%。不能因比FORCE快约25%就合入。独立连接对照同方向；两组各1/16无探针窗、1/12trace窗、1/3CPU窗干净，未通过正式环境门。
- **根因范围：** SF串行HNSW/BM25估算3.990/0.283ms CPU，请求未被均分为两份。反向方案文本执行与HNSW重叠0.442ms，但图搜索多0.304ms，省下0.055ms完整执行，准备/收尾却多0.336ms。SF图搜索AUTO/FORCE/反向3.719/4.142/4.022ms，guest可见off-CPU仅0.053/0.894/0.152微秒；硬件PMU不支持，不能声称已证明缓存/频率/NUMA或排除所有host-vCPU调度。剩余热点为预取、pin/内容锁、visited/堆及内积，不是又发生TOAST解压。
- **测量陷阱：** WSL sched payload kernel PID与perf样本TID不同，要从同一switch-out事件验证映射；原始负后台TID也合法。首次解析失败和pilot保留，完整对照另起目录。单worker的旧leader_tail_ms还含leader继续HNSW，不是纯融合尾部，汇总叫leader_after_workers_ms。CPU和墙钟、重叠阶段不相加。
- **证据/复跑：** `验证/HNSW工作量核对_20261016/{orientation_audit.json,work_state.json,completion.json}`；全量计数full_scifact_counts，抽样用base_counts/graph_scoped，CPU在orientation_diagnostics，正式完整诊断trace在orientation_trace_same_client，无探针在orientation_same_client和orientation_crossover。脚本count_work.py/branch_orientation.py/run_orientation_diagnostics.py/summarize_orientation.py；新输出目录、重新解析ELF偏移，保留失败记录。原型ELF位于`.runtime/hybrid-leader-vector-20261016.xsw496/gaussdb_orientation`，patch在证据根；同目录0b2三个二进制不是完整安装快照。证据/测试脚本在源码Git外。
- **下一步：** 分别验证add_guc_variable对已排序数组重复qsort的初始化成本与HNSW预取/pin/内容锁热路径；前者不能改称图搜索提速。短支路提早清理须先证明Stream上下文/错误/取消所有权，不跳过quitSyncPoint，不省GUC继承，不保留脏session。恢复后的验证、停止与配置还原由completion.json核验；本轮没有重建基准索引或改变默认线程池设置。
- **最终恢复：** 恢复版原生5/5、四模式12872有序ID、并发400、20次运行期worker错误恢复、80次短超时尝试/恢复均通过。仅BASELINE§29提交e90623b659，未push，原smartmatch.pl权限dirty保留。10:16 fast停库，55440/55450/25632/19530/1521无监听、Milvus/Oracle停；配置hash精确恢复8c83e997…、默认池OFF，自有探针已清理、原tracefs保留。completion.json已完成实际核验。

### 6.29 同虚拟CPU控制与底层搜索定位（2026-10-16）

- **状态：** 产品仍099656b971/0b2，只有BASELINE§30文档提交5bde98afc0，未push；无新代码/编译/成本/索引或Milvus PK。详细见交接§0.45、总报告§4.5.24；证据`验证/HNSW执行位置对照_20261016/diagnosis_audit.json`，恢复见completion.json。
- **实测：** SciFact99池ON、每组长连接、冻结ID。双worker与串行按同查询/同guest CPU配对4436次，图搜索3.241/3.482ms，worker慢7.46%。原VECTOR单HNSW worker与串行均固定CPU2，等窗口3.376/3.721ms（慢10.20%）；底层3.316/3.651ms，贡献0.344ms差额的97.06%。不能仅归给两个worker竞争、静态CPU编号、上层下降或初始化计时。
- **边界：** CPU切换状态914个query/CPU观察配对差0.052ms，非缓存因果证据。新独立火焰图HNSW CPU3.562/3.782ms，差额分散在内积、visited、邻接、预取/pin/锁；不是已证明锁争用。PMU不可用，同guest CPU不保证频率/物理核/宿主机排他。默认亲和性完整执行正序4.068/3.838、反序3.787/3.893ms，方向不同；Bind→Sync等窗4.179/5.959ms，不能强推AUTO。
- **验收与复跑：** 14追踪窗36597请求、4CPU窗；含各阶段控制共133080次ID一致（不含initial warmup），实际0/2/1worker及Hybrid Recall/ANN计划核验。追踪5/14、CPU0/4过环境门，无干净对称组合。保留请求加权summary和等窗口汇总，不挑快窗。placement_control.py/single_worker_control.py/capture_fixed_cpu.py均新目录、finally恢复亲和性；CPU采样前移除时间戳探针。
- **下一步/收尾：** 底层热点做最小同质量对照；GUC重复排序仍是独立初始化课题。全页预取/页内位置提示已失败，不重复、不开无锁页访问、不默认绑核。11:14:25 fast停库，相关端口无监听、Milvus/Oracle未启动，原亲和性/8c83e997配置hash精确恢复、默认池OFF、自有探针清空、原tracefs保留，原smartmatch.pl权限dirty和数据保留。

### 6.30 独立收敛预热后的串并行三轮复测（2026-10-16）

- **入口/版本：** 交接§0.46、总报告§4.5.25；`验证/独立预热稳态对照_20261016/run_v2/summary.json`。HEAD/产品/hash仍5bde98afc0/099656b971/0b2，无产品改动/编译/新提交或Milvus PK。线程池ON、默认亲和性，每数据集AUTO/FORCE独立长连接；两数据集共同参考99%配置不变，真实0/2worker及Hybrid Recall/ANN计划验证。
- **固定规则：** 每个无探针客户端窗、执行trace窗各自预热≥20秒；最近5完整周期均值相对极差≤3%、P95≤5%，连续两次通过，上限120秒（完整周期边界结束）。测量≥30秒且≥5周期，前后半段均值漂移≤3%；host换页/major及目标major均需0。固定3轮，不按速度挑窗、不无限重跑；预热收敛不能替代环境门。只启用4类边界事件，串行/并行4/8次命中/请求，完整执行含最后接收/融合/输出；其他TPLworker内部SQL按真实Bind TID排除归属。
- **观察结果：** SciFact完整执行串行/双worker3.709/3.776ms，慢1.80%（三轮1.59%～1.93%）；D2为1.101/1.015ms，快7.85%（6.65%～8.72%）。无探针端到端分别4.184/6.006、1.475/3.007ms，均串行更快，不改cost强推并行。三轮等权，不混CPU与墙钟，不与旧36探针窗口直接算代码收益。
- **验收边界：** 24次预热全收敛（23.65～95.57秒）；客户端计时23/24窗满足漂移门，纯执行12/12满足；但环境仅1/24干净、0完整合格配对，正式接受结果为null。含预热698959次ID一致，测量263348请求（trace130924、无探针132424），不是生产或正式性能验收。保留run_v1内部SQL归属断言失败及修正后的首窗，不混入run_v2；脚本单测7/7通过。
- **复跑/收尾：** `run_steady.py --out <新目录> --rounds 3 --seconds 30`，之后`summarize.py --run <目录>`；须先按§6.3启动正确Release并临时开池，结束手工fast停库/恢复原配置。12:22:23已停库，配置SHA精确恢复8c83e997…、默认池OFF；6个自有事件已移除、原tracefs保留，未改亲和性。Milvus/Oracle未启动，数据/索引、smartmatch.pl权限dirty保留；完成核验见completion.json。下一步仍分别验证初始化GUC排序与HNSW底层，正式复测先解决环境干扰。

### 6.31 HNSW底层差额慢路径诊断（2026-10-16 14:37收尾）

- **入口/状态：** 交接§0.47、总报告§4.5.26；`验证/HNSW底层差额诊断_20261016/diagnosis_audit.json`。HEAD/产品/hash仍5bde98afc0/099656b971/0b2，无产品代码、GUC初始化优化、成本、索引、新编译/产品回归或Milvus PK；最终微架构根因未查实。
- **新证据：** SciFact99池ON，固定16查询×AUTO/VECTOR/FORCE×两轮=96次，真实0/1/2worker、全部ID一致。每请求底层展开350、内积4016.5、pin4016.5、首次pin4011.9375、共享锁获取8033均一致；实际机器分支计数无底层CAS重试、锁忙/排队、buffer头等待/usage重试或ReadBuffer回退。向量地址对齐/非共享分配器相同。保留2个分配入口/32字节及各1个预取入口差异，不能称所有计数逐位一致。计数是重探针，耗时不是性能；火焰图锁CPU不等于争锁。
- **测量边界：** 完整reverse_v1纯底层：连续串行3.342、间隔串行3.325、分核worker3.431、同核worker3.452ms；同核未消除差额，请求间隔影响未在反向复现，不能认定频率/cache原因。11个测量窗66540请求含失效位置窗，仅1窗干净、无合格配对。原整套两轮因新线程守卫中止，不能声称完整通过；相邻配对paired_v1的worker预热122.70秒仍未收敛，没有测量结果。不无限重跑或放宽门。
- **探针/线程陷阱：** 新Stream会继承创建者的临时CPU绑定；所有实际层入口/出口需核验，finally也要恢复中途新增线程。NewPrivateRefCountEntry、WaitBufHdrUnlocked在PinBuffer里内联，独立符号零计数不证明没执行；v3按SHA/指令字节定位分支。path_counts_v2在LOCK前缀指令激活uprobe报errno524、空perf，作废；v3用同分支后继非原子指令，解码无warning。6项脚本边界单测通过，失败原始资料均保留。
- **后续/收尾：** 下一步低扰动底层分段和同线程计算/访存校准，或真实CPU对应的硬件事件；不能把缓存/TLB/频率猜测写成根因，不跳过pin/锁，不以GUC准备收益替代执行收益。execution_controls.py/count_paths.py/paired_requests.py各用新目录，正确Release临时开池后运行。14:37 fast停OG，配置8c83e997…/默认池OFF、亲和性均恢复，两组自有probe清理、原tracefs保留；Milvus/Oracle未启动、用户dirty与数据/索引保留，无源码仓新提交。恢复核验见completion.json。

### 6.32 HNSW原生分段诊断：首次页访问差额（2026-10-16 15:51:58收尾）

- **入口/版本：** 交接§0.48、总报告§4.5.27；`验证/HNSW原生分段诊断_20261016/{diagnosis_audit.json,completion.json}`。127行临时.cpp插桩Release诊断版为12cd30da…，已撤回并Release重编恢复0b2。HEAD/产品仍5bde98afc0/099656b971，无新产品优化、成本/GUC初始化改动、索引重建、源码仓提交或Milvus PK。
- **定位结果：** SciFact99池ON、独立长连接/收敛预热、HNSW固定guest CPU2。1/128抽样的首次读/pin、首个共享内容锁、页内定位/预取提示合计串行236.64/worker281.11ns（+18.79%），实际含访存内积318.59/325.21ns（+2.08%）；纯底层每请求3.202216/3.392063ms（+5.93%）。每阶段是每次抽样调用ns，不能混每请求ms。首次三段外推增量约0.175ms只作数量级，不说精确92%贡献或可全部消除。cache/TLB/频率最终原因未证明，不能从锁CPU判争锁。
- **控制和边界：** 原生每32/128操作采时一次，每请求结束后1次汇总uprobe；sample0仍有诊断布局，非原0b2。单独轮换率/位置控制1/128开销观察为串行0.284%/worker0.453%，非精确校正。7有效窗47687请求、测量漂移过门但环境干净0/7；无完整平衡ABBA，不能发布正式收益。含排除CPU窗及开销控制73194测量请求，含预热/pilot186888次ID一致，恢复版另外12872次通过。逐查询工作量一致；全1109条展开1084条350、24条351、1条352，ef并非精确展开次数。
- **复跑陷阱：** 当前0b2没有诊断emit符号，run_segments.py须先apply_patch应用归档v2补丁、按统一Release构建并核验hash，再用新目录，不能套用诊断ELF偏移。v1漏Mmap包装→Disk回退计时无效；v2错误固定展开350，按既有全量基线重解析首窗；v3新Stream继承CPU6、5545条中2634位置错误，整窗排除；v4续原定ordinal4，未挑快窗补跑。全部失败保留，finally恢复新增线程绑定。诊断ELF快照仅供解码，不是完整安装。
- **下一步/收尾：** 优先验证首次descriptor/页头触碰，可研究有界前瞻提示，但仍正常pin、完整tag验证、内容锁/失效回退，先正确性后无插桩同质量对照；不重复失败的整页预取/tuple偏移缓存、不绕锁、不以GUC准备收益代替HNSW执行收益。恢复日志release_incremental_20261016_154148.log；原生5/5、四档四模式12872 ID、脚本7/7通过。15:51:58 fast停OG，配置SHA精确恢复8c83e997…/默认池OFF，亲和性/自有probe清理、原tracefs保留；Milvus/Oracle未启动、各端口无监听，原smartmatch.pl权限dirty及基准数据/索引保留。

### 6.33 首次触碰干预：worker底层约8%改善候选，尚未产品化（2026-10-16）

- **入口/进展：** 交接§0.49、总报告§4.5.28；`验证/HNSW首次触碰因果对照_20261016/run_v1/causal_audit.json`。同一临时Release诊断版中，worker纯lc0底层原路径→实际预取正序3.376→3.123ms（降7.49%）、反序3.410→3.137ms（降8.02%），六遍方向一致；等权减少0.263ms/7.76%。相同地址准备但不prefetch控制无总收益。尚非产品优化，无新Milvus PK或源码仓提交，HEAD/产品仍5bde98afc0/099656b971。
- **机制/约束：** 前瞻8个候选，对已知普通shared-buffer槽位的descriptor两cache line、不可变content_lock、页头一cache line发4条read-prefetch；既有4页pin窗口不变，pin前不读页内容。实际访问仍完整pin/tag/BM_VALID/内容锁/回退，不是整页预取、磁盘预读、页内tuple偏移缓存或绕锁。1/128抽样首锁78.15→34.68ns、页内定位/预取提示141.32→36.45ns，实际内积322.05→329.62ns未变快；地址准备在局部分段外、总时间内，不能求和分段节省当净收益。
- **控制与边界：** SciFact共同参考99.0171%、1109查询、100/100/350/4096KiB、M64/800，池ON长连接，HNSW guest CPU2/leader6/client8。固定AUTO/VECTOR/VECTOR/AUTO，三条件独立预热，worker6次过；两个AUTO原路径超120秒未收敛，整块跳过，**本轮无串行测量**。39924测量、含预热190844次有序ID全一致，工作量/CPU核验通过。worker两块host换页/major均非0，环境0/2合格；正序中间遍漂移，原路径/预取三遍极差为3.33%/3.94%，三遍干预不等于五独立稳态周期。不能用本轮worker3.130对比旧串行3.202，不能称原0.190ms差额解决或正式8%产品提速。
- **版本/复跑：** 170行临时hnswutils.cpp补丁及诊断ELF2e00dbc7…已归档，源码已撤回；当前0b2无诊断emit，不能直接跑run_causal.py或复用2e00偏移。重做须先核对HEAD/dirty、应用归档temporary_native_instrumentation_touch1.patch、统一Release重编核验、临时开池、使用新目录；v[27]低2位variant/上位hint计数，不能按旧padding解码。ELF快照仅解码用途。原始预热失败和环境污染记录必须保留，不无限重跑。
- **下一步/收尾：** 先补同改动串行对照；候选去计时后做并发回收/槽位复用/失效与多形态语义，再验无插桩完整执行/端到端，不改cost强推并行。恢复日志release_incremental_20261016_164220.log，16:46:13恢复版安装/运行回到0b2完整hash，原生5/5、12872次ID回归全通过；16:53 fast停库，配置精确恢复8c83e997…/默认池OFF。hnsw_touch16探针及既存/新增线程临时绑定清理，原tracefs保留；Milvus/Oracle未启动、各目标端口无监听。原smartmatch.pl权限dirty、基准数据/索引保留；脚本6/6、恢复及独立收尾见restored_validation/summary.json和completion.json。

### 6.34 同预取串并行对照：worker仍观察慢3.83%（2026-10-16）

- **入口/数字：** 交接§0.50、总报告§4.5.29；`验证/HNSW首次触碰因果对照_20261016/serial_v1/comparison_audit.json`。双方均预取，纯lc0串行/worker正向3.070/3.215ms、反向3.077/3.167ms，等窗口3.073/3.191ms，worker仍慢0.118ms/3.83%，不是领先。原路径仅反向一组3.235/3.527ms（慢9.01%），缺正向，不能与预取双组直接算正式差额缩小比例。
- **方法/门：** 同2e00诊断ELF，原170行补丁无新增原生改动；SciFact99.0171%、1109查询、100/100/350/4096KiB/M64/800、池ON、HNSW guestCPU2/leader6/client8。固定S2/W2/S0/W0/W0/S0/W2/S2，每条件独立过原预热门，6/8通过，原路径正向两窗超120秒跳过。每测≥30秒/5周期；正反是条件顺序，非查询列表倒序。rate0仍有诊断开销，只计lc0；诊断客户端4.291/5.897ms不是无插桩产品PK。
- **结果边界：** 39924测量请求、含预热136471次有序ID一致，工作量/CPU全过。6窗底层半段漂移均≤3%，客户端4/6过；反向预取客户端漂移3.55%/3.18%，底层全周期极差4.40%/6.12%。6窗host均有换页/major、环境0/6，正式比较null。不能说原0.190ms消除、不能将旧worker自身约8%改称相对串行提升；硬件最终根因未证明，不改cost强推并行。
- **脚本失败保留：** 第二窗原始测量保存后，progress.json独占写重名中止；改为按ordinal命名，新增单测，保留INCOMPLETE，从ordinal2续原排程且前两窗不重测。恢复时重建长连接，每窗内持续会话/独立预热，不称全程只有同一连接。compare_serial.py全跑须新目录；--resume-from 2仅适配本次特定失败。当前0b2无emit，复跑先归档补丁/统一Release构建核验，不能套诊断偏移。
- **后续/恢复：** 候选去计时后先回收/并发语义，再无插桩完整执行/端到端，同时研究残余底层差额。HEAD/产品仍5bde98afc0/099656b971，无新源码仓提交或Milvus PK。源码已撤回，恢复构建release_incremental_20261016_172137.log于17:25:25成功，安装/实际PID2263337回到0b2完整hash；原生5/5、12872 ID、脚本13/13通过。17:29:45 fast停库，原配置8c83e997…/默认池OFF、亲和性/自有probe恢复，原tracefs保留；Milvus/Oracle未启动、各目标端口无监听，原dirty与基准数据/索引保留。核验见serial_restored_validation/summary.json、completion_serial.json。

### 6.35 首次触碰组件消融：页头主导、锁其次（2026-10-16）

- **入口/观察：** 交接§0.51、总报告§4.5.30；`验证/HNSW首次触碰因果对照_20261016/component_v1/component_audit.json`。同一诊断ELF原路径→全预取，纯lc0串行3.101→2.902ms（降6.44%）、worker 3.336→3.072ms（降7.90%）；差额0.234→0.171ms，观察少27.24%，但worker仍慢5.88%。不同ELF/采样率，不能与上一轮3.83%直接算产品回退；没有新Milvus PK或成本改动。
- **新定位：** 去掉页头预取使worker多0.278ms/9.05%，页内定位/提示36.03→135.07ns；去掉锁预取多0.079ms/2.58%，首锁36.10→78.86ns，正反方向均复现。去掉描述符额外两条预取仅多0.005ms/0.16%，但仍提前读content_lock指针；不能说完全不触碰描述符。单段/组件作用不可相加，锁CPU不是争锁证据；cache层/TLB/频率最终原因未证实。
- **方法/门：** SciFact99.0171%、1109查询、100/100/350/4096KiB/M64/800、池ON长连接、HNSW guestCPU2/leader6/client8、1/128抽样。5条件×2模式×2方向20窗全部独立预热收敛（26.86～66.47秒），每测≥30秒/5周期，无跳过/恢复/补跑。133080测量、含预热290638有序ID一致，工作量/提示覆盖/CPU全过；20窗前后半段稳定，但环境仅4/20合格，无完整干净双向对照，不是产品性能验收。全预取诊断客户端4.064/5.687ms不是无插桩产品PK。
- **下一步/复跑：** 优先剥离锁＋页头候选的计时，先并发回收/槽位复用/失效/多形态，再无插桩完整执行/端到端。178行temporary_native_components.patch、component_protocol.json、helper机器码和布局审计均归档；record v2的v[27]低8位为variant/其余hint数，不能用旧低2位解码。run_components.py只用新目录，固定20窗，不无限重试。机器码另确认描述符步长72字节，64字节union仅是最小大小；未证实伪共享，不直接改布局，改头文件必须Release full。
- **最终恢复：** 诊断PID2284542/hash0ba59e4f…，18:20:33测量完，178行源码精确撤回；Release日志release_incremental_20261016_182116.log于18:24:56成功，安装/实际恢复版PID2299687精确恢复0b2，原生5/5、四档四模式12872 ID通过。18:29 fast停库，原配置8c83e997…/默认池OFF、亲和性/hnsw_parts16探针恢复，原tracefs保留。HEAD/产品仍5bde98afc0/099656b971，无新产品提交或push；Milvus/Oracle未启动、各目标端口停止、基准数据/索引/原dirty保留。脚本22/22及独立核验见completion_components.json，恢复版证据component_restored_validation/summary.json。

### 6.36 BM25 重叠因果诊断与待解释的并行结果异常（2026-10-19）

- **入口/结论：** 交接§0.52、总报告§4.5.31；证据`验证/HNSW并发重叠因果对照_20261019/diagnosis_audit.json`。自然时序VECTOR 2218次搜索重叠0%，FORCE1777/2218（80.12%）。保留HNSW worker、禁止BM25重叠，干净正向窗3.593ms、同轮串行3.341ms；强制重叠3.550ms。FORCE同时3.632ms、HNSW先完成/零重叠3.665ms。BM25同时搜索不是单worker差额的必要原因，不能把它直接解释成两路争锁；具体worker执行状态/cache/TLB/频率原因仍未证明。
- **口径：** SciFact共同参考99.0171%，5183文档/1109查询/768维，Top10/RRF60、100/100/350/4096KiB、M64/800、池ON长连接。计完整GetScanItems图搜索，含上层下降，不含门闩/初始化/融合/客户端，非旧lc0。HNSW guestCPU2，worker模式文本路6、串行2、client8；TID轮换，不是物理核隔离或全程同TID。同查询＋同TID配对未改变上述方向，非随机化完整因果验收。
- **失败必须保留：** 原14窗只完成7个正向窗39924测量，环境6/7、漂移6/7、同时通过5/7；反向w07预热未收敛，w08 FORCE/同时预热ordinal233 ID断言失败停止。固定11346次跟查通过不能清除异常；既定64查询交替确认中VECTOR预热未收敛、FORCE第59预热周期ordinal1038/BM25先完成再次ID失败，无交替测量结果。完整自然时序4436次ID/工作量通过不等于整套通过，不补跑到满意、不改冻结基准。
- **下一项先查正确性：** 第二次错误恰好等于纯向量Top10，串行与候选离线融合匹配冻结基准，疑似文本候选未完整合入。`gs_find_memory_data()`先读dataStatus再读is_connect_end，旧EMPTY＋新结束标记可疑提前EOF时序尚未现场定案。未捕获失败请求收端状态/支路行数，不能断言是诊断专属或原产品必现；不擅自留下修复。详见`confirmation_v1/force/ID_MISMATCH.json`及`anomaly_shape.json`。
- **恢复/资源：** 四个.cpp临时262新增/1删除已精确撤回，统一Release恢复日志release_incremental_20261019_102320.log于10:27:10成功，安装/实际PID2980537 hash精确回到0b2完整SHA。原生5/5、四档四模式12872 ID通过，原版FORCE长连接两个异常查询各5000次共10000次未复现，但不清除风险。10:33 fast停OG，原配置8c83e997…/默认池OFF、亲和性/新增线程绑定及hybrid_overlap19探针恢复，原tracefs保留；Milvus/Oracle未启动，所有目标端口停止，用户dirty/基准数据/索引保留。HEAD/产品仍5bde98afc0/099656b971，无新产品提交或Milvus PK；脚本10/10及核验见`completion.json`。
- **复跑限制：** `temporary_overlap.patch`只归档诊断，当前0b2无emit符号；重做须apply_patch、统一Release编译核验、新输出目录、临时开池。门闩为单客户端顺次请求的临时全局状态，不能用于多客户端压力测试。`check_restored_anomaly.py`则只允许原0b2；不能用反复通过的小样本掩盖已有正确性失败。

### 6.37 Local Stream 提前 EOF 已实证修复（2026-10-19 11:43收尾）

- **当前产品：** `feature/hybrid-recall@e84f354971`，仓内BASELINE§31、交接§0.53、总报告§4.5.32。只改`libcomm_memory.cpp`，加两份独立回归文件；没有旧HNSW临时插桩，无头文件/索引/候选/成本变化，未push。权威安装/实际PID3010992 SHA256为efe0796e…完整值见§5.3；0b2已是旧版。
- **确切证据：** 原0b2、池ON、SciFact99 ordinal1038。non-stop GDB仅暂停consumer，两次读取间producer正常发100条并结束；consumer返回FINISH时100条仍未读，最终ID精确等于历史纯向量Top10异常。修复版同条件先DATA后FINISH；在两次读取后才发布则先WAIT后DATA，均正确。没有写状态/候选/寄存器值，未估计自然发生率；此前两个失败请求无收端现场，不能追溯保证其唯一原因。
- **修复/验证：** 完成标记→读屏障→队列状态，producer写屏障→完成标记，不加锁/睡眠。Release增量日志release_incremental_20261019_113044.log，11:35:41成功。85种真实函数交错旧18失败/新全过；原生5/5、四档四模式12872 ID、并发400、20错误/80超时尝试恢复、200组完整批次结果/支路行数、两异常查询各5000次全部通过。普通行式/向量化DOP2/4、Gather/Redistribute另160次完整结果和4次错误恢复通过。ARM、长期稳定性未验收。
- **证据/复跑：** `验证/LocalStream提前EOF验证_20261019/`的[audit.json](公共文档/两路召回对比Milvus/验证/LocalStream提前EOF验证_20261019/audit.json)、[validation_v1/summary.json](公共文档/两路召回对比Milvus/验证/LocalStream提前EOF验证_20261019/validation_v1/summary.json)、[completion.json](公共文档/两路召回对比Milvus/验证/LocalStream提前EOF验证_20261019/completion.json)。GDB脚本仅适配记录的x86-64 ELF/hash/机器码，须新输出目录；前两次调试器接入技术失败保留。validate_fixed.py包含既有全量正确性门，不用于性能测量。
- **资源/下一步：** 11:43 fast停OG，原配置8c83e997…/池OFF恢复，Milvus/Oracle未启动，调试器已脱离、无探针/CPU绑定变化。仅自建批次夹具已删除，基准数据/索引及用户dirty保留。HNSW worker纯执行差额仍待查；接下来恢复无BM25重叠条件，定位首次页访问/执行状态，不用此正确性修复宣称性能超越Milvus。

### 6.38 同线程重新搜索诊断：未形成新的根因证据（2026-10-19 14:41收尾）

- **结果/入口：** 交接§0.54、总报告§4.5.33；`验证/HNSW同线程重新搜索对照_20261019/diagnosis_audit.json`。BM25零重叠，正常搜索反向串行/worker3.400977/3.657376ms，观察慢0.256399ms/7.54%。连续第二次完整搜索：串行两窗降7.00%/6.73%，worker唯一窗降11.46%；缺反向worker，正向双次窗漂移，不是worker反超或唯一cache/TLB/频率根因。
- **方法/门：** e420诊断Release、SciFact99.0171%/1109查询、100/100/350/4MiB/M64/800、池ON长连接；AUTO0worker/VECTOR1worker，同一查询两遍同TID/CPU2，worker模式leader6/client8。第二遍前清空工作区/visited/candidates/discarded并重新归一化，不复用结果；计完整GetScanItems，排除gate/reset/指纹/初始化/融合，不是旧lc0或端到端。8窗固定排程，w01/w04预热127.13/126.35秒失败跳过，6实测窗35488请求，稳定4/6、环境0/6；117586 ID核对/169725搜索，指纹/工作量/CPU通过。不补跑到满意，不按未平衡均值算差额消除。
- **代码/恢复：** HEAD/产品仍e84f354971，无新产品提交/成本/索引变化。三个.cpp临时180新增/1删除归档在run_v1/temporary_reentry.patch，14:31精确撤回；Release日志release_incremental_20261019_143145.log于14:35:34成功，安装/实际PID3076055回到efe0796e完整hash。恢复版原生5/5、EOF交错85/85、12872四模式ID、并发400、20错误/80超时尝试及恢复全过；脚本解码5项通过，见restored_validation/summary.json。
- **资源/继续：** 14:41:30 fast停OG，配置精确恢复8c83e997…/默认池OFF；hnsw_repeat19探针和原/新增线程亲和性恢复，原tracefs保留。Milvus/Oracle未启动，目标端口关闭，原dirty/基准数据/索引保留；最终核验completion.json。重做必须新目录、重新诊断编译并核验hash，恢复版无emit符号；不得恢复旧0b2。下一步须先有稳定干净配对才能定微观因，或单独推进页头＋锁预取去插桩的安全与完整执行/端到端验收，不合入重复搜索。

## 7. PG19 原生多跳图执行、graph_adj 与 AGE 邻接适配（本机记录截至 2026-10-25）

**跨机接续补充（2026-11-08）：** 远端 PG 已发布为 `baefa21b3`、AGE 为 `dc9f36cb`，完整 SHA 见 §0.2。本次新增所有权迁移、并发诊断与统计预算修复仅完成静态复核，未编译测试；下方的安装、运行、性能和回归数据均是旧源码对应的本机历史证据，不能改写为远端新版本已通过。

### 7.1 接手顺序与当前状态

**2026-10-25 最终复测及提交快照：** PG `ca3c0206446317812d0fab015e9c598a314dee01`、AGE `a1cf53a657c14d291fd0f442e9ee6fac1d96a4db`，两仓clean；新增公共7feb54db/B-tree539cf867/adjca3c0206＋AGEa1cf53a6，未amend/push。安装仍065407/49827567，提交前后源码字节/模式及三二进制独立核验一致，active manifests保留compiled_sources后绑定commit，未重编。报告§36、性能§2.46、简表§2.6为最新；下方未提交是历史。072335四例26064样本/073645复算PASS，单链/稀疏/汇聚计时合格，连续角色分布3:1未过；B执行−0.02/未签收/−4.25/+1.78%，区间跨零，前后相对relational4.25→4.17、未签收、2.33→2.40、4986.45→4781.01×。073654 LIMIT/073734审计6480条全部过门，8192边三跳LIMIT1/10/100相对旧B89.79/55.33/13.40×、相对本版relational1.06/1.24/1.08×；不是普遍relational几十倍。单链规划+4.41%与adj单链+4.47%不利信号仍在，Ir各仅+0.26%，全部耗时原因未定；无新产品改动。证据072246 closeout含commit/rebind/completion，首次未改bat CRLF校验误拒资料保留、重核通过。自有实例/采样停止，数据/源码/构建保留；AUTO/DML/生产门及task_state不变，不自动追加短轮。

**2026-10-25 审查修复最新：** 当前PG e040226＋dirty、AGE01f2d36＋既有dirty；065407配套O2/`49827567f5af5ca10fe4e730ca0c13f461315d3a0c4f18e09dea1f829299418f`，未提交。报告§35/设计§29/交接顶部为最新，下面19379为历史恢复版。修复顶点VM leaf pin错结果（旧3例复现、修复6例通过），阶段过滤和重复选择率、普通角色统计授权/鲜度及membership generic计划失效；大批去重和公共数组/顶点扫描整理。LIMIT部分读取新增Fixed/Native成本候选，三跳4096前沿LIMIT1仅7条子计划交付、64KiB成功；非spill，完整边游标未强行合并。最终图30/PG245/AGE43及统计/资源/缓存/角色继承、LIMIT专项通过。062445固定六启动26064样本和064315复算正确，19379→中间d92有效启动1/1/1/3不足，**不能发布最终4982的新耗时倍率/零退化**，不无限追加短轮。证据`20261025T054921.358050Z-multihop-review-remediation`保留前后补丁、差量、progress/completion；源码/种子/冻结安装保留，自有服务已停。AUTO/DML/生产门及task_state不变。

**2026-10-25 四例图B-tree自身执行恢复完成：** 当前PG e040226＋dirty、AGE01f2d36＋dirty，040045/19379fe5。持锁内部页下界复用和已准入bigint叶页比较保留NULL回退/锁/MVCC/SSI/预算及原生结果断言。图27/PG245/AGE43、批访问/覆盖/完整结果、040327组合断言/资源/并发统计和040538的134专项通过。040644六启动26064条/041923复算/24份8候选PASS，合格启动6/5/6/6；B自身执行−5.13/−5.30/−5.85/−8.28%，四项区间上界<0。前后相对relational3.74→4.14、4.69→4.88、2.20→2.30、4963.43→4771.92×；平行R自身−12.22%，倍率仍下降，不能称所有相对倍率已恢复。六关系计划除计时相同，035014前一版da指令仅+0.00017%，物理周期全部原因未定因；不可冒称当前19379指令。主表/交接已刷新当前版；C8/B1当前补丁与验证见035251，完整四组合仍180939。原四例图B自身恢复范围已闭环，不自动增加同版短轮；adj严格零退化、AUTO/DML/生产门/task_state不变。全部自有实例/采样器已停，种子/源码/安装保留、未提交。

**2026-10-24 原四场景变慢归因最新断点：** 简表§2.4、性能§2.44、交接顶部及142307汇总。全部forced，不是AUTO；图B现有8候选只解释新增规划工作，未做候选数消融。旧5d3源码O2重建核心d4f00bd1＋匹配模块位于`.build/pg19-graph-core/report-before-4szzbppg/install`，非10月6日原ELF／quality旧c99。134944与140815两组同期各26064样本，完整结果／元数据一致，140500／142211复算通过但跨启动环境门均未过；不发布新耗时退化百分比、不覆盖下方历史表。140539指令及141845的60份dump复算：单链／连续／稀疏／平行B +4.06/+4.28/+1.47/+3.61%，连续公共执行器＋AGE +15.53%，稀疏adj +5.12%；树例端点hash约占净增一半，adj新增重复metadata注册子树占77.04%。指令不是耗时。当前仍123401/ead5ab89及PG既有dirty，未改产品／重编当前产品／提交；后续优先减少重复注册和小前沿hash，禁止旁路安全／预算检查。自有实例全停、原种子／源码／安装／保护环境／task_state不变，分析范围已完成；AUTO/DML/生产门不变，不无限追加计时。

**2026-10-24 原汇报四场景已重测：** 简表§2/性能§2.43及交接顶部为最新原场景数据；仍123401/ead5ab89、PG既有dirty，未改产品/重编/提交。原100k/十跳四套数据、SQL、索引统计和设置与10月6日相同，完整返回值一致；253712原始样本复算PASS，平行/单链/连续/稀疏合格6/6、8/18、6/6、5/6轮。B/adj/公共执行器＋AGE热态倍率分别4043.27/4044.17/未测、8.40/7.69/15.04、5.40/1.96/9.93、2.27/1.89/16.00；B规划595.03～1546.33×。已测图路线自身热态耗时比10月6日历史增加3.47%～20.99%；不是同期新旧代码归因，单链倍率上升不等于自身变快。AGE首次9/9合格，60.060/54.179/54.590ms。总证据131542、热态合并132849、审计132903、平行132202/审计132848、首次132818。原表与当前的执行/规划前后表已写入，未覆盖下方不同quality场景的同期优化收益。自有实例全停、原种子/保护环境/task_state不变；此次报告任务完成，AUTO/DML/adj/生产门不变。

**2026-10-24 单链定向优化已完成：** 报告§32/性能§2.42/设计§27及交接顶部为最新。用户“请优化”后新增B-tree空工作区有界复用：nbtree.c/nbtree.h，正常endscan释放全部pin/键/数组/图提示再入池，活动context仍归调用者、异常按PG清理；每backend空闲最多4份/256KiB，不缓存结果/快照，不改glibc阈值或恢复8MB预留。当前123401/ead5ab896bfdd90e20b1465e6d82ae42d68bc1d395677d463803215465b0a0e9 O2，AGE01f2d36源码干净并配套重编；PG e040226加未提交dirty。123620同期四路1296样本及124219复算：此次5dcb→ead5ab89单链/属性/稀疏纯执行耗时配对-37.32%/-2.71%/-8.76%，相对relational1.93→3.21×、2.53→2.78×、1.34→1.43×；稀疏区间跨零，不签严格零退化。三例10万边，属性1跳1000行，其余10跳1/100行。124359两启动各300条问题查询执行阶段minor fault从7次/条降0；无PREPARE连接原本无此问题，不能外推所有查询37%。图27/PG245/AGE43、124346的24并存扫描/回滚/错误/DDL、124536资源及跨provider、124652双会话MVCC均通过。最初惰性mark候选无耗时收益，已撤回，仅保留证据。改前5dcb在`.build/pg19-graph-core/quality-pre-lazy-mark-cfu7gnz8/install`有baseline.json；c99/a93/种子/原文档保留。未提交；本轮未重跑四组合CASSERT/DML长期/生产门，AUTO/既存adj问题/task_state不变。自有实例与采样器全退，125156收尾核验通过、保护安装/种子不变。此定向优化已完成，不自动扩展调优范围。

**2026-10-24 单链差额根因最新闭环：** 见报告§31/性能§2.41/交接顶部，覆盖下方“未定位”。原093418直接EXPLAIN连接保留校验PREPARE，只有新B每查询13次minor fault；实际遍历/索引工作不变。115823缺页栈复现、120153双向trim控制、120526一次旧8MB分配/free对照及120754复算完成。旧8MB端点arena释放抬高glibc动态回收阈值，新版按需分配去掉该副作用，某些连接布局导致堆尾反复收缩/扩张，扫描工作区重复触页。新B两个启动观测约44～46→29～32µs（仅隔离backend禁trim），恢复阈值又慢；旧B强制相同阈值也慢；全新无PREPARE连接无持续缺页。原2.88→1.94×/47%作为特定会话真实信号保留，不能泛化或拿带探针数字替换。只编译证据目录observer，未改正式源码/安装，仍065148/5dcb及既有dirty；自有实例全停、种子/构建身份不变。分析任务完成，不自动新增优化/计时；后续可稳定有界工作区分配复用，禁止靠恢复浪费的8MB预留或全局关trim包装为修复。AUTO/DML/adj/生产门/task_state不变。

**2026-10-24 同期四路计时最新断点：** 用户所需三项前后分别相对relational已补齐，见报告§30/性能§2.40/交接顶部。093418普通EXPLAIN四路1296样本及114724复算通过：旧c99→当前5dcb，单链/非重复属性/稀疏纯执行倍率2.88→1.94×、2.49→2.53×、1.57→1.55×，合格启动6/5/6。092723复用计划86400样本及093410复算通过，对应2.08→2.09×、2.67→2.69×、1.14→1.12×，各6启动。两组各固定轮次独立统计，原数据均10万边；属性为1跳1000行，其余10跳1/100行。普通单链新B/旧B耗时+47.24%（探索区间+28.89%～+66.73%，六启动均增），须保留为待定位信号，不能用复用或§29跨零结果覆盖，也未证明唯一根因；不签严格无退化。当前源码/065148安装未改或重编，仍未提交；自有实例全停，原种子/构建/二进制身份不变。此轮测表已完成，不自动追加轮数；AUTO/DML/adj缺陷/生产门及task_state不变。090207/090522旧对照仍保留，075026为未收尾历史。清理已释放67.3GiB，仅删156个旧运行副本，全部构建/安装/源码/原公共文档及关键种子保留，详见执行包`evidence/archives/20261024T083913Z-graph-runtime-cleanup/`。

**2026-10-24 同版本计时/换页来源最新诊断：** 072523当前5dcb对5dcb的A/A完成28800样本，072746计时/主机观测审计通过；PG后端major fault/swap及查询物理/temp IO全0，外部缺页主要来自VS Code扩展宿主，不是PG后端换页。有效轮2/2、0/0、0/2，单链合格轮同版本仍有-4.87%配对差异，不能仅凭零换页门归因几个百分点的代码收益。按预先规则未执行v6新旧对照，不再同条件刷轮；需安静窗口和更多独立启动的交错A/A、A/B设计，暂停用户后台进程需另获授权。见报告§28/性能§2.38/交接顶部。产品未改/重编、仍065148/5dcb；自有实例全退、种子/保护环境/task_state及未签收门不变。不得将同版本差异当加速或从新旧结果中扣除。

**2026-10-24 最新端点哈希/节点分配修复：** 当前065148/5dcbad42配套O2，`graph_fixed.c`使用PG simplehash＋稳定节点小块分配，实际预算含全部预留块及新旧hash数组共存峰值；MVCC/权限/语义不放宽。主安装图27/PG245/AGE43、资源及8组跨provider、生命周期/GUC6/AGE语义13/adj安全9通过。065757四组合CASSERT从7873独立重放PASS：图3/23/5/27，四组各PG245，含B两组资源通过；当前完整C/B/A补丁采用该run，不再用020638旧包。065636/065757指令核对48份profile：相对开发前c99图B-tree，单链/属性/稀疏额外指令+0.88%/+0.10%/+0.77%，稀疏上版a93为+2.06%；这是CPU工作，不是时延。065400最终版28800样本环境门未过（有效0/1、0/1、0/0），070700三段10秒预检均swap-in，未追加计时，预检机会已用完；需合格测量窗口另验，不能放宽门/同版刷轮。见报告§27、设计§26、性能§2.37及交接顶部。PG e040226加未提交修改，AGE01f2d36干净；自有进程全退，种子/保护安装/task_state不变。未重跑批访问/覆盖全专项/DML性能；另有既存adj高扇出构建错误，064647在a93/1bda复现、未修复。未提交，AUTO/DML/生产门不变。

**2026-10-24 最新CPU开销归因：** 产品仍033820/b26未改/未重编。035200在原c99/b26上只统计Executor四阶段，指令单链+1.02%、属性+0.70%、稀疏+3.35%，索引比较/扫描/回表次数一致；稀疏增量主要为通用端点hash及903次内存统计，属性为零命中缓存维护及新增3000次内存统计。035520同d13诊断版旁路缓存/预算分别少171996/65000条，严格可加；035948审计80份profile通过。见报告§25、性能§2.35/交接顶部；不是原生耗时倍率或旧+3.09%/+6.80%全部差额精确归因。只诊断未实施新优化，禁止把预算旁路装进产品；所有自有进程退出、种子/保护安装/task_state不变，未提交。

**2026-10-24 最新Sort修复及三项复测：** 当前033820/b26ebc48配套O2，`graph_upper_paths()`不再保留无ORDER BY消费者的冗余DISTINCT排序；原证明边界/必要排序不变。图27/PG245通过；033919两次启动角色互换、每例12轮，034141审计28800条及四负例通过。仅比较开发前c99→修复后b26图B-tree：单链耗时+0.34%（有效5/4轮）、非重复属性+3.09%（5/5）、稀疏+6.80%（3/4），正号表示本轮变慢。三项环境门均过，Sort在实际计划已消失，但整体性能退化未解决，区间跨零不等于零退化。见报告§24、性能§2.34及交接顶部。PG e040226＋八项/本修复未提交，AGE01f2d36干净；自有实例全停，种子/保护安装/task_state及默认开关不变。未重跑本修复四组合/完整AGE，不把历史证据当新验收，不自动追加短轮。

**2026-10-24 最新三项开发前后复测：** 用户明确只比较c99旧图B-tree→f4c9新图B-tree，不能混入相对relational倍率。032259两次启动/角色互换、各例12轮，032501复算28800样本通过；单链有效轮分布0/6，第二批全慢但跨实例门未过；非重复属性2/4轮，执行点估计快1.76%；稀疏6/2轮，慢1.73%，后两项探索性区间均跨零，未证明稳定增减/修复，旧负信号保留。新版单链/稀疏计划均多外围Sort，实际时间保留，未修复。详见报告§23、性能§2.33、交接顶部。产品源码/安装未改、未重编/提交，自有实例全停；task_state/原多模环境/种子不变。不自动追加短轮或实施排序修复。

**2026-10-24 最新因果诊断/补测，覆盖下方“稀疏轮数不足”：** 正式源码及015721/f4c9安装未改，八项质量代码仍未提交。024921属性单因素16800样本/7有效轮，关闭缓存/绑定预算均无稳定改善，根因未闭环；本协议旧/新约+8.46%与旧+3.04%负信号均保留。030121重启/角色/ASLR对照6400样本未达有效周期门，不得归因为布局/噪声；030332审计通过。025206原10万边/十跳单链、稀疏有效2/8轮，030040复算158125条一致；稀疏B相对relational规划1267.41×、执行1.918×，adj1.730×、公共执行器＋AGE热态11.081×；单链仍不发布，连续键本轮未测。见报告§22、性能§2.32及交接顶部。隔离诊断版`.build/pg19-graph-core/quality-cause-xpHElVnN`含私有旁路开关，严禁当产品安装；自有实例全停、原多模环境/种子/task_state不变，未commit/amend/push。不无限追加短轮，不擅停用户其他应用。

**2026-10-24 八项质量改进的实现/正确性已完成，覆盖下方 clean/旧安装/额度阈值：** PG HEAD仍`e040226`，本轮修改未提交、禁止覆盖；AGE`01f2d36`干净，用户已取消额度停止限制。当前015721配套O2、ELF`f4c9fad8…dc3a311`；图27/PG245/AGE43、Native61、资源/生命周期、GUC6/AGE语义13/adj安全9、批访问673及覆盖索引224＋64＋1通过。020638独立四组合实际CASSERT=on，图3/23/5/27、各PG245及入口/符号/资源门通过。Fixed/Native元数据均已版本化；旧布局SIGSEGV、内部PARAM_EXEC来源断言、Unique及HashAggregate行数规整问题已修复，失败证据保留。021602同期旧/新B筛查：早过滤24.24×、重复属性2.09×、宽前沿1.14×；单链/非重复属性配对耗时仍约+1.65%/+3.04%，不签全场景零退化。022119原十跳三场景固定12轮完成，023303审计236108条一致；单链/连续/稀疏有效1/3/2轮，仅连续键发布B相对relational规划1391.96×、执行4.313×，另外两项环境门未过，不使用旧数字填空。C+B内核净增21410行，含注释空行、不含测试/adj/AGE。详见报告§21、设计§24及交接入口；自有服务全停，AUTO/DML/生产门与task_state不变，未commit/amend/push，保护原多模环境与用户种子。

**2026-10-23 16:47 提交完成，优先于下方历史dirty状态：** 新增C公共层`569023180f4dad8371f3ea3908f89703b9cb851d`、B专属层`e04022675d1235d77b8fe71a93af07396574b9ba`，旧六提交未改写、未push。PG HEAD=e040226且干净，AGE01f2d36仍干净；A/AGE无新提交。提交前后7762个源码文件及权限与072836验收快照完全一致，现用070256二进制不变，本轮未重编/重测。构建活动清单仅在等价核验后绑定新HEAD，保留`compiled_sources`和原始构建证据。见[提交记录](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/evidence/runs/20261023T084745.579699Z-graph-cb-commit/summary.json)及[分层说明](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/代码分层与独立迁移说明.md)。C+B内核相对7873db5净增19587行（含注释空行，不含测试/A/AGE）；测试及expected另7738行。G1-080/DML零退化/生产门不变，服务均停。08:39 UTC额度实读59%，后续重读，≤10%立即停止。历史迁移v3脚本固定旧HEAD，重新运行须先适配版本校验，不能直接把旧日志当新验收。

**2026-10-23 强制hint十跳最新复测：** 用户要求的原三例10万边/十跳/无重复复测已收尾，无内核改动/重编/提交，仍下段070256 O2。080534 PASS，单链/连续/稀疏有效轮5/9/10；强制B-tree相对relational规划1542.35/1431.36/1294.52×，纯执行8.37/5.28/2.23×。081814审计233221样本PASS，新旧数据/SQL/统计/索引/设置一致。见[报告§20](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#20-最新版强制hint十跳复测2026-10-23)及性能§2.29；不是AUTO/default GEQO结论，也不是新增代码加速。DML/生产门未重测，全部自有实例已停、源码及task_state不变。账户08:18 UTC恢复实读60%，后续须重读、≤10%立即停止，不自动追加实验。

**2026-10-23 三叉树AUTO/DELETE最新断点，覆盖下方034327状态：** 见[报告§19](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#19-三叉树auto修复与delete受控对照2026-10-23)、设计§23及性能§2.28。原生JOIN在路径创建前消费同一条件边前缀基数，处理DP推迟顶点检查；批量索引成本使用实际数组请求并区分尾批。071004原10＋新4场景完成，三叉树自然选B，R/B/AUTO纯执行0.14236/0.05889/0.07400ms，AUTO/最快1.26；AUTO规划2.709ms高于R2.323ms，不承诺端到端改善或通用GraphExpandNative AUTO。48平行边落盘仅验计划/结果。071455同一物理数据目录完成49152提交事务，192计时段无checkpoint；direct新ON/旧OFF1.0223 [0.9784,1.0915]、prepared0.9960 [0.9587,1.0417]，剩余正差额主导在WAL同步等待，不改DML/WAL、不签零退化、不追溯旧5.05%全部归因。072116原始复算通过，生产12＋5输入及G1-080门仍未关闭。

现用070256配套O2，PG ELF SHA256=`c99ab06c9e3523c3a626db4cae77e058deddf17f18e4e5b52a253c16f3624dc6`；PG b9108dc＋**31tracked＋10新文件未提交**，含前轮改动，AGE01f2d36干净。图23/PG245/AGE43/GUC6/AGE语义13/adj安全9及072836四组合图2/19/4/23、各PG245和额外入口/符号通过，当前完整迁移补丁用072836。全部自有实例已停，原PG19＋ORCA＋Daft/旧AGE/用户种子/task_state不改，未commit/amend/push。账户最近成功读取07:14 UTC余67%，收尾查询网络失败；继续前重读，实际≤10%立即停止，不能拿旧比例当实时余量。限定工程工作已收尾，不自动扩大生产压测或继续短窗追数。

**2026-10-23 AUTO/DML/生产门最新续做，优先于下方旧状态：** 已接通安全单常量锚点的WALK端点集合统计（result_mode2/metric210），不改原PG/AGE统计入口、DML/WAL或默认开关。041549无测试模块10个10万边四跳场景＋7边界完成，修复32/48平行边及36留出漏选；三叉树仍选关系，AUTO0.12425ms对最快强制B0.06336ms，通用路径AUTO未开放。034754有98304个实际测量提交事务，DELETE新ON/旧OFF+5.05%不利信号保留；旧/新checkpoint不平衡，零退化未签收。042548原始复算为AUDITED_NOT_SIGNED，生产缺12业务＋5统计预注册、用户尚未答范围/SLO/预算，三项整体门未关闭。见[报告§18](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#18-autodml和生产门续做2026-10-23)、设计§22、性能§2.27。现用034327 O2、ELF SHA256=`17d0dd58b978cc53875a4f8a1228a96e6dd0d396bcada6f98e3874a717227b6d`；PG b9108dc＋29tracked＋10新文件未提交、AGE01f2d36干净。图23/PG245/AGE43/GUC6/AGE语义13/adj安全9通过；040829新四组合图2/19/4/23、各PG245，完整C/B/A补丁使用这版，不用021421。源码/安装身份见交接顶部；自有实例全停，未commit/amend/push、未改保护项目/种子/task_state。账户12:28北京时间实读72%，≤10%立即停止，接手重读。C函数定义本身会加载库，统计测试的辅助定义和调用须放独立管理会话；032107/035710失败不得冒称无模块性能通过。

**2026-10-23四组合迁移最新验收，覆盖下方“未复验”：** 当前dirty源码的C/C+B/C+A/C+B+A独立O2编译、图2/18/4/22、各PG245/245及无模块PGQ/缺provider拒绝/通用类型SRF/工厂符号检查全通过，见[报告§17](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#17-四组合独立迁移复验2026-10-23)。脚本`pgq_btree_portability_v3.py`生成完整C/B/A补丁，从干净7873db5实际重放编译，A/B可交换；证据021421，临时根目录`.build/pg19-graph-core/provider-matrix-_p_plpmw`保留。不是六commit的cherry-pick或新提交验收。未改内核、原源码/现用014550安装/保护项目未变，PG仍b9108dc＋21tracked＋8新文件未提交，AGE01f2d36干净；自有实例全停。AUTO、通用路径性能、DML严格零退化/生产门及task_state不变，不自动扩项。账户10:17北京时间实读82%，≤10%立即停止，继续须重读。

**2026-10-23九类接入最新增量，覆盖下方旧限制/安装：** 新增`GraphExpandNative`通用类型图绑定状态机，接入非整数/复合键、NULL、RLS、分区/继承、视图/file_fdw、LATERAL、多标签、合格部分/表达式索引及合法复杂表达式；不是整图JOIN改名，实际原生扫描完整显示。保留整数`GraphExpandFixed`；通用路径仅强制B-tree、完整bag，AUTO/AGE默认off及原边界不变，不承诺新性能倍率或任意FDW/TableAM。见[报告§16](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#16-九类原生能力接入2026-10-23)、设计§20。PG HEAD b9108dc＋21tracked＋8新文件未提交（含前两轮），AGE01f2d36干净，禁止覆盖。当前014550配套O2，PG ELF SHA256=`ff0d8af078535aada30001be58440052519e47ff73a6277d47cb439a66648096`；语义4、固定执行3、provider2/lease4/adj磁盘7，Query/Plan结构改变，扩展须配套重编。新专项61、图22、PG245、AGE43、旧专项13/30、GUC6、AGE语义13、adj安全9通过；新脚本`pgq_btree_native_v1.py`。限定功能已完成，无新性能/四组合迁移验收，AUTO精度/DML零退化/task_state不变。自有实例均停，未commit/amend/push，不自动扩项。账户09:43北京时间实读86%，≤10%立即停止，继续须重读。

**2026-10-23第二轮Top 3最新增量，覆盖下方旧状态：** 已接入smallint/integer/bigint单列非空键及跨整数连接、必要列的列级SELECT授权、同边表多图元素精确映射。PG HEAD仍b9108dc，累计10tracked修改＋5新文件未提交（含前轮），禁止覆盖；AGE01f2d36干净。当前`20261023T003604.394152Z-build-pgq-integrated`配套O2，ELF SHA256=`2db5401c1b36238c2852773140644799ece1119499ed775dbff797583b664d9d`。图语义/固定执行版本3；provider V2、lease V4、adj磁盘V7不变，扩展须配套重编。最终图21/PG245/AGE43/GUC6、新专项30、前轮映射13、AGE语义13、adj安全9通过，见[报告§15](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#15-第二轮-top-3-接入2026-10-23)与设计§19；新脚本`pgq_btree_integer_acl_v1.py`。RLS/text/UUID/复合键、多标签UNION未接入，AGE/adj不随之扩展。无新性能或四组合迁移验收，AUTO准确性/DML零退化/task_state不变。全部自有实例已停，未commit/amend/push，不自动扩项。额度≤10%立即停止；08:38北京时间实测94%，继续须重读。

**2026-10-22 Top 3最新增量，覆盖下方干净/162702状态：** 已补齐异构点边表固定路径、非首列bigint单列键/三列边表/合格UNIQUE索引、复用顶点变量闭环和自环。PG HEAD仍`b9108dc`，4tracked＋2新回归文件尚未提交，禁止覆盖；AGE01f2d36干净。安装更新到`20261022T101239.435855Z-build-pgq-integrated`，ELF SHA256=`ac3364923897466ab0184e95d252bbcfbca1df4f2408559be98faec5b7159394`。最终图20/20、PG245/245、AGE43/43、GUC6/6、两会话专项13/13通过，见[报告§14](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#14-pgq-top-3-接入补齐2026-10-22)和设计§18；新脚本`pgq_btree_mappings_v1.py`。不是文本/复合键/任意多路径支持，无新性能倍率或四组合迁移复验，AUTO准确性/DML零退化未签收。全部自有实例已停，保护项目/种子/task_state不变。本次限定任务已完成，不自动扩项。账户最新18:15实测22%，≤10%立即停止，继续须重新读取。

**2026-10-22提交更新，优先于下方历史状态：** 用户授权后新增`8937fa8`（公共层）、`4bcae38`（B-tree专属回归）、`b9108dc`（adj元数据适配），旧三个提交未改写、未push，当前六个特性提交。PG HEAD=`b9108dc04b76cecd795b56e4bb644e2495963b5f`且工作树干净；AGE仍01f2d36且干净。最终tree=`c4c35c449dfd4eaa405072a7ae1cc5e967bfddbb`，与162702构建的tracked补丁和16个新文件摘要逐项一致；本次没有重编或重新计时。AUTO准确性、DML零退化和G1-080状态不变。迁移映射见[代码分层说明](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/代码分层与独立迁移说明.md)。**用户最新要求剩余额度≤10%立即停止，替代25%；最近17:12北京时间实测26%，后续重新读取实际额度。** 下方未提交/25%规则为历史记录，不得据此覆盖源码或旧阈值停工。

2026-10-22进一步归因：[报告第12节](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#12-commit慢提交的系统层归因)、性能§2.26，2048事务内核追踪确认WAL数据写＋虚拟块设备FLUSH完成等待占有效简单链同步时间92.67%，最慢旧版12.23ms中FLUSH11.89ms。结论到Hyper-V块请求完成层，宿主内部未取证；不是图代码新增提交工作，也不追溯签收旧+3.91%或整体零退化。无内核/构建/提交变化，仍162702；自有库/采集已停。账户最新09:01余32%，≤25%停止。

2026-10-22 08:37补充：DELETE修改前后均OFF的分段归因见[报告第11节](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#11-修改前后均off的delete提交差额归因)及性能§2.25。24576个提交事务、固定数据目录交叉二进制，direct有效8轮端到端+2.18%点估计主要在COMMIT的WAL同步，DELETE请求/CPU无稳定增加；prepared48段零重规划、有效11轮，等待方向反转。旧+3.91%无法事后精确分段，纯执行/INSERT和DML零退化仍未结案。无内核/构建/commit变化，仍162702；自有实例已停，账户35%，≤25%安全停止。

这是在 PG19 SQL/PGQ 之上开发的图规划、公共多跳执行器和邻接访问能力，不是 §3 的 ORCA/Daft，也不是 §4 的 AGE 属性统计。下列相对路径均以实际 `CPP_WORKSPACE` 为根；原机器的根为 `/home/qiumc/data/code/cpp`。

**最新断点（2026-10-22）：公共图执行层＋图B-tree，无hint解析优化已验证，DML零退化未签收。** [清单第17节](公共文档/PostgreSQL19原生多跳图执行与AGE适配/设计/图Btree与关系算子组合执行设计.md#17-delete诊断与无hint规划优化)及[报告第10节](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#10-delete差额诊断与无hint解析优化)为当前结果。162702配套O2，图19/PG245/AGE43、无模块7组、GUC6、164026四组合各PG245通过。只对无`/*+`SQL跳过hint复制/词法解析，规划约减2%～3.5%；12288条对照和73728提交事务已复算，DELETE纯执行收益未证实，INSERT后ON/前OFF仍有+8.59%不利信号。prepared组被逐条VACUUM触发重规划，不能当无规划对照。没有改heap/索引/MVCC/WAL，不放宽零容忍门。图查询本轮未重测，沿用报告§9的无重复B-tree7.76/5.14/2.21×，AUTO全面准确仍未签收。HEAD5d3d3c1＋25tracked＋16新文件**未提交、不能覆盖**；graph_pgq.c仍304731ae…683915，hint为40983142…d8154d，安装ELF7fbd89d1…22ee88，AGEclean。正式入口无需LOAD、AUTO/AGE默认off，外围JOIN的AUTO保留原生，AGE全OUT。全部自有实例已停；不改保护项目、用户种子或正式task_state，G1-080仍in_progress，不自动追数重跑。账户≤25%停止约束及前后快照路径见交接顶部。

1. 先核对 §7.2 两仓的分支、HEAD、dirty 状态、实际运行二进制和端口；保留用户改动，不按历史快照 reset。不得影响原 PG19+ORCA+Daft 的源码、安装及 55419 服务，也不要借用 55439 的旧 AGE 实例。
2. 文档权威目录为 `公共文档/PostgreSQL19原生多跳图执行与AGE适配/`。继续开发先读[执行包交接入口](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/MODEL_HANDOFF_PROMPT.md)、[最新断点](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/继续开发交接提示词.md)的顶部当前状态，再读[代码分层与独立迁移说明](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/代码分层与独立迁移说明.md)。下方旧日期记录中的“未合入”“原型目录”不代表当前状态。
3. 性能见[GraphRoute 三路 hint 使用与性能](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/golden/G1-080/GraphRoute三路hint使用与性能.md) **§2.33 三项开发前后复测、§2.32 因果诊断、§2.31 最终质量改进筛查、§2.29 前版强制hint十跳**；不混淆开发前后与相对relational，不把旧PASS改名为本轮全部复现，也不将规划收益算作执行加速。设计总文档为[详细设计](<公共文档/02. PostgreSQL19原生多跳图执行与AGE适配详细设计.md>)；设计目标不能当已实现能力。
4. `执行包/state/task_state.json` 的活动任务仍为 **G1-080 / in_progress**。本轮实施与有限评估执行完，但有未签收的性能门，不等于 G1/G2～G6 全部完成或生产准入；任务状态只能经 `taskctl.py` 修改。除非用户要求，不因历史“持续优化到 10×”记录自动重启实验。

### 7.2 本机迁移源快照、历史提交与代码入口

| 对象 | 权威路径 / 分支 | 当前版本 |
|---|---|---|
| PG 图内核 | `.worktrees/pg19-graph-core/postgres19`；`feature/pg19-graph-core` | 本机迁移源 `ca3c0206446317812d0fab015e9c598a314dee01`；远端已归并并修复，见 §0.2；本机未同步，禁止覆盖现有资产 |
| AGE 适配 | `.worktrees/pg19-graph-core/apache-age-pg19-port`；`feature/pg19-graph-core-age` | 本机迁移源 `a1cf53a657c14d291fd0f442e9ee6fac1d96a4db`；远端归并版本见 §0.2 |
| Release/O2 构建 | `.build/pg19-graph-core/pgq-integrated` | 启用依赖跟踪，无 debug/cassert，保留调试符号 |
| 配套安装 | `.install/pg19-graph-core/pgq-integrated` | PostgreSQL、`test_graph_core`、AGE 必须匹配 |

PG 基线为 `7873db5369b967dd53984df34fd800003410d04f`，2026-10-06归并的三个完整特性提交如下；其后有C/B/A增量`8937fa8`/`4bcae38`/`b9108dc`，以及2026-10-23的C/B增量`5690231`/`e040226`，截至 2026-10-23 当时共八个特性提交（属于本地开发历史，**不是上游 PostgreSQL 已合入**）；其后提交见 §7.1，2026-11-08 远端已归并为 §0.2 的特性提交：

| 层 | commit | 归属 |
|---|---|---|
| C 公共层 | `c8166b1e7091f3c6afcde1db7271aa58f424b84d` | 图 IR、规划/代价/统计、公共执行器、PGQ hint、批访问公共接口、AGE 可选接口及 GUC |
| B 图 B-tree | `9b260b0e4958c9d0d556815cc8c72d3f3a881ab4` | 普通端点 B-tree provider、批量查找和安全的覆盖索引交付 |
| A graph_adj | `5d3d3c1d29af20946b238a4bbde2c38bebeefd77` | 独立邻接 AM、页/WAL/VACUUM/稳定引用、V7 多层目录及扩页并发修复 |

- 功能依赖是 **B→C、A→C，A 不依赖 B**；Git 提交顺序不等于功能依赖。最近完整补丁使用180939矩阵，从干净7873db5实际重放四组合、开启CASSERT独立编译，图3/23/5/27及各PG245通过，交换A/B顺序也一致；之后C八文件/B一文件增量只完成040327组合断言和040538的134项专项，见[迁移说明](公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/代码分层与独立迁移说明.md)。完整补丁不可重复应用到已有图特性的工作树；旧八个commit尚不包含本轮质量改进，逐commit cherry-pick组合不等同于这次源码补丁重放验收。
- 公共图语义版本 **v4**、固定整数执行 **v3**，provider ABI **v2**、lease ABI **v4**；Fixed/Native私有计划均有独立版本封套，CustomScan新增参数来源契约。PG、测试模块、AGE及其他依赖服务端头文件的扩展须配套重编，backend重启并重新生成计划。graph_adj 磁盘仍 **V7**，本轮没有新的磁盘/WAL/catalog升级要求；历史V5/V6索引的升级要求不变，不能未经授权重建用户索引。
- PG主要入口：`src/backend/executor/graph_pgq.c`及`graph_route_hint.inc`/`graph_btree_search.inc`（规划）；`graph_pgq_exec.inc`、`graph_pgq_native.inc`（两类执行）；`graph_pgq_plan.c/.h`（计划格式边界）、`graph_pgq_diagnostic.inc`（诊断）；`graph_fixed.c`、`graph_btree_provider.c`（公共展开/邻接）；`src/backend/optimizer/path/graphpath.c`、`costgraph.c`；`src/backend/access/graphadj/graphadj.c`；`src/backend/statistics/graph_stats_*.c`。AGE事务私有邻接在AGE仓`src/backend/utils/adt/age_visible_adj.inc`。
- 图统计使用独立 `pg_graph_statistic` / `pg_graph_statistic_data` 及图采集/消费接口；保持原生 `ANALYZE`、PG 列/扩展统计、pgstat 和 AGE 原有统计入口的语义，不用图统计覆盖它们。

### 7.3 三路执行与 AGE 开关：不要混淆的概念

当前标准图语法由内核直接接入，无需 `LOAD 'test_graph_core'`；默认自动选路仍关闭，不等于生产准入。对同一条 `GRAPH_TABLE()` SQL，仅改变 hint：

| hint | 实际含义 |
|---|---|
| `/*+ GraphRoute(relational) */` | 传统关系规划，允许各种 JOIN 和 Memoize，**不是强制 Nested Loop**；旧名 `nestloop` 不用于当前口径 |
| `/*+ GraphRoute(btree) */` | 合格整数用`GraphExpandFixed`＋普通B-tree provider；新类型/表形态用`GraphExpandNative`＋原生扫描子计划。没有新B-tree AM，也不保证每个孩子都选索引；实际访问见EXPLAIN |
| `/*+ GraphRoute(adj) */` | 同一公共执行器＋graph_adj 专用邻接存储/provider，使用稳定引用访问后续邻接 |

- 强制 hint 的规划路线独立，图路线不先完成传统多表 JOIN 搜索再替换计划。规划收益与执行收益分别报告。
- 逐层端点去重仅用于已证明安全的集合语义；不能把完整路径、bag 或 TRAIL 查询偷偷换成终点集合。当前准入的标准 PGQ 前端范围不能从底层 OUT/IN provider 测试反推为全部图语法支持。
- graph_adj 仍走 PG buffer/WAL/MVCC，不是整个图常驻内存、所有访问都免回表。覆盖索引免边回表已合入，但只在用户提供合适索引且可见性条件满足时启用；不自动创建索引，未建覆盖索引就不会凭空新增该索引的写维护。
- 可选组合明确称为 **“GraphExpandFixed 公共执行器＋AGE 内存邻接”**，不是“图 B-tree 索引＋AGE”、也不是调用 AGE VLE。正式 GUC `enable_graph_age_adjacency` 为 superuser 级、默认 **off**，不是 Cypher 总开关；不再使用实验参数 `test_graph_core.visible_adj`。

```sql
LOAD 'age';
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;
SET LOCAL enable_graph_age_adjacency = on;
-- 执行原来的 GraphRoute(btree) PGQ SQL；实际访问标识见 EXPLAIN ANALYZE：
-- Graph Visible Adjacency: AGE transaction-private adjacency (experimental)
COMMIT;
```

AGE 组合当前要求实际 superuser、非 security-definer、RR READ ONLY、无既有写入/子事务、无 RLS/继承的永久 heap，以及已准入的固定 OUT WALK 端点集合。首次执行从同一 PGQ 原表快照装载顶点和边，事务内热复用，提交/回滚释放；其他会话提交不改变当前 RR 快照，下个事务重新构建。不是边读边补的跨事务全局缓存。仅 `LOAD 'age'` 不启用；显式启用却缺兼容 provider 或不满足准入条件时报错，不静默换路线。普通图访问在 GUC off 时不依赖 AGE。

### 7.4 构建、回归与数据保护

下面是既有环境的执行入口。新机器先核对脚本中的本机路径、source lock 和依赖，按远端版本准备独立源码及实例；仅复制这些命令不能代替环境配置。

服务端口为 **55674**，回归使用 **55675/55677**；先检查实际占用和使用该安装的全部进程。按需要运行下列已有脚本，不另造 configure 参数，也不使用 `.install/pg19-pgorca-daft` 编译本特性：

```bash
GRAPH_TEST="$CPP_WORKSPACE/公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/golden/G1-080"
python3 "$GRAPH_TEST/pgq_integrated_build_v1.py" --jobs 6
python3 "$GRAPH_TEST/pgq_integrated_accept_v1.py" modules
python3 "$GRAPH_TEST/pgq_integrated_accept_v1.py" pg
python3 "$GRAPH_TEST/pgq_integrated_accept_v1.py" age
python3 "$GRAPH_TEST/pgq_integrated_accept_v1.py" guc
python3 "$GRAPH_TEST/pgq_integrated_accept_v1.py" age-semantics
python3 "$GRAPH_TEST/pgq_integrated_validate_v1.py" --suite routebatch
python3 "$GRAPH_TEST/pgq_integrated_validate_v1.py" --suite cover
python3 "$GRAPH_TEST/pgq_integrated_accept_v1.py" adj-safety
```

- 构建脚本直接使用两份主线源码，重编配套测试模块和 AGE；验收脚本使用隔离实例。既有基础回归为 PG **245/245**、AGE **43/43**、图模块 **13/13**；GUC、快照/权限、批访问、覆盖索引、adj 故障恢复与并发专项证据见性能文档 §2.19～§2.20，重叠集合不相加。
- 原始数据与证据保留。`graph-bench-data/graphroute-all-base-v1/p0l/data`、`graph-bench-data/pgq-leafprep-recovery-v1/data` 等种子只克隆到新隔离目录，**不直接启动种子、不覆盖旧结果**。失败数据库现场也不是可用种子；复测脚本、种子版本和页格式按当前报告选取。
- 结束后停止本轮自有实例、保留数据，并恢复其他服务初始状态。文档、基准脚本和证据部分在源码 Git 仓外；两仓 clean 不代表这些工作区资产已提交或有备份。

### 7.5 最新阶段性能与解释边界

2026-10-21最新续做见[阶段报告第9节](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/图Btree组合执行M0-M6验收报告.md#9-三项未签收问题的后续验证)：单链/连续/稀疏各12轮中4/8/5有效，B-tree执行7.76/5.14/2.21×，adj7.13/1.83/1.87×，公共执行器＋AGE热态13.54/10.02/15.13×，规划和执行分列。前轮高汇聚5/12有效、B-tree/adj执行5093×/4565×未在本轮重测。自然CBO全面准确、DML严格零退化仍未签收；73728实际提交事务及假对照的全部区间见报告，旧+27.8%不删除。以下保留为10月6日历史基线，不覆盖旧数字或跨日期拼表。

2026-10-06 归并后回归：四场景均 **10 万边、10 跳、热态、Release/O2**。下表为相对同场景 relational 的**纯执行加速比**，详细计时和轮次汇总见性能文档 §2.20；不是全场景保底收益。

| 场景 | 图 B-tree | graph_adj | 公共执行器＋AGE 内存邻接热态 |
|---|---:|---:|---:|
| 平行边重复路径，终点集合 1 行 | 5420.04× | 4696.14× | 未测 |
| 无重复单链，1 行 | 8.18× | 7.09× | 14.80× |
| 连续键多分支树，100 行 | 5.88× | 2.28× | 10.91× |
| 稀疏键同构树，100 行 | 2.37× | 2.06× | 16.14× |

- 第一行确有不同边 ID 的平行边，路径组合大量重复；主要收益来自逐层端点集合减少工作量，不是索引读取快几千倍。后三例没有重复遍历收益；单链访问 10 条边，两棵树各访问 910 条边、输出 100 行。
- 两棵树拓扑相同，规模为 `1→10→100→100…`；不是每一跳持续指数扩张。稀疏键指顶点 ID 的数值分布/置换，不是图更稀疏；不要把键局部性、多分支、高汇聚和高扇出混成一个维度。
- 规划约 **1571～1899×** 的收益对应本次强制 hint、10 跳/21 个关系、GEQO off、展开限制等特定配置，不是 PG 默认 CBO 的通用倍率，更不能计入“纯执行加速”。AGE 首次完整执行含装载约 **66～67 ms**，热态倍率只适用于同一受限事务内复用。
- 公平复测须同数据、同 SQL、同结果语义，只换 hint/被测开关；使用完整结果校验、`EXPLAIN (ANALYZE, TIMING OFF, FORMAT TEXT)`，规划/执行分列。预热并采用镜像配对轮次，按既定规则排除含换页/major fault、查询物理读或 spill 的完整污染轮；不能只挑快轮或用不同日期孤立数字拼倍率。火焰图只解释 CPU 去向，不代替无采样计时。
- 10月6日当时未测 DML 吞吐非劣化、新四组合独立迁移或自然 CBO 全面选优；本轮四组合及普通事务/权限专项见最新报告，DML吞吐/生产门仍未通过。adj在这些历史非重复场景未胜图 B-tree，不承诺所有图查询10×。

面向汇报的场景与两篇 HANA 论文对比见[主要测试场景简表](公共文档/PostgreSQL19原生多跳图执行与AGE适配/报告/主要测试场景简表.md)。比较时区分 relational 基线与 FI 聚簇/不聚簇内部比较，不把不同语义、数据或计时范围强行对齐。

### 7.6 历史实验已归档：不要沿旧路径接着开发

2026-10-08 已验证归档并清理 **20 组历史 pgq 实验的源码/构建/安装，共 60 个目录**；`graph-experiments/pgq-*` 和对应旧安装不再是可直接运行的入口。当前主线、`pgq-integrated`、种子、原始证据及 FlameGraph 工具均保留。

- 归档目录：`公共文档/PostgreSQL19原生多跳图执行与AGE适配/执行包/evidence/archives/20261008T064745Z-graph-history/`；其中 `historical-experiments.tar.zst`、文件清单、Git 补丁及 `summary.json` 保存恢复证据，状态为 `CLEANUP_COMPLETE`。
- 恢复须先读 [graph-experiments/README.md](graph-experiments/README.md)，向新的空目录完整解包并核验，再决定是否恢复特定实验；保留原权限，注意跨目录硬链接和旧构建的绝对路径。不要直接解包覆盖当前主线或重跑清理脚本。
- 此次净释放约 **5.61 GiB 的 Linux 文件系统空间**，不代表 Windows VHDX 已缩小；历史文档中的旧路径/提交身份保留作证据，不能因其不存在就重新生成并覆盖原测量。
