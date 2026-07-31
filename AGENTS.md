# TUIDO - 终端看板工具

基于终端的 Kanban 看板 TUI，用于管理 TODO.md 文件。支持本地看板管理和飞书多维表格双向同步。

## 项目概览

- **语言**: Python 3.12+
- **框架**: Textual (TUI 框架)
- **入口**: `tuido [tui|create|add|pick|list|push|pull] [options]`
- **路径参数**: `--path` 可选，默认为当前目录 `.`
- **配置**: `pyproject.toml` (hatchling 打包)
- **版本**: 0.1.0

## 架构

```
tuido/
├── __init__.py           # 包版本 (0.1.0)
├── main.py               # CLI 入口 (Click)，命令组定义、子命令注册
├── models.py             # 数据模型: Task, Board, FeishuTask, RemoteConfig, GlobalConfig
├── parser.py             # TODO.md 读写逻辑，front matter 解析，任务内容解析
├── ui.py                 # TUI 实现: TaskCard, KanbanColumn, KanbanBoard, TuidoApp
├── cmd_list.py           # list 命令实现（本地 + 远程）
├── cmd_add.py            # add 命令实现
├── cmd_pick.py           # pick 命令实现
├── util.py               # 工具函数（文件查找、时间戳解析）
├── feishu.py             # 飞书 API 封装（FeishuTable 类、fetch_tasks）
├── config.py             # 全局配置加载/保存 (~/.config/tuido/config.yaml)
├── cmd_tui.py            # tui 命令实现（本地 + 全局视图）
├── cmd_create.py         # create 命令实现（创建示例 TODO.md）
├── cmd_push.py           # push 命令实现（推送到飞书，本地 + 全局）
└── cmd_pull.py           # pull 命令实现（从飞书拉取到本地）
```

### 数据流

```
TODO.md ──parse──▶ Board ──render──▶ TUI (KanbanBoard)
    ▲                                   │
    │                                   ▼
  save ◀────────── 编辑/移动任务 ◀──────┘
    │
    ▼
  push ──▶ 飞书多维表格
    ▲
    │
  pull ◀── 飞书多维表格
```

## 数据格式 (TODO.md)

**栏目(Column)是动态的，由二级标题决定**。TODO.md 中的每个 `## ` 标题成为一个看板列，按文件中出现的顺序展示。

### Front Matter 配置

TODO.md 支持 YAML front matter 格式配置，放在文件开头：

```markdown
---
theme: atom-one-dark
remote:
  feishu_api_endpoint: https://open.feishu.cn/open-apis
  feishu_table_app_token: xxx
  feishu_table_id: yyy
  feishu_table_view_id: zzz
  feishu_bot_app_id: aaa
  feishu_bot_app_secret: bbb
---
```

> **注意**: `remote` 配置也支持无前缀的简化格式（`api_endpoint`, `table_app_token`, `table_id`, `view_id`），代码中两种格式均可识别。

### 示例文件

```markdown
## Todo
- 任务描述 #标签 !优先级

## Active
- 进行中的任务 #功能 !P1

## Review
- 待审核的任务

## Done
- 已完成的任务
```

### 动态栏目
- 栏目由 `## 标题` 自动读取，无需修改代码
- 可在文件中定义任意数量的栏目
- 栏目顺序遵循文件中的出现顺序
- 空栏目也会被保留（不被删除）

