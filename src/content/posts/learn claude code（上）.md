---
title: learn claude code（上）
published: 2026-10-12
description: '从零开始学习coding agent的搭建'
image: ''
tags: [agent, LLM, harness, python]
category: '学习'
draft: false
lang: 'zh_CN'
---

# 简介

该项目围绕一条主线：**让LLM负责决策，让 Harness 负责提供能力、维护状态、约束执行和保障运行**

项目的大致主题可以分成三个部分：

1. 执行内核（s01~s04）：让agent动起来，建立 agent loop、工具调用、权限控制和扩展机制，保证harness**没有二次架构**
2. 上下文工程（s05~s09）：管理任务状态、子任务上下文、知识加载、上下文压缩和长期记忆，解决**信息丢失、上下文污染与信息冗余**
3. 系统工程（s10~s17）：引入任务持久化、异步执行、定时调度、多 Agent 协作、MCP、工作流编排与目标验收，使 Agent 从短时单体执行扩展为可持续、可协调、可验证的**系统**

# 第一部分-执行内核

## s01-agent loop

从组成上agent = LLM + harness，从流程上最简单最原始的agent就是一个while true循环

agent会不断重复“LLM调用工具 -> harness执行 -> 追加工具调用结果 -> 再调用”的过程，直到LLM不调用工具为止，整理成代码得到：

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        tool_calls = [
            block for block in response.content if block.type == "tool_use"
        ]
        if not tool_calls:
            return

        results = []
        for block in tool_calls:
            output = run_bash(block.input["command"])
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": output,
            })
        messages.append({"role": "user", "content": results})
```

## s02-tool use

虽然只要给LLM提供shell就能完成绝大多数操作，但是需要LLM自己完成从意图到操作的转换，这一步就可能造成token的浪费：

- 想要读一个大文件，但是没有加`head -n 1000`限制行数，导致一个10万行的文件直接塞进上下文
- 命令拼错，导致结果不对，又要重复进行一次LLM调用
- 不同环境shell环境也可能不同（powershell，bash，zsh，fish....），教LLM怎么适应不同shell环境要浪费额外的上下文

所以工具就是给agent提供的一个规范化完成操作的函数，LLM想要完成什么特定操作，只要调用对应的工具给出合适的参数就行

一个agent大概率不会只有一个工具，有的甚至会有很多工具，为了便于管理工具，需要建立一个工具分发表：

```python
# 给LLM看
TOOLS = [
    {"name": "bash",       "description": "Run a shell command.", ...},
    {"name": "read_file",  "description": "Read file contents.",  ...},
    {"name": "write_file", "description": "Write content to file.", ...},
    {"name": "edit_file",  "description": "Replace text in file once.", ...},
    {"name": "glob",       "description": "Find files by pattern.", ...},
]

# 从工具名到真实函数的映射
TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  run_read,
    "write_file": run_write,
    "edit_file":  run_edit,
    "glob":       run_glob,
}

def run_read(path, limit=None):
    lines = safe_path(path).read_text(encoding="utf-8").splitlines()
    if limit:
        lines = lines[:limit]
    return "\n".join(lines)

def run_write(path, content):
    safe_path(path).write_text(content, encoding="utf-8")
    return f"Wrote {len(content)} bytes to {path}"

def run_edit(path, old_text, new_text):
    text = safe_path(path).read_text(encoding="utf-8")
    if old_text not in text:
        return "Error: text not found"
    safe_path(path).write_text(text.replace(old_text, new_text, 1), encoding="utf-8")
    return f"Edited {path}"

def run_glob(pattern):
    import glob as g
    matches = sorted(set(g.glob(
        pattern, root_dir=WORKDIR, recursive=True)))
    shown = matches[:200]
    if len(matches) > 200:
        shown.append("... (more matches omitted; narrow the pattern)")
    return "\n".join(shown)

# 循环里只改了一行——从硬编码 run_bash 变成查表：
for block in tool_calls:
    handler = TOOL_HANDLERS[block.name]    # 查表
    output = handler(**block.input)         # 调用
    results.append(...)
