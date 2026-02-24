查阅 Claude Code Hooks 配置指南，根据用户需求配置合适的 hook。

## Claude Code Hooks 完整配置指南

---

### 一、Hook 是什么

Hook 是 Claude Code 在特定生命周期事件触发时自动执行的 shell 命令（或 LLM 提示）。用途：
- 自动格式化代码（PostToolUse）
- 阻止危险命令（PreToolUse）
- 保护敏感文件不被修改（PreToolUse）
- 运行测试（PostToolUse async）
- 桌面通知（Notification）

---

### 二、配置文件位置

| 位置 | 作用域 | 是否共享 | 是否入 git |
|------|--------|----------|-----------|
| `~/.claude/settings.json` | 所有项目（全局） | 否 | 否 |
| `.claude/settings.json` | 当前项目 | 是 | 是 |
| `.claude/settings.local.json` | 当前项目（私有） | 否 | 否 |

优先级：Local > Project > Global

---

### 三、完整配置结构

```json
{
  "hooks": {
    "<事件名>": [
      {
        "matcher": "正则表达式",
        "hooks": [
          {
            "type": "command",
            "command": "shell 命令",
            "timeout": 600,
            "statusMessage": "自定义等待提示文字",
            "async": false
          }
        ]
      }
    ]
  }
}
```

**字段说明：**
- `matcher`：正则匹配（区分大小写），空字符串或省略表示匹配全部
- `type`：`"command"`（shell 命令）、`"prompt"`（单轮 LLM 判断）、`"agent"`（多轮子代理验证）
- `timeout`：超时秒数（command 默认 600s，prompt 默认 30s，agent 默认 60s）
- `async`：设为 true 则后台运行不阻塞（仅 command 类型）
- `statusMessage`：hook 执行时的 spinner 提示文字

---

### 四、所有事件类型

| 事件 | 触发时机 | matcher 匹配对象 | 能否阻塞 |
|------|---------|-----------------|---------|
| `SessionStart` | 会话启动/恢复 | startup / resume / clear / compact | 否 |
| `UserPromptSubmit` | 用户提交提示词 | — | 是（exit 2） |
| `PreToolUse` | 工具调用前 | 工具名 | 是（exit 2 或 JSON deny） |
| `PermissionRequest` | 权限弹窗时 | 工具名 | 是 |
| `PostToolUse` | 工具成功后 | 工具名 | 否（已执行完毕） |
| `PostToolUseFailure` | 工具失败后 | 工具名 | 否 |
| `Notification` | 通知发出时 | 通知类型 | 否 |
| `SubagentStart` | 子代理启动 | 代理类型 | 否 |
| `SubagentStop` | 子代理结束 | 代理类型 | 是 |
| `Stop` | 主代理结束响应 | — | 是 |
| `TeammateIdle` | 团队成员空闲 | — | 是（exit 2） |
| `TaskCompleted` | 任务完成 | — | 是（exit 2） |
| `ConfigChange` | 配置变更 | 配置来源 | 是 |
| `WorktreeCreate` | 工作树创建 | — | 是 |
| `WorktreeRemove` | 工作树删除 | — | 否 |
| `PreCompact` | 上下文压缩前 | manual / auto | 否 |
| `SessionEnd` | 会话结束 | 退出原因 | 否 |

---

### 五、退出码与行为

| 退出码 | 含义 | 阻塞？ | stdout | stderr |
|--------|------|--------|--------|--------|
| **0** | 成功 | 否 | 解析 JSON（部分事件） | 仅 verbose 模式显示 |
| **2** | 阻塞错误 | **是** | 忽略 | 作为错误反馈给 Claude |
| **其他** | 非阻塞错误 | 否 | 忽略 | 仅 verbose 模式显示 |

---

### 六、stdin 输入数据

Hook 通过 **stdin 接收 JSON**，不是环境变量。用 `jq` 提取字段。

**通用字段（所有事件）：**
```json
{
  "session_id": "abc123",
  "cwd": "/当前工作目录",
  "hook_event_name": "PostToolUse",
  "transcript_path": "/path/to/transcript.jsonl",
  "permission_mode": "default"
}
```

**PreToolUse / PostToolUse 额外字段：**
```json
{
  "tool_name": "Edit",
  "tool_use_id": "toolu_01ABC...",
  "tool_input": {
    "file_path": "/path/to/file.py",
    "old_string": "...",
    "new_string": "..."
  },
  "tool_response": { "success": true }
}
```

**tool_input 因工具不同而异：**
| 工具 | 关键字段 |
|------|---------|
| Bash | `command`, `description`, `timeout` |
| Write | `file_path`, `content` |
| Edit | `file_path`, `old_string`, `new_string` |
| Read | `file_path`, `offset`, `limit` |
| Glob | `pattern`, `path` |
| Grep | `pattern`, `path`, `glob` |
| WebFetch | `url`, `prompt` |
| Task | `prompt`, `subagent_type`, `model` |