### 元数据语法
- `#标签` - 标签 (如 #功能, #缺陷)
- `!优先级` - 优先级 (如 !P0, !P1, !P2, !P3, !P4，P0 最高)
- `~YYYY-MM-DDTHH:MM` - 时间戳 (如 ~2026-02-28T14:30，自动更新)
- `「项目名」` - 项目名称（用于全局视图显示）

### 行内样式
任务标题支持 Markdown 风格的行内格式，在 TUI 中会渲染为对应样式：

- `**加粗**` 或 `__加粗__` → **加粗文本**
- `` `代码` `` → `代码文本` (青色)
- `~~删除线~~` → ~~删除线文本~~

样式由 `ui.py` 中的 `parse_inline_styles()` 函数处理，支持嵌套样式（如 `**bold with `code` inside**`）。

示例：
```markdown
- 实现 **加粗** 和 `代码` 样式支持 !P1 #ui
- ~~废弃功能~~ 将在 v2.0 中移除
- **重要:** 运行前检查 `config.yaml`
```

## 核心类与模块

### models.py

- **`Task`**: 单个任务数据模型
  - `title`: 任务标题（含元数据解析前的原始文本）
  - `column`: 栏目名称（对应 TODO.md 中的二级标题）
  - `tags`: 标签列表
  - `priority`: 优先级 (P0-P4)
  - `project`: 项目名称（用于全局视图显示 `「项目名」`）
  - `updated_at`: 最后更新时间，格式 `YYYY-MM-DDTHH:MM`

- **`Board`**: 看板数据模型
  - `title`: 看板标题
  - `columns`: 有序字典 `{栏目名: [Task, ...]}`，保持插入顺序
  - `settings`: 从 front matter 解析的设置字典
  - 方法: `get_tasks_by_column()`, `get_all_tasks()`, `reorder_task()`, `move_task_to_column()`, `delete_task()`, `add_task()`
  - 类方法: `from_feishu_records()` - 从飞书记录创建 Board

- **`FeishuTask`**: 飞书同步用的扁平化任务模型
  - `title`, `project`, `status`, `tags`, `priority`, `timestamp`

- **`RemoteConfig`**: 飞书远程配置
  - 字段: `feishu_api_endpoint`, `feishu_table_app_token`, `feishu_table_id`, `feishu_table_view_id`, `feishu_bot_app_id`, `feishu_bot_app_secret`
  - 方法: `from_yaml(path)`, `is_valid()`, `get_missing_fields()`

- **`GlobalConfig`**: 全局配置
  - `theme`: 主题名称
  - `remote`: `RemoteConfig` 实例
  - 方法: `from_yaml(path)`, `save(path)`

### parser.py

- `parse_front_matter(lines)` - 解析 YAML front matter，返回 `(settings_dict, line_index)`。支持嵌套块（2空格缩进）
- `parse_task_content(content)` - 从任务行解析元数据（时间戳、项目、标签、优先级）
- `parse_todo_file(file_path)` - 完整解析 TODO.md 文件为 `Board` 对象
- `save_todo_file(file_path, board)` - 将 Board 写回 TODO.md，保留 front matter 和栏目顺序

### ui.py

- **`parse_inline_styles(text)`** - 解析 Markdown 行内样式为 Rich markup。支持嵌套样式、方括号转义
- **`THEMES`** - 可用主题列表（12 个）：`textual-dark`, `nord`, `gruvbox`, `catppuccin-mocha`, `dracula`, `monokai`, `flexoki`, `catppuccin-macchiato`, `solarized-dark`, `rose-pine`, `rose-pine-moon`, `atom-one-dark`

- **`TaskCard(Static)`**: 单个任务卡片组件
  - **重要: 使用 `task_obj` 属性，不要用 `task`**（避免与 Textual 的 `Static.task` 属性冲突）
  - `render_task()` - 渲染任务标题 + 元数据（项目名、优先级、标签、时间戳）
  - 优先级颜色映射: P0=red, P1=bright_red, P2=yellow, P3=green, P4=dim

- **`ColumnHeader(Static)`**: 栏目标题组件
  - 显示栏目名称，居中加粗，背景色为 `$primary-darken-3`

- **`KanbanColumn(Vertical)`**: 列容器
  - `add_task(task)` - 向列中添加任务卡片
  - `clear_tasks()` - 清空列中所有任务
  - `get_task_count()` - 获取列中任务数

- **`KanbanBoard(Vertical)`**: 主看板组件
  - 管理栏目布局、任务卡片、选中状态
  - `refresh_board()` - 刷新整个看板（检查栏目是否变化，如变化则重建）
  - `_rebuild_columns()` - 栏目变化时重建 UI 组件
  - `get_all_task_cards()` - 获取所有 TaskCard（跨栏目）
  - `update_selection()` - 更新选中视觉状态
  - `move_task(direction)` - 移动任务（左右换栏 / 上下排序）
  - `navigate_tasks(direction)` - 导航（上下选择 / 左右跳栏）

- **`TitleBar(Static)`**: 应用标题栏

- **`AddTaskScreen(ModalScreen)`**: 添加/编辑任务的模态对话框
  - 支持创建模式和编辑模式
  - Enter 保存，Esc 取消

- **`TuidoApp(App)`**: 主应用
  - `global_mode`: 是否为全局视图模式（禁止上下排序、禁止刷新）
  - 绑定 Vim 风格导航和任务操作
  - `action_change_theme()` - 切换主题（全局模式保存到 config.yaml，本地模式保存到 TODO.md）

### feishu.py

- **`FeishuTable`**: 飞书多维表格 API 封装
  - `get_access_token()` - 获取 tenant_access_token，自动管理 token 过期（提前 5 分钟刷新）
  - `_make_request(method, endpoint)` - 通用 HTTP 请求方法
  - `batch_create(records)` - 批量创建记录
  - `update(table_app_token, table_id, record_id, fields)` - 更新单条记录
  - `batch_delete(record_ids)` - 批量删除记录
  - `fetch_records(table_view_id, field_names, ...)` - 分页获取记录
  - `fetch_all(table_view_id, field_names, ...)` - 获取全部记录（自动分页）
  - `_parse_record(record)` - 解析单条记录，动态提取字段

- **`fetch_tasks()`**: 便捷函数，从飞书获取所有任务记录
  - 返回标准化的记录列表，字段包括: `Task`, `Project`, `Status`, `Tags`, `Priority`, `Timestamp`, `record_id`

### config.py

- `load_global_config()` - 从 `~/.config/tuido/config.yaml` 加载全局配置
- `save_global_theme(theme)` - 保存主题到全局配置

### util.py

- `find_todo_file(path)` - 查找 TODO.md 文件（支持多种大小写命名）
- `parse_timestamp_to_ms(timestamp_str)` - 解析时间戳为毫秒数
- `parse_feishu_timestamp(timestamp_value)` - 将飞书时间戳（毫秒/ISO）转换为 `YYYY-MM-DDTHH:MM` 格式

## 命令

### tui - 看板界面
```bash
tuido tui                       # 打开本地看板
tuido tui --path /project       # 指定路径
tuido tui --remote              # 打开全局视图（从飞书读取）
```

### add - 添加任务
```bash
tuido add "Fix bug #bug !P0"           # 添加到当前目录
tuido add "New feature #feature" --path /project
```

### list - 列出任务
```bash
tuido list                           # 列出本地所有任务
tuido list --status Active           # 按栏目过滤
tuido list --tag bug                 # 按标签过滤
tuido list --priority P0             # 按优先级过滤
tuido list --remote                  # 列出飞书上的任务
```

### pick - 选取任务
```bash
tuido pick                           # 从首列取第一个任务移到下一列
```

### push - 推送到飞书
```bash
tuido push                           # 推送本地任务到飞书
tuido push --remote                  # 推送全局视图到飞书
```

### pull - 从飞书拉取
```bash
tuido pull                           # 从飞书拉取任务到本地
```

### create - 创建示例
```bash
tuido create                         # 创建示例 TODO.md
```

## 键盘快捷键

### 导航 (Vim 风格)
| 按键 | 功能 |
|------|------|
| `h` / `←` | 跳到上一栏 |
| `j` / `↓` | 选中下一任务 |
| `k` / `↑` | 选中上一任务 |
| `l` / `→` | 跳到下一栏 |
| `q` / `Ctrl+C` | 退出 |

### 任务操作
| 按键 | 功能 |
|------|------|
| `Shift+←` / `Shift+H` | 左移任务（移到上一栏目末尾） |
| `Shift+→` / `Shift+L` | 右移任务（移到下一栏目开头） |
| `Shift+↑` / `Shift+K` | 上移任务（同列内调整顺序） |
| `Shift+↓` / `Shift+J` | 下移任务（同列内调整顺序） |

### 任务编辑
| 按键 | 功能 |
|------|------|
| `a` | 在当前栏添加新任务 |
| `d` | 删除选中任务 |
| `e` | 编辑选中任务 |

### 其他操作
| 按键 | 功能 |
|------|------|
| `r` | 从文件刷新（仅本地模式） |
| `s` | 保存到文件 |
| `t` | 切换主题 |
| `?` | 显示帮助 |

## 关键约定

### 1. TaskCard 属性命名
**必须使用 `task_obj`，不能用 `task`**

```python
# 错误 - 与 Textual Static.task 属性冲突
@dataclass
class TaskCard(Static):
    task: Task

# 正确 - 避免命名冲突
@dataclass
class TaskCard(Static):
    task_obj: Task
```

### 2. 异步 DOM 操作
调用 `refresh_board()` 后，使用 `call_after_refresh()` 更新选中状态：

```python
def move_task(self, direction: str) -> None:
    self.refresh_board()
    
    def update_selection_after_refresh():
        self.update_selection()
    
    self.call_after_refresh(update_selection_after_refresh)
```

### 3. 当前状态验证
使用索引前，先验证状态是否存在于可见列：

```python
# 错误 - 可能引发 ValueError
if current_status:
    current_idx = columns.index(current_status)

# 正确 - 安全
if current_status in columns:
    current_idx = columns.index(current_status)
```

### 4. CLI 错误处理与退出码
命令函数统一返回 exit code，由 `main()` 集中处理退出：

```python
# main.py - 命令函数返回 int
@cli.command(name="pick")
@path_option
def pick_command(path: Path) -> int:
    todo_file = util.find_todo_file(path.resolve())
    return run_pick_command(todo_file)

# cmd_*.py - 子命令实现返回 int
def run_pick_command(todo_file: Path) -> int:
    if not todo_file.exists():
        click.echo(f"Error: TODO.md not found", err=True)
        return 1
    return 0

# main() 入口集中处理
import sys

def main():
    logger.remove()
    logger.add(lambda msg: print(msg, end=""), level="WARNING")
    sys.exit(cli())
```

**约定：**
- **禁止在命令函数中直接使用 `raise SystemExit(exit_code)`**
- 命令函数签名添加 `-> int` 返回类型注解
- 成功返回 `0`，错误返回非零值（通常是 `1`）
- 错误信息使用 `click.echo(..., err=True)` 输出到 stderr
- 所有 `run_*_command` 辅助函数统一返回 `int` exit code

### 5. 全局视图模式限制
全局视图（`--remote`）中：
- **禁止**上下移动任务（排序）—— 只允许左右换栏
- **禁止**从文件刷新（`r` 键）
- 主题切换会保存到 `~/.config/tuido/config.yaml`

## 飞书同步

### 配置

**本地 TODO.md front matter**（项目级配置）：
```yaml
---
remote:
  feishu_api_endpoint: https://open.feishu.cn/open-apis
  feishu_table_app_token: your_app_token
  feishu_table_id: your_table_id
  feishu_table_view_id: your_view_id
---
```

**全局配置** `~/.config/tuido/config.yaml`：
```yaml
theme: atom-one-dark
remote:
  feishu_api_endpoint: https://open.feishu.cn/open-apis
  feishu_table_app_token: your_table_app_token
  feishu_table_id: your_table_id
  feishu_table_view_id: your_table_view_id
  feishu_bot_app_id: your_bot_app_id
  feishu_bot_app_secret: your_bot_app_secret
```

**配置字段说明：**
- `feishu_api_endpoint` - 飞书 Open API 端点
- `feishu_table_app_token` - 多维表格 App Token
- `feishu_table_id` - 表格 ID
- `feishu_table_view_id` - 视图 ID（决定同步哪张视图的数据）
- `feishu_bot_app_id` / `feishu_bot_app_secret` - 飞书机器人凭据（用于获取 token）

### 推送到飞书 (push)

```bash
tuido push                          # 推送当前项目任务
tuido push --path /my/project       # 指定项目
tuido push --remote                 # 推送全局视图所有项目
```

**流程：**
1. 从 TODO.md 解析本地任务
2. 从飞书拉取远程记录进行对比
3. 显示差异预览（新增/变更/删除）
4. 用户确认后执行：新增用 `batch_create`、变更用 `update`、删除用 `batch_delete`

**对比键：**
- 本地推送：以 `title` 为唯一键
- 全局推送：以 `(title, project)` 为复合键

### 从飞书拉取 (pull)

```bash
tuido pull                          # 拉取到当前项目
tuido pull --path /my/project       # 指定项目
```

**流程：**
1. 从飞书获取远程记录
2. 与本地任务对比（以 title 为唯一键）
3. 显示差异预览
4. 用户确认后合并到本地 Board
5. 保存回 TODO.md

### 飞书表格字段映射

| 飞书字段 | 本地字段 | 说明 |
|----------|---------|------|
| `Task` | `title` | 任务标题 |
| `Project` | `project` | 项目名称 |
| `Status` | `column` | 任务状态（栏目名） |
| `Tags` | `tags` | 标签（逗号分隔） |
| `Priority` | `priority` | 优先级 |
| `Timestamp` | `updated_at` | 时间戳 |

## 全局视图

使用 `tui --remote` 查看所有项目的任务：

```bash
tuido tui --remote
```

**实现方式：**
1. 从 `~/.config/tuido/config.yaml` 读取全局配置
2. 调用 `fetch_tasks()` 从飞书获取所有任务
3. 通过 `Board.from_feishu_records()` 转换为 Board
4. 保存到临时文件 `/tmp/TODO_global.md`
5. 使用与本地模式相同的 `TuidoApp` 展示

**栏目顺序：** Backlog → Todo → Active → Done → 其他自定义栏目

**特性：**
- 任务标题显示格式：`[项目名] 任务名`（使用 `「项目名」` 格式）
- 主题切换会保存到全局配置
- 禁止上下排序和刷新操作

## 常见任务

### 添加新快捷键
1. 在 `TuidoApp.BINDINGS` 添加绑定
2. 实现对应的 `action_*` 方法
3. 如需更新 UI，调用 `refresh_board()`
4. 刷新后需更新选中状态时，使用 `call_after_refresh()`

### 修改任务显示
1. 更新 `TaskCard.render_task()` 方法
2. 如需新增行内样式，更新 `parse_inline_styles()` 函数
3. 用特殊字符测试标题/标签/负责人
4. 确保 Rich 标记已正确转义

### 添加新栏目
直接在 TODO.md 中添加新的二级标题即可，无需修改代码：

```markdown
## 新栏目
- 新任务
```

刷新看板后，新栏目会自动显示。

### 修改主题列表
编辑 `ui.py` 中的 `THEMES` 列表：

```python
THEMES = [
    "textual-dark",
    "nord",
    # ... 添加新主题
]
```

## 测试

手动运行：
```bash
pip install -e .
tuido tui                       # 打开看板
tuido tui --remote              # 打开全局视图
tuido create                    # 创建示例文件
tuido add "Fix bug #bug !P0"    # 添加任务
tuido pick                      # 选取首任务并移到下一栏
tuido list                      # 列出所有任务
tuido list --remote             # 列出飞书上的任务
tuido push                      # 推送到飞书
tuido pull                      # 从飞书拉取
```

运行测试：
```bash
pytest tests/                   # 运行单元测试
```

## 依赖

| 包 | 版本 | 用途 |
|----|------|------|
| `textual` | >=0.52.0 | TUI 框架 |
| `rich` | >=13.0.0 | 终端格式化（textual 自带） |
| `pydantic` | >=2.0.0 | 数据模型验证 |
| `requests` | >=2.32.5,<3.0.0 | HTTP 请求（飞书 API） |
| `pyyaml` | >=6.0.0 | YAML 解析（front matter、config） |
| `click` | >=8.0.0 | CLI 框架 |
| `loguru` | >=0.7.3,<0.8.0 | 日志记录 |

开发安装：
```bash
pip install -e .
```