```

## s03-permission

现在LLM可以通过harness对用户本地环境进行操作，必然有些操作是敏感或破坏性的，所以在工具执行前需要人为的用代码进行安全边界审查，审查分三步：

1. 硬边界：危险操作`rm -rf /`、`reboot`、`poweroff`、`sudo`等，直接拒绝
   ```python
   DENY_LIST = [
       "rm -rf /", "sudo", "shutdown", "reboot",
       "mkfs", "dd if=", "> /dev/sda",
   ]
   
   def check_deny_list(command: str) -> str | None:
       for pattern in DENY_LIST:
           if pattern in command:
               return f"Blocked: '{pattern}' is on the deny list"
       return None
   ```

2. 软边界：对操作进行规则匹配，如果命中交给用户判断，未命中直接放行
   ```python
   import re
   
   DESTRUCTIVE_COMMAND_WORD = re.compile(
       r"(?i)(?:^|[;&|()\n])\s*(?:rm|del)(?=\s|$|[;&|()])"
   )
   
   def contains_destructive_command(command: str) -> bool:
       return bool(DESTRUCTIVE_COMMAND_WORD.search(command))
   
   PERMISSION_RULES = [
       {
           "tools": ["read_file", "write_file", "edit_file"],
           "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
           "message": "Access outside workspace",
       },
       {
           "tools": ["bash"],
           "check": lambda args: contains_destructive_command(args.get("command", "")) or any(
               kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]
           ),
           "message": "Potentially destructive command",
       },
   ]
   
   def check_rules(tool_name: str, args: dict) -> str | None:
       for rule in PERMISSION_RULES:
           if tool_name in rule["tools"] and rule["check"](args):
               return rule["message"]
       return None
   ```

3. 用户审查：人工判断第二步命中的操作是否合理，自行选择是否放行
   ```py
   def ask_user(tool_name: str, args: dict, reason: str) -> str:
       print(f"\n⚠  {reason}")
       print(f"   Tool: {tool_name}({args})")
       choice = input("   Allow? [y/N] ").strip().lower()
       return "allow" if choice in ("y", "yes") else "deny"
   ```

完整的判断过程为：

```py
def check_permission(block) -> bool:
    # 闸门 1: 硬拒绝
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            print(f"\n⛔ {reason}")
            return False

    # 闸门 2 + 3: 规则匹配 → 用户审批
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False

    return True

# 在 agent_loop 中——s02 的循环只加了一行：
for block in tool_calls:
    if not check_permission(block):           # ← 新增
        results.append({... "content": "Permission denied."})
        continue
    output = TOOL_HANDLERS[block.name](**block.input)  # s02 原有
    results.append(...)
```

## s04-hooks

出于对代码维护的考虑，当我们想要对agent的功能进行扩展时，不能对loop的流程代码本身进行直接扩展，而是通过hook的方式来扩展

也就是说，在现在的loop中，循环和权限逻辑完全保留。唯一的变动是把 `check_permission()` 从循环体内移到了hook上，循环不再直接调用任何检查函数，改为 `trigger_hooks("PreToolUse", block)`，由注册表决定跑什么

四个事件，覆盖一个完整的 agent cycle：

| 事件             | 触发时机                    | 典型用途                            |
| ---------------- | --------------------------- | ----------------------------------- |
| UserPromptSubmit | 用户输入提交后、进入 LLM 前 | 输入验证、注入上下文                |
| PreToolUse       | 工具执行前                  | 权限检查、日志记录                  |
| PostToolUse      | 工具执行后                  | 副作用（自动 git add 等）、输出检查 |
| Stop             | 循环即将退出时              | 收尾清理、决定是否继续循环          |

全局有一个hooks注册表：

```py
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # 返回值 ≠ None → hook 说"停"
            return result
    return None
```

想要添加什么功能，就实现对应函数并通过`register_hook`往注册表里加，现在agent loop主循环里永远只有四个地方增添了`trigger_hooks`函数，这样不论扩展什么功能，在代码层面上都不会再改动agent loop主循环，例如权限检查现在就变为了：

```py
# PreToolUse: 权限检查（s03 的逻辑，从循环移到 hook）
def permission_hook(block):
    if block.name == "bash":
        for pattern in DENY_LIST:
            if pattern in block.input.get("command", ""):
                return "Permission denied by deny list"
    if block.name in ("read_file", "write_file", "edit_file"):
        path = block.input.get("path", "")
        if not (WORKDIR / path).resolve().is_relative_to(WORKDIR):
            choice = input("   Allow? [y/N] ").strip().lower()
            if choice not in ("y", "yes"):
                return "Permission denied by user"
    return None

