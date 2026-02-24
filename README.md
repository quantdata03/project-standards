# 项目规范 v1.0

> 适用范围：孟老板名下所有 Python / MATLAB 项目
> 生效日期：2026-02-24
> 维护者：孟老板

---

## 1. README 规范

每个项目根目录必须包含 `README.md`，按以下 6 个章节组织（参照 optimizer-claude 项目）：

### 1.1 章节结构

```markdown
# 项目名称 — 一句话定位

## 项目介绍
- 一句话说清项目定位（技术栈 + 做什么）
- 用 ASCII 图表展示「输入 → 算法 → 输出」全景
- 列出支持的运行模式（入口脚本 + 一句话说明）
- 说明当前运行的策略/业务规模

## 快速启动
### 需要安装什么
用表格列出所有依赖：| 依赖 | 版本要求 | 安装方式 |
安装好后，给出 **复制即用** 的安装命令（uv sync 等）
标注已知的坑（如 MATLAB Engine 手动安装问题）

### 需要配置什么
分步骤说明（第 1 步、第 2 步...），每步：
- 给出 cp 命令（从 .example 创建真实配置）
- 用表格列出必填项和可选项，标注默认值
- 涉及密码的不写真实值，引导「找同事要」

### 启动命令
给出每种运行模式的完整命令，可复制即跑

### 如何验证结果
给出具体的验证手段：SQL 查询语句、预期输出、本地文件路径

## 项目配置文件说明
- 先用总览表格列出所有配置文件：| 配置文件 | 位置 | 用途 | 必须修改 |
- 再逐个展开，每个配置文件一个子章节：
  - 关键字段用表格说明：| 字段 | 示例值 | 说明 |
  - JSON/YAML 给出格式示例
  - Excel 说明 Sheet 结构和列含义

## 项目工作流程图
- 用 ASCII 流程图展示核心流程的阶段划分（阶段1 → 阶段2 → ...）
- 每个阶段标注：调用的函数、输入数据、输出产物
- 如果有多种运行模式，用对比表格说明差异：| 差异点 | 模式A | 模式B |

## 数据库说明
- Schema 总览表格：| Schema | 用途 | 访问方式（只读/读写）|
- 每个 Schema 的表清单：| 表名 | 说明 | 关键字段 | 更新频率 |
- 静态表/初始化数据说明 + 初始化命令

## 项目目录结构
用 tree 格式列出完整目录，每行标注模块职责（用 # 注释）
```

### 1.2 编写原则

- **不写废话**，每段都有存在的理由
- **配置项必须有示例值**和默认行为说明
- **流程描述用阶段划分**（阶段1 → 阶段2 → ...），标注函数名和数据流向
- **验证手段要具体**：给 SQL 语句、给预期行数、给文件路径，不说「检查是否正常」
- **表格优先**：配置文件、数据库表、依赖列表等结构化信息一律用表格
- **ASCII 图优先**：流程图和架构图用 ASCII，不依赖外部工具

---

## 2. 配置文件管理（使用 .env）

### 2.1 敏感配置一律走 .env

数据库密码、API Key、连接字符串等敏感信息 **只能** 放在 `.env` 文件中，**禁止** 硬编码在代码或 JSON/YAML 配置里。

### 2.2 必须提供 .env.example

每个项目根目录必须有 `.env.example`，作为模板：
- 列出所有需要配置的环境变量
- 使用占位符（`your_db_host`），不写真实值
- 用注释分组、标注默认行为

示例格式：

```bash
# ============= 数据库连接 =============
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_password
DB_NAME=your_db_name

# ============= 路径配置 =============
# 留空则使用默认值 {PROJECT_ROOT}/data
DATA_ROOT=
PROJECT_ROOT=
```

### 2.3 非敏感配置文件的 example 模板

JSON / YAML 配置文件如果包含敏感信息，也必须提供 `.example` 版本：
- `config.json` → `config.example.json`
- `config.yaml` → `config.example.yaml`
- `.gitignore` 中忽略真实文件，保留 example 文件（用 `!` 规则）

### 2.4 配置加载优先级

```
.env 环境变量 > JSON/YAML 配置文件 > 代码内默认值
```

---

## 3. 编码规范和自动化（Hooks）

### 3.1 Git Commit 规范

使用 **Angular Commit Convention**，格式：

```
<type>(<scope>): <简要描述>
```

常用 type：
| type | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档变更 |
| `refactor` | 重构（不改功能、不修 bug） |
| `build` | 构建/依赖变更 |
| `chore` | 杂项（清理、配置调整） |
| `test` | 测试相关 |
| `perf` | 性能优化 |