**提取示例：**
```bash
# 提取文件路径
file=$(jq -r '.tool_input.file_path')

# 提取 Bash 命令
cmd=$(jq -r '.tool_input.command')

# 提取工具名
tool=$(jq -r '.tool_name')
```

---

### 七、可用环境变量

| 变量 | 说明 | 可用范围 |
|------|------|---------|
| `$CLAUDE_PROJECT_DIR` | 项目根目录 | 所有 hook |
| `$CLAUDE_PLUGIN_ROOT` | 插件根目录 | 仅插件 hook |
| `$CLAUDE_ENV_FILE` | 环境变量持久化文件 | 仅 SessionStart |
| `$CLAUDE_CODE_REMOTE` | 远程 web 环境为 "true" | 所有 hook |

---

### 八、Matcher 正则语法

```
"matcher": "Edit|Write"          # 匹配 Edit 或 Write
"matcher": "Bash"                # 仅匹配 Bash
"matcher": "mcp__memory__.*"     # 匹配 memory MCP 的所有工具
"matcher": "mcp__.*__write.*"    # 匹配所有 MCP 的 write 类工具
"matcher": ""                    # 匹配全部（或省略 matcher 字段）
```

注意：**区分大小写**，`Bash` ≠ `bash`。

---

### 九、实战示例

#### 示例 1：Python 文件自动格式化（当前项目已配置）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'file=$(jq -r .tool_input.file_path); if [[ \"$file\" == *.py ]]; then /Users/asher/.local/bin/ruff format --quiet \"$file\" && /Users/asher/.local/bin/ruff check --fix --quiet \"$file\" 2>/dev/null; fi; exit 0'"
          }
        ]
      }
    ]
  }
}
```

#### 示例 2：阻止危险 Bash 命令

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'cmd=$(jq -r .tool_input.command); if echo \"$cmd\" | grep -qE \"rm -rf|drop table|truncate\"; then echo \"危险命令被拦截: $cmd\" >&2; exit 2; fi; exit 0'"
          }
        ]
      }
    ]
  }
}
```

#### 示例 3：保护敏感文件不被修改

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'file=$(jq -r .tool_input.file_path); for p in .env package-lock.json uv.lock; do if [[ \"$file\" == *\"$p\" ]]; then echo \"保护文件: $file\" >&2; exit 2; fi; done; exit 0'"
          }
        ]
      }
    ]
  }
}
```

#### 示例 4：macOS 桌面通知

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code 需要你的注意\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

#### 示例 5：后台异步跑测试

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'file=$(jq -r .tool_input.file_path); if [[ \"$file\" == *_test.py ]] || [[ \"$file\" == *test_*.py ]]; then cd \"$CLAUDE_PROJECT_DIR\" && python -m pytest \"$file\" -x -q; fi; exit 0'",
            "async": true,
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

#### 示例 6：SessionStart 注入环境变量

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'if [ -n \"$CLAUDE_ENV_FILE\" ]; then echo \"export PYTHONPATH=$CLAUDE_PROJECT_DIR\" >> \"$CLAUDE_ENV_FILE\"; fi; exit 0'"
          }
        ]
      }
    ]
  }
}
```

#### 示例 7：使用脚本文件（推荐复杂逻辑时使用）

**配置：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/format.sh"
          }
        ]
      }
    ]
  }
}
```

**脚本 `.claude/hooks/format.sh`：**
```bash
#!/bin/bash
FILE_PATH=$(jq -r '.tool_input.file_path')

case "$FILE_PATH" in
  *.py)
    ruff format --quiet "$FILE_PATH"
    ruff check --fix --quiet "$FILE_PATH" 2>/dev/null
    ;;
  *.js|*.ts|*.jsx|*.tsx)
    npx prettier --write "$FILE_PATH" 2>/dev/null
    ;;
  *.go)
    gofmt -w "$FILE_PATH"
    ;;
esac

exit 0
```

---

### 十、注意事项

1. **配置修改需重启会话**：改 settings.json 后，当前会话不一定立即生效，重启最可靠
2. **用绝对路径**：hook 的 shell 环境不一定有完整 PATH，工具路径用绝对路径更稳
3. **stderr 用于错误信息**：stdout 用于 JSON 返回，普通日志输出用 `>&2`
4. **stdout 必须是纯 JSON**：如果 `~/.bashrc` 有 echo 输出会干扰解析，需用 `[[ $- == *i* ]]` 包裹
5. **Stop hook 防死循环**：必须检查 `stop_hook_active` 字段，为 true 时直接 exit 0
6. **Matcher 大小写敏感**：`Bash` ≠ `bash`，`Edit` ≠ `edit`
7. **exit 0 = 放行，exit 2 = 阻塞**：其他退出码视为非阻塞错误
8. **async hook 无法阻塞**：后台运行的 hook 不能返回阻塞决策，结果下一轮才到达