# PreToolUse: 日志
def log_hook(block):
    print(f"[HOOK] {block.name}(...)")

# PostToolUse: 大文件提醒
def large_output_hook(block, output):
    if len(str(output)) > 100000:
        print(f"[HOOK] ⚠ Large output from {block.name}")

register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
register_hook("PostToolUse", large_output_hook)

# agent loop
for block in tool_calls:
    # s03: if not check_permission(block): ...
    # s04: hook 替代硬编码
    blocked = trigger_hooks("PreToolUse", block)
    if blocked:
        results.append({"type": "tool_result", "tool_use_id": block.id,
                        "content": str(blocked)})
        continue

    handler = TOOL_HANDLERS.get(block.name)
    output = handler(**block.input) if handler else f"Unknown: {block.name}"

    trigger_hooks("PostToolUse", block, output)

    results.append({"type": "tool_result", "tool_use_id": block.id,
                    "content": output})
```

# 第二部分-上下文工程

## s05-TodoWrite

完成第一部分后，我们已经保证了agent的执行功能完备，但agent执行是一个多步完成的过程，在这个过程中LLM是有可能“忘记”一些事情的，比如在任务不断执行的过程中，中间过程占用了LLM大量的注意力，导致LLM忘记了用户最初的需求

这里的问题不是agent缺乏执行能力，而是**任务目标与当前进度没有得到持续、显式的维护**

因此，现在要引入TodoWrite：

- 使用结构化TODO记录任务及其状态
- 通过 `pending`、`in_progress`、`completed` 跟踪进度
- 一定轮次未更新TODO时，由Harness注入提醒

相当于把原本依赖模型从对话中自行回忆的任务状态，提取出来交给harness管理

现在agent保留了工具分发，权限检查和hooks的基本机制，加入了`todo_write`和`reminder`计数器，`todo_write`只更新计划状态，实际工作仍由原有工具完成，当连续三个工具调用都没有使用`todo_write`时（也就是原计划流程连续三轮没有推进），harness把`reminder`追加到第三轮工具调用结果中

```py
TOOLS = [
    {"name": "bash",       ...},
    {"name": "read_file",  ...},
    {"name": "write_file", ...},
    {"name": "edit_file",  ...},
    {"name": "glob",       ...},
    # s05: 新增一条
    {"name": "todo_write", "description": "Create and manage a task list ...",
     "input_schema": {
         "type": "object",
         "properties": {
             "todos": {
                 "type": "array",
                 "items": {
                     "type": "object",
                     "properties": {
                         "content": {"type": "string"},
                         "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]},
                     },
                 },
             },
         },
     },
    },
]

TOOL_HANDLERS["todo_write"] = run_todo_write

# 持有内存中的任务列表，负责校验更新，并把渲染结果返回给模型
class TodoManager:
    def __init__(self):
        self.items = []

    def update(self, todos: list | str) -> str:
        # Parse and validate before replacing the current list.
        validated = []
        ...
        self.items = validated
        return self.render()

    def render(self) -> str:
        # [ ] pending, [>] in progress, [x] completed
        ...


TODO = TodoManager()

# 把TodoManager维护的状态打印到终端
def run_todo_write(todos: list | str) -> str:
    output = TODO.update(todos)
    print(output)
    return output

# reminder机制
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.append({
        "type": "text",
        "text": "<reminder>Update your todos.</reminder>",
    })
    rounds_since_todo = 0
```

## s06-subagent

现在为止，所有的消息都存储在一个`message[]`列表中，虽然可以确保LLM在大量上下文中依然按照原定流程执行，但是大量无关上下文的存在会导致token的浪费，例如：

- 让agent进行认证模块的重构，为了弄清某个函数的调用关系，agent 可能阅读几十个文件，得到数万token的工具输出。但主agent真正需要的可能只是：“该函数有三个调用位置，分别位于 A、B、C。”
- 如果把整个调查过程放进主上下文，大多数信息在完成调查后已经没有价值，却仍然持续占用token

所以现在agent需要一个委托机制，这个机制的核心是对**上下文进行隔离**

主agent将接下来要做的事情总结为一个task，将task交付给subagent，这个task完成的全过程都是在一个新的`message[]`中完成，最后subagent完成task后只会返回主agent需要的那部分信息，从而达到节省上下文空间的目的：

```py
SUB_TOOLS = list(BASE_TOOLS)  # no task tool