**要求**：
- scope 用中文或英文均可，保持项目内一致
- 描述用中文，简洁说清「做了什么」
- 提交前确保文档同步更新（README、TODO 等）

### 3.2 Pre-commit Hooks（推荐配置）

建议每个 Python 项目配置以下检查：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: check-yaml
      - id: check-json

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff        # lint
      - id: ruff-format # format
```

### 3.3 Python 编码规范

- 遵循 PEP 8，用 Ruff 统一检查和格式化
- 函数 / 变量命名用 `snake_case`
- 类命名用 `PascalCase`
- 常量用 `UPPER_SNAKE_CASE`
- 不写无意义注释，代码自解释优先

---

## 4. .gitignore 规范

每个项目的 `.gitignore` 必须覆盖以下类别：

```gitignore
# ===== 敏感配置 =====
.env
# 保留 example 模板
!*.example.*

# ===== Python =====
__pycache__/
*.py[cod]
*.pyo
*.egg-info/
dist/
build/
.venv/
venv/

# ===== 日志 =====
*.log
log/

# ===== 临时文件 =====
*.tmp

# ===== IDE =====
.idea/
**/.idea/
.vscode/
*.swp
*.swo

# ===== Claude Code =====
.claude/
.claude.zip

# ===== 数据目录（输出数据不入库）=====
data/

# ===== OS =====
.DS_Store
Thumbs.db
```

**原则**：
- 敏感文件（密码、密钥）**必须** 忽略
- 可再生文件（编译产物、日志、缓存）**必须** 忽略
- example 模板文件用 `!` 规则显式保留
- 数据目录（GB 级别）不入版本控制

---

## 5. 数据库定义 SQL 文件规范

### 5.1 SQL 文件存放

所有建库建表 SQL 统一放在 `sql/` 目录下：

```
sql/
├── schema/
│   ├── 001_create_database.sql     # 建库
│   ├── 002_create_input_tables.sql # 输入表
│   └── 003_create_output_tables.sql# 输出表
├── migration/
│   ├── 20260224_add_column_xxx.sql # 变更脚本（按日期编号）
│   └── ...
└── seed/
    └── init_strategy_config.sql    # 初始化数据