TASK_TOOL = {
    "name": "task",
    "description": "Run a subagent with fresh conversation context and return its final text.",
    "input_schema": {
        "type": "object",
        "properties": {"prompt": {"type": "string"}},
        "required": ["prompt"],
    },
}

TOOLS = [*BASE_TOOLS, TASK_TOOL]
TOOL_HANDLERS = {**BASE_HANDLERS, "task": run_subagent}

def run_subagent(prompt: str) -> str:
    messages = [{"role": "user", "content": prompt}]

    for _ in range(30):
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=SUB_TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        tool_calls = [
            block for block in response.content if block.type == "tool_use"
        ]
        if not tool_calls:
            return extract_text(response.content) or "(no summary)"

        results = []
        for block in tool_calls:
            output = execute_tool(block, SUB_HANDLERS)
            results.append({... "content": output})
        messages.append({"role": "user", "content": results})

    return "Subagent stopped after 30 turns without a final answer."
```

## s07-skill

前两章重点关注在agent工作中如何组织信息，这一章开始关注：**在模型执行任务之前，该给它多少知识**

我们可以简单的把所有知识塞进system prompt里， 但是很多时候agent只是在完成某一个特定领域的工作，其它领域的知识反而造成了不必要的浪费

所以从这一章开始agent引入了skill和渐进式披露：

1. 首先只把skill的名字和简介放进system prompt
2. LLM根据当前任务判断需要哪个skill
3. 调用`load_skill`，将完整的skill.md以tool result形式注入上下文

```py
# skill目录
skills/
  agent-builder/SKILL.md
  code-review/SKILL.md
  mcp-builder/SKILL.md
  pdf/SKILL.md

# skill加载器
class SkillLoader:
    def scan(self):
        self.skills.clear()
        skills_root = self.skills_dir.resolve()
        for manifest in sorted(self.skills_dir.glob("*/SKILL.md")):
            if (not manifest.is_file()
                    or not manifest.resolve().is_relative_to(skills_root)):
                continue
            content = manifest.read_text(encoding="utf-8")
            metadata, body = self.parse_frontmatter(content)
            raw_name = metadata.get("name")
            name = raw_name.strip() if isinstance(raw_name, str) else ""
            name = name or manifest.parent.name
            raw_description = metadata.get("description")
            description = (raw_description.strip()
                           if isinstance(raw_description, str) else "")
            description = description or body.split("\n", 1)[0]
            description = " ".join(str(description).lstrip("# ").split())
            self.skills[name] = {
                "name": name,
                "description": description,
                "content": content,
            }
            
# 组装system prompt
ENVIRONMENT_PROMPT = (
    "Windows: the bash tool runs through cmd.exe; use cmd.exe syntax, not Unix "
    "Bash or PowerShell syntax, and prefer dedicated file tools for file operations"
    if os.name == "nt"
    else "Unix-like: the bash tool runs the system shell"
)

def build_system_prompt() -> str:
    return (
        f"You are a coding agent at {WORKDIR}. Environment: {ENVIRONMENT_PROMPT}. "
        "Use tools to solve tasks. "
        "Act, don't explain.\n\n"
        f"Skills available:\n{SKILL_LOADER.catalog()}\n\n"
        "Use load_skill to read the full instructions when a skill applies."
    )
    
# 加载skill完整内容
def load(self, name: str) -> str:
    skill = self.skills.get(name)
    if skill:
        return skill["content"]
    available = ", ".join(self.skills) or "none"
    return f"Error: Unknown skill '{name}'. Available: {available}"
```

## s08-context compact

虽然我们通过一些方式尽量的减慢了上下文的增长，但是agent在运行过程中上下文的增长无法避免，终有一刻上下文会耗尽，因此我们必须对agent上下文窗口进行管理，这就引入了上下文压缩的概念

learn claude code采用一个四层pipeline架构进行上下文窗口压缩：

1. `tool_result_budget`
2. `snip_compact`
3. `micro_compact`
4. `compact_histrory`

pipeline整体的设计原则是优先执行低成本操作，优先保留可恢复信息，经过整理仍超限才调用api生成摘要

首先是整理工具调用结果，对coding agent而言各种工具的调用结果通常占有最多空间，我们把视角放到某一轮对话中工具调用的结果，如果这一次tool results过大，按以下策略进行处理：

- results合计超过200000字符时，按从大到小顺序处理
- 超过30000字符的result保存到磁盘
- 上下文中只保留2000字符的预览和完整文件路径

```py
blocks = [block for block in content
          if isinstance(block, dict)
          and block.get("type") == "tool_result"]
total = sum(len(str(block.get("content", ""))) for block in blocks)

ranked = sorted(
    blocks,
    key=lambda block: len(str(block.get("content", ""))),
    reverse=True,
)
for block in ranked:
    if total <= max_chars:
        break
    content = str(block.get("content", ""))
    if len(content) <= self.LARGE_RESULT_CHAR_LIMIT:
        continue
    block["content"] = self.persist_large_output(
        block.get("tool_use_id", "unknown"), content)
    total = sum(len(str(item.get("content", ""))) for item in blocks)
```

然后是视角转到整体`messages[]`的消息数量方面，当agent执行很多轮以后，即使每一轮上下文占用不多，累计起来消息也会越来越多，所以要对agent能看到的消息窗口大小进行管理，当消息总数超过50条时：

- 保留最初3条消息，目的是保留任务初始背景
- 保留最近46条消息，维持当前工作的连续性
- 将中间的完整消息归档存储到磁盘
- 用归档文件路径代替中间历史

```py
# 点需要保护 assistant(tool_use) 和 user(tool_result) 的配对关系
# 孤立的工具结果缺少对应调用，下一次 API 请求会被判定为无效
head_end = 3
tail_start = len(messages) - (max_messages - head_end - 1)

if self.has_tool_use(messages[head_end - 1]):
    while (head_end < tail_start
           and self.is_tool_result(messages[head_end])):
        head_end += 1

if (tail_start > 0
        and self.is_tool_result(messages[tail_start])
        and self.has_tool_use(messages[tail_start - 1])):
    tail_start -= 1

transcript = self.write_transcript(messages)
marker = {"role": "user", "content":
          f"[{tail_start - head_end} messages archived at {transcript}]"}
messages = [*messages[:head_end], marker, *messages[tail_start:]]

```

现在再来关注整个对话所有轮次的tool result，对agent来说工具调用的结果是有时效性的，有些早期的工具调用结果虽然留在上下文中，但是可能已经对现在的任务没有作用

那么此时经过两次整理后上下文长度仍然超过限制，那么那么就开始着手对更早的**已读取结果**进行替换，只保留最近3条，并逐条缩短更早且超过120个字符的结果，旧结果落盘后被替换为可恢复路径，整个过程持续到上下文缩减到阈值的80%

```py
unseen = self.unseen_tool_result_positions(messages)
consumed = [entry for entry in results if entry[:2] not in unseen]

for _, _, block in consumed[:-self.KEEP_RECENT_RESULTS]:
    if self.estimate_chars(messages) <= target_chars:
        break
    content = str(block.get("content", ""))
    if len(content) <= 120:
        continue
    saved_path = self.persisted_output_path(content)
    if not saved_path:
        saved_path = self.save_output(block["tool_use_id"], content)
    block["content"] = f"[Earlier tool result saved at {saved_path}]"
```

当然，压缩过程中需要优先保存LLM还未读取的结果，但是如果结果实在太大，也必须提前压缩，否则LLM可能无法进行下一次推理

最后，在经历了上述三轮处理后，如果字符数仍超过限制，那么现在就要调用api进行上下文压缩：

1. 将完整消息历史写入 `.transcripts/`。
2. 请求LLM生成只包含事实的状态摘要。
3. 将入口处捕获的当前用户请求与摘要明确分开。
4. 用一条 `[Compacted]` 消息替换当前历史。

```py
def compact_history(messages, active_request):
    transcript = self.write_transcript(messages)
    print(f"[transcript saved: {transcript}]")
    summary = self.summarize_history(messages)
    return [self.summary_message(
        "Compacted", active_request, summary, transcript)]