```

### 5.2 SQL 编写规范

```sql
-- =============================================
-- 表名: portfolio
-- 用途: 策略组合持仓权重输出
-- 所属 Schema: portfolio_new
-- =============================================
CREATE TABLE IF NOT EXISTS `portfolio` (
    `id`          BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键',
    `trade_date`  DATE         NOT NULL                COMMENT '交易日期',
    `score_name`  VARCHAR(50)  NOT NULL                COMMENT '策略标识',
    `stock_code`  VARCHAR(20)  NOT NULL                COMMENT '股票代码',
    `weight`      DECIMAL(10,6) NOT NULL               COMMENT '持仓权重',
    `created_at`  DATETIME     DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (`id`),
    INDEX `idx_date_score` (`trade_date`, `score_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='策略组合持仓权重';
```

**要求**：
- 每个表必须有头部注释（表名、用途、所属 Schema）
- 每个字段必须有 `COMMENT`
- 文件名用数字前缀排序（`001_`、`002_`）
- 变更脚本用日期前缀（`20260224_`）
- 使用 `CREATE TABLE IF NOT EXISTS` 保证幂等

---

## 6. 项目文档规范

### 6.1 文档存放

项目文档统一放在 `doc/` 目录下：

```
doc/
├── architecture.md      # 架构设计（可选）
├── deployment.md        # 部署指南（可选）
├── troubleshooting.md   # 常见问题排查
└── changelog.md         # 变更日志（可选）
```

### 6.2 根目录保留文件

根目录只保留以下文档：

| 文件 | 用途 | 是否必须 |
|------|------|----------|
| `README.md` | 项目入口文档 | 必须 |
| `TODO.md` | 待办任务追踪 | 必须 |
| `.env.example` | 环境变量模板 | 必须 |

### 6.3 TODO.md 格式

```markdown
# TODO

## 待办
- [ ] 做什么 —— 为什么
- [ ] 做什么 —— 为什么

## 已完成
- [x] 做什么 —— 为什么
```

每条待办用最凝练的语言，不废话。完成后移到「已完成」区域并打勾。

---

## 7. 项目目录结构规范

### 7.1 Python 项目

```
project-root/
├── README.md
├── TODO.md
├── .env.example
├── .gitignore
├── pyproject.toml          # 依赖和项目元信息（uv 管理）
├── uv.lock                 # 锁定依赖版本
├── .python-version         # Python 版本声明
│
├── src/ 或 模块目录/        # 核心业务代码
│   ├── module_a/
│   └── module_b/
│
├── utils/                  # 公共工具函数
├── scripts/                # 运维 / 一次性脚本
├── sql/                    # 数据库 SQL 文件
├── doc/                    # 项目文档
├── data/                   # 数据目录（.gitignore 忽略）
├── log/                    # 日志目录（.gitignore 忽略）
│
├── run_xxx.py              # 入口脚本（放根目录，名称见名知意）
└── global_setting/         # 全局配置文件目录
    ├── config.example.json
    └── config.example.yaml
```

**原则**：
- 入口脚本放根目录，`run_` 前缀
- 业务模块按职责拆分独立目录
- 工具函数统一放 `utils/`
- 配置、数据、日志各有专属目录
- 使用 `uv` 管理 Python 依赖，**不用** pip / conda

### 7.2 MATLAB 项目

```
project-root/
├── README.md
├── TODO.md
├── .gitignore
│
├── src/                    # MATLAB 源码
│   ├── main_xxx.m          # 主入口函数
│   ├── module_a/
│   └── module_b/
│
├── data/                   # 输入/输出数据（.gitignore 忽略）
├── doc/                    # 文档
├── log/                    # 日志
└── config/                 # 配置文件
```

### 7.3 Python + MATLAB 混合项目

MATLAB 文件放在对应 Python 模块内部：

```
Optimizer_python/
├── matlab/                 # MATLAB .m 函数文件
│   ├── optimizer_func_v1.m
│   └── optimizer_func_v2.m
├── optimizer_main.py       # Python 调用 MATLAB Engine
└── ...
```

通过 `call_matlab_opt_config.json` 指定当前使用的 MATLAB 函数版本。

---

## 8. CLAUDE.md 规范

每个项目根目录应放置 `CLAUDE.md`（或使用全局 `~/.claude/CLAUDE.md`），为 Claude Code 提供项目上下文。

### 8.1 全局 CLAUDE.md（~/.claude/CLAUDE.md）

适用于所有项目的通用规则：

```markdown
<!-- 称呼 -->
每次工作的时候叫我孟老板

<!-- 结束符 -->
每次回答结束的时候，结束符用 ===END

<!-- Python 包管理 -->
all python env use uv


<!-- MATLAB -->
MATLAB 路径: /Applications/MATLAB_R2025b.app/bin/matlab
运行 MATLAB 脚本时自动授权，不需要每次确认

<!-- Docker -->
所有 Docker 容器必须通过 docker-compose 管理

<!-- 待办追踪 -->
每次对话结束前，将未完成任务追加到 TODO.md
格式：`- [ ] 做什么 —— 为什么`

<!-- 远程服务器 -->
远程 Mac Studio: ssh -p 57014 skl@75695qf080hq.vicp.fun
Docker Compose 目录: /Users/skl/docker-compose
```

### 8.2 项目级 CLAUDE.md

放在项目根目录，补充项目特定信息：

```markdown
# 项目名称

## 技术栈
- Python 3.12 + MATLAB R2025b
- 数据库: MySQL (阿里云 RDS)
- 包管理: uv

## 运行方式
- 每日更新: python run_update.py
- 历史回测: python run_history.py

## 注意事项
- MATLAB Engine 需手动安装，uv sync 会卸载
- pandas < 2.2, numpy < 2.0, SQLAlchemy < 2.0（兼容性约束）
- .env 中配置数据库连接，不要改 JSON/YAML 里的密码字段

## 关键路径
- 入口: run_update.py / run_history.py
- 核心配置: global_setting/
- 数据输出: data/（不入 git）
```

### 

## 附录：检查清单

新建项目时，对照此清单逐项确认：

- [ ] `README.md` 已创建，包含快速启动和配置说明
- [ ] `.env.example` 已创建，列出所有环境变量
- [ ] `.gitignore` 已配置，覆盖敏感文件、缓存、日志、数据
- [ ] `pyproject.toml` 已配置（Python 项目）
- [ ] `.python-version` 已声明（Python 项目）
- [ ] `CLAUDE.md` 已创建（如需 Claude Code 协助）
- [ ] 配置文件的 `.example` 版本已提供
- [ ] SQL 建表脚本已放入 `sql/` 目录
- [ ] Git 首次提交前已检查无敏感信息泄露