```

## s09-memory

当agent开始新对话时，我们希望有些在之前的对话中强调过的东西agent还能“记得”，就要引入持久化记忆功能

memory不能是简单的把完整对话副本存储并加载到新对话，这会造成上下文的急速膨胀，所以我们要解决两个问题：

1. 哪些信息值得跨会话保存
2. 当前任务应该需要哪些信息

和skill类似，memory也采用渐进式披露方式，不过skill由人主动编写并只读，而memory是agent从对话中提取内容，并在后续任务中再次使用

在根目录下有`MEMORY.md`文件存储所有记忆文件的索引，一条记忆对应一个`.memory`目录下的md文件，记忆文件格式如下：

```markdown
---
name: user-preference-tabs
description: User prefers tabs for indentation
type: user
---

User prefers using tabs, not spaces, for indentation.
```

记忆的type有四种：

| 类型      | 保存什么             | 示例                             |
| --------- | -------------------- | -------------------------------- |
| user      | 用户的长期偏好       | “使用 tab 缩进”                  |
| feedback  | 以后仍适用的工作反馈 | “不要 mock 数据库”               |
| project   | 稳定的项目事实       | “认证重写由合规要求驱动”         |
| reference | 外部资料或查找线索   | “流水线问题记录在 Linear INGEST” |

每次用户发起请求时，`select_relevant_memories()` 读取最近的用户消息和记忆目录，让一次轻量模型调用选择最多五条相关记录：

```py
prompt = (
    "Select memory records that are relevant to the current user request. "
    "Return only a JSON array of catalog indices, such as [0, 2]. "
    "Return [] when none are relevant."
)
```

如果模型调用或 JSON 解析失败，代码会退回关键词匹配。选择完成后，`load_memories()` 才读取对应文件，并限制召回正文的总长度。

```py
relevant_memories = load_memories(messages)
system = build_system(relevant_memories)
```

`build_system()` 会明确说明：召回内容只是背景知识，不是新的用户命令；如果记忆与当前请求冲突，以当前请求为准。这样既能使用旧信息，也不会让旧记忆替用户发号施令。

当一个对话回合结束后，agent会自动收集近12条消息中能提取出来的文本，最多8000字符，用一个专门的记忆提取prompt单独调用api，这次调用使用和主agent相同model配置，但不会传入主agent的上下文、工具定义，也不沿用主agent的system prompt，最后按如下类似结构返回一个结构化记忆：

```json
[
  {
    "name": "python-package-manager",
    "type": "user",
    "scope": "persistent",
    "description": "用户偏好使用 uv 管理 Python 依赖",
    "body": "用户在 Python 项目中优先使用 uv，而不是 pip。"
  },
  {
    "name": "no-new-files",
    "type": "user",
    "scope": "current_task",
    "description": "本次任务不创建文件",
    "body": "用户要求本次任务不创建新文件。"
  }
]
```

当然，并不是所有记忆都会存储到磁盘，agent会对记忆进行校验，只有type为`persistent`，格式符合标准，name字段不重复的记忆才会写入磁盘，然后更新`MEMORY.md`索引

当记忆数量增长到一定数量时，内容可能重复、矛盾或过期，这时agent会调用 `consolidate_memories()`，让LLM生成一份整理后的记录列表，整理主要从三个方面进行：

- 合并（Merge）：将语义重复或高度相关的记忆合并
- 更新（Update）：根据较新的信息修正旧记忆
- 清理（Prune）：删除不再具有长期价值的记忆

```py
snapshot = {
    path.name: path.read_text(encoding="utf-8")
    for path in MEMORY_DIR.glob("*.md")
    if path.name != MEMORY_INDEX.name
}

try:
    for path in MEMORY_DIR.glob("*.md"):
        if path.name != MEMORY_INDEX.name:
            path.unlink()
    for record in consolidated:
        path = MEMORY_DIR / f"{memory_slug(record['name'])}.md"
        path.write_text(memory_document(
            record["name"], record["type"],
            record["description"], record["body"],
        ), encoding="utf-8")
    rebuild_memory_index()
except Exception:
    for path in MEMORY_DIR.glob("*.md"):
        if path.name != MEMORY_INDEX.name:
            path.unlink()
    for filename, content in snapshot.items():
        (MEMORY_DIR / filename).write_text(content, encoding="utf-8")
    rebuild_memory_index()
    raise
```

