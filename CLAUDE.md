# CLAUDE.md

## 架构设计概览

这是一个基于 Go 的 GitHub CodeAgent 系统，采用处理器模式（Processor Pattern）处理不同类型的 GitHub 事件。

### 用户交互方式

用户在 Issue/PR 评论中与 AI 代理交互，有两种**独立**的触发方式：

1. **@mention + 自然语言**：`@fennoai 帮我分析下这个 bug 的根因` — 由 TagProcessor 处理，用户用自然语言描述需求
2. **斜杠命令**：`/code`、`/review`、`/plan` — 由 SlashProcessor 处理，用户通过预定义命令触发特定功能

**注意**：这两种方式是独立的，不是 `@fennoai /code` 这样组合使用。@mention 后面跟的是自然语言，斜杠命令是单独的评论。

### 处理器架构 (Processor Architecture)

系统核心采用 **Dispatcher + Processor** 模式：

#### 1. 处理器类型 (Processor Types)

- **TagProcessor** - 处理 @ 提及事件 (如 @codeagent, @example-org)
- **SlashProcessor** - 处理斜杠命令 (/code, /continue, /review, /plan, /simplify, /close)
- **AgentProcessor** - 处理自动化事件 (Issue 分配、标签添加、自动审查)
- **CustomCommandProcessor** - 处理自定义命令和子代理

#### 2. 执行模式 (Execution Modes)

```go
type ExecutionMode string

const (
    TagMode           ExecutionMode = "tag"            // @提及模式
    SlashMode         ExecutionMode = "slash"          // 斜杠命令模式
    AgentMode         ExecutionMode = "agent"          // 自动化模式
)
```

#### 3. 处理器接口 (Processor Interface)

所有处理器都实现统一接口：

```go
type Processor interface {
    CanHandle(ctx context.Context, event models.GitHubContext) bool
    Execute(ctx context.Context, event models.GitHubContext) error
    GetPriority() int
    GetMode() ExecutionMode
    GetDescription() string
    GetHandlerName() string
}
```

#### 4. 分发器 (Dispatcher)

- 管理所有注册的处理器
- 按优先级排序处理器
- 根据事件类型选择合适的处理器
- 支持模式启用/禁用控制

### 核心组件

#### 1. 上下文管理 (Context Management)

- **ContextCollector** - 收集 GitHub 上下文信息
- **ContextFormatter** - 格式化上下文内容
- **ContextBuilder** - 构建特定类型的上下文

#### 2. 工作空间管理 (Workspace Management)

- **Manager** - 工作空间生命周期管理
- **ContainerService** - Docker 容器服务
- **GitService** - Git 操作服务
- **RepoCacheService** - 仓库缓存服务

#### 3. MCP 集成 (MCP Integration)

- **MCPClient** - MCP 协议客户端
- **Manager** - MCP 服务管理
- **Servers** - GitHub 集成服务器

#### 4. GitHub 集成 (GitHub Integration)

- **ClientManager** - GitHub 客户端管理
- **GraphQLClient** - GraphQL API 客户端
- **Auth** - 认证管理 (App/PAT)

### 安全代理网关 (Security Proxy Gateway)

**核心原则：凭证永不进入容器，所有认证请求通过宿主机代理转发。**

详细设计见 `docs/architecture/codeagent/security-proxy-gateway-design.md`

#### 架构概要

```
宿主机 Proxy (:8887)              Docker 容器
┌─────────────────────┐          ┌─────────────────────────┐
│  AI Handler (/ai)   │◀─────────│ ANTHROPIC_BASE_URL      │
│  Git Credential     │◀─────────│ codeagent-credential    │
│  (/git/credential)  │          │ (git credential helper) │
│                     │          │                         │
│  Token Store        │          │ 环境变量:               │
│  (真实 token)       │          │ - CODEAGENT_PROXY       │
└─────────────────────┘          │ - CODEAGENT_SESSION_TOKEN│
                                 │ - CODEAGENT_TRACE_ID    │
                                 │ (无真实 token)          │
                                 └─────────────────────────┘
```

#### Trace ID 传递机制

确保事件到 Proxy 请求的全链路追踪：

1. **事件入口** → `server.HandleWebhook()` 从 `X-Trace-ID` 或 `X-GitHub-Delivery` 获取 trace ID
2. **Prompt 调用** → `docker exec -e CODEAGENT_TRACE_ID=xxx` 传递到容器
3. **容器内请求** → MCP server 和 `codeagent-credential` 读取环境变量，在请求 Proxy 时带上 `X-Trace-ID` header
4. **Proxy 日志** → 从 header 获取 trace ID，关联到 reqid context

#### 关键文件

| 文件                                        | 职责                                                                   |
| ------------------------------------------- | ---------------------------------------------------------------------- |
| `internal/proxy/server.go`                  | Proxy 服务入口                                                         |
| `internal/proxy/handlers/ai.go`             | AI API 转发，注入真实 token                                            |
| `internal/proxy/handlers/git_credential.go` | Git 认证，返回 platform token                                          |
| `internal/code/*_docker.go`                 | 容器启动，设置 `CODEAGENT_TRACE_ID` 环境变量                           |
| `cmd/codeagent-*-mcp-server/main.go`        | MCP server，请求时带 `X-Trace-ID`                                      |
| `cmd/codeagent-credential/main.go`          | Git credential helper 和 CLI wrapper（Go 实现），请求时带 `X-Trace-ID` |
| `internal/credential/helper.go`             | Git credential helper 核心逻辑                                         |
| `internal/credential/wrapper.go`            | CLI wrapper 核心逻辑（gh/glab/tea）                                    |

#### CLI Wrapper 兼容性注意事项

`internal/credential/wrapper.go` 中的 `extractOrgFromCLIArgs` 函数依赖 gh/glab CLI 的参数格式来提取目标 org，用于从 proxy 获取正确的 token。

**当前支持的参数格式：**

| CLI     | 参数格式                               | 示例                                             |
| ------- | -------------------------------------- | ------------------------------------------------ |
| gh/glab | `--repo OWNER/REPO` 或 `-R OWNER/REPO` | `--repo CarlJi/codeagent`                        |
| gh      | `--repo HOST/OWNER/REPO`               | `--repo github.com/CarlJi/codeagent`             |
| glab    | `--repo GROUP/NAMESPACE/REPO`          | `--repo my-group/subgroup/repo`                  |
| gh/glab | `--repo <HTTPS URL>`                   | `--repo https://github.com/owner/repo`           |
| gh/glab | `--repo <SSH URL>`                     | `--repo git@gitlab.com:group/repo.git`           |
| gh      | `api repos/OWNER/REPO/...`             | `api repos/CarlJi/codeagent/pulls`               |
| glab    | `api projects/OWNER%2FREPO/...`        | `api projects/CarlJi%2Fcodeagent/merge_requests` |

**升级 gh/glab CLI 时的检查清单：**

- [ ] 确认 `--repo` / `-R` 参数格式是否有变化
- [ ] 确认 `api` 命令的路径格式是否有变化（gh 用 `repos/`，glab 用 `projects/`）
- [ ] 运行 `go test ./internal/credential/... -run "TestExtract"` 确保解析逻辑仍然正确
- [ ] 如有格式变化，更新 `extractOrgFromCLIArgs` 和 `extractOwnerFromRepoArg` 函数
- [ ] 更新对应的测试用例

**相关测试文件：** `internal/credential/wrapper_test.go`

## 开发指南

### 对外文档约定

- 面向用户的对外文档主入口位于 `www` 目录，具体内容文件放在 `www/content/docs/{en,zh}`。
- 需要新增、调整或规划公开文档时，默认先检查 `www` 文档站的信息架构、导航与多语言内容，而不是根目录 `docs/`。
- 根目录 `docs/` 主要承载架构设计、实现说明、内部方案等仓库内文档；若内容是给最终用户阅读的，优先放到 `www` 文档站，再按需从其他位置链接过去。

### 添加新 AI Model Provider

当需要添加新的 AI Model Provider (如 Zhipu GLM、DeepSeek 等) 时，需要在以下位置进行修改：

#### 1. 配置定义 (internal/config/)

**provider.go** - 添加 provider 常量和配置路由：

```go
// 添加 provider 常量
const (
    ProviderZhipu = "zhipu"  // 新增
)

// 在 GetProviderConfig() 中添加路由
case ProviderZhipu:
    return &c.Zhipu, nil
```

**config.go** - 在 Config 结构体中添加字段：

```go
type Config struct {
    // ...
    Zhipu ClaudeCodeOnConfig `yaml:"zhipu,omitempty"`  // 新增
}
```

**validation.go** - 添加验证逻辑：

```go
// ValidateAllProviders() 中添加到 providers 列表
providers := []string{
    ProviderClaude,
    // ...
    ProviderZhipu,  // 新增
}

// isProviderConfigured() 中添加到 switch case
case ProviderClaude, ProviderMiniMax, ProviderDeepSeek, ProviderKimi, ProviderZhipu:  // 新增 ProviderZhipu
```

**repo_config.go** - 添加仓库级配置支持（6 处修改）：

```go
// 1. RepoConfig 结构体
type RepoConfig struct {
    Zhipu *RepoProviderModelConfig `yaml:"zhipu,omitempty"`  // 新增
}

// 2. Validate() - validProviders 列表
validProviders := []string{
    ProviderClaude, ProviderMiniMax, ProviderDeepSeek,
    ProviderKimi, ProviderZhipu, ProviderGemini, ProviderCodex,  // 新增 ProviderZhipu
}

// 3. Validate() - providerConfigs map
providerConfigs := map[string]*RepoProviderModelConfig{
    ProviderZhipu: rc.Zhipu,  // 新增
}

// 4. IsEmpty() 方法
return rc.DefaultAIProvider == "" &&
    rc.Zhipu == nil &&  // 新增
    // ...

// 5. GetConfiguredProviders() 方法
if rc.Zhipu != nil {
    providers = append(providers, ProviderZhipu)  // 新增
}

// 6. MergeWithRepoConfig() 方法
if repoConfig.Zhipu != nil {
    mergeClaudeCodeOnConfig(&c.Zhipu, *repoConfig.Zhipu)  // 新增
}
```

#### 2. 容器服务 (internal/container/)

**service.go** - 添加到 supportedProviders 列表：

```go
supportedProviders := []string{
    config.ProviderClaude,
    // ...
    config.ProviderZhipu,  // 新增
}
```

**service_test.go** - 更新测试用例注释：

```go
expected: 7, // All supported providers (claude, gemini, codex, minimax, deepseek, kimi, zhipu)
```

#### 3. 配置示例文件

**config.example.yaml** - 添加完整配置示例：

```yaml
# Zhipu GLM Provider (智谱 AI, 使用 Claude Code On)
# zhipu:
#   implementation: claude_code_on
#   api_key_env: ZHIPU_API_KEY
#   base_url: https://open.bigmodel.cn/api/paas/v4
#   model: glm-4-plus
#   default_haiku_model: glm-4-flash
#   default_sonnet_model: glm-4-plus
#   default_opus_model: glm-4-plus
#   container_image: codeagent:local
#   timeout: 30m
```

#### 4. 文档更新

**README.md / README.zh.md** - 添加到支持模型表格：

```markdown
| `-zhipu` | Claude Code CLI (Claude Code On) | glm-4-plus | 所有场景 |
```

#### 5. 验证清单

添加新 AI Model Provider 后，请确保：

- [ ] 所有配置文件中都添加了新 provider
- [ ] 验证逻辑完整（包含在 ValidateAllProviders 和 isProviderConfigured 中）
- [ ] 仓库级配置支持完整（6 处修改）
- [ ] 容器服务的 supportedProviders 列表已更新
- [ ] 测试用例已更新
- [ ] 配置示例文件已添加
- [ ] 用户文档已更新
- [ ] 运行 `go test ./internal/config/...` 确保测试通过
- [ ] 运行 `go test ./internal/container/...` 确保测试通过

### 添加新处理器

1. 实现 `Processor` 接口
2. 继承 `BaseProcessor` 获得基础功能
3. 在 `Dispatcher` 中注册处理器
4. 设置合适的优先级和模式

### 添加新斜杠命令

添加新的斜杠命令时，必须评估是否需要权限检查：

#### 需要权限检查的命令

以下类型的命令**必须**添加权限检查（使用 `checkClosePermission` 方法）：

| 命令类型     | 示例        | 原因                       |
| ------------ | ----------- | -------------------------- |
| 修改仓库状态 | `/close`    | 关闭 Issue/PR 影响仓库状态 |
| 删除内容     | `/compact`  | 删除 bot 评论，不可恢复    |
| 代码修改     | `/simplify` | 直接修改 PR 代码           |

#### 权限检查要求

- 用户必须是 Issue/PR 创建者，**或**
- 用户必须拥有仓库 `write`/`admin`/`maintain` 权限

#### 实现模板

```go
func (sp *SlashProcessor) executeXxxCommand(ctx context.Context, event platform.PlatformContext, cmdInfo *models.CommandInfo) error {
    xl := xlog.NewWith(ctx)

    issueCommentCtx, ok := event.AsIssueCommentContext()
    if !ok {
        return fmt.Errorf("/xxx command is only supported in Issue or PR comments")
    }

    repo := issueCommentCtx.GetRepository()
    owner := repo.GetOwner().GetLogin()
    repoName := repo.GetName()
    commentUser := issueCommentCtx.GetComment().GetAuthor().GetLogin()
    isPRComment := issueCommentCtx.IsPRComment()
    number := sp.getIssueOrPRNumber(event, isPRComment)

    instanceKey := event.GetInstanceKey()
    adapter, err := sp.platformManager.GetAdapterByKey(instanceKey)
    if err != nil {
        return fmt.Errorf("failed to get adapter: %w", err)
    }

    // 权限检查
    hasPermission, err := sp.checkClosePermission(ctx, event, adapter, owner, repoName, number, commentUser, isPRComment)
    if err != nil {
        return fmt.Errorf("failed to check permission: %w", err)
    }

    if !hasPermission {
        xl.Errorf("Permission denied: user %s is neither the creator nor has write permission for %s #%d",
            commentUser, ifElse(isPRComment, "PR", "Issue"), number)
        return fmt.Errorf("permission denied: user %s is neither the creator nor has write permission", commentUser)
    }

    // 执行命令逻辑...
}
```

#### 检查清单

添加新斜杠命令时，请确保：

- [ ] 评估命令是否需要权限检查（参考上表）
- [ ] 如需权限检查，使用 `checkClosePermission` 方法
- [ ] 权限不足时返回明确的错误信息
- [ ] 在 `pkg/models/events.go` 中添加命令常量
- [ ] 在 `slash_processor.go` 的 `Execute` 方法中添加 case 分支
- [ ] 添加对应的单元测试

### /plan 命令 (Plan Mode)

`/plan` 命令使用 Claude Code CLI 的 `--permission-mode plan` 参数进入只读模式，用于分析 Issue/PR 并产出实现计划，而不直接执行代码修改。

#### 核心特性

- **只读模式**：Claude 只能使用 Read, Glob, Grep, WebSearch 等只读工具
- **禁止写操作**：不能执行 Edit, Write, Bash 等写操作
- **复用上下文**：与 `/code` 使用相同的 IssueContextBuilder/PRContextBuilder
- **支持 AI 模型选择**：支持 `-claude`、`-gemini` 等模型参数

#### 实现原理

```
用户触发 /plan
    ↓
SlashProcessor.executePlanCommand()
    ↓
设置 cmdInfo.PlanMode = true
    ↓
ActionExecutor.Execute() → 设置 workspace.PlanMode = true
    ↓
Claude Docker/Local 添加 --permission-mode plan 参数
    ↓
Claude 收到完整上下文，但只能使用只读工具
    ↓
自动输出分析和实现计划
```

#### 关键文件

| 文件                                        | 修改内容                                    |
| ------------------------------------------- | ------------------------------------------- |
| `pkg/models/events.go`                      | CommandPlan 常量，CommandInfo.PlanMode 字段 |
| `pkg/models/workspace.go`                   | Workspace.PlanMode 字段                     |
| `internal/code/claude_docker.go`            | 根据 PlanMode 添加 `--permission-mode plan` |
| `internal/code/claude_local.go`             | 同上                                        |
| `internal/processors/slash_processor.go`    | executePlanCommand 方法                     |
| `internal/processors/execution/executor.go` | 传递 PlanMode 到 workspace                  |

#### 使用示例

```
# Issue 中使用
/plan

# PR 中使用并指定模型
/plan -claude analyze the implementation approach

# 后续可通过 /code 执行计划
/code
```

### 日志规范

- 只使用 Error、Info、Debug 三个级别，不使用 Warn
- Error: 操作失败或异常状态（需要关注的真正错误）
- Info: 重要的操作记录，包括预期中的降级/回退、配置变更、生命周期事件
- Debug: 详细的诊断信息

### 测试规范

- 单元测试: `*_test.go`
- 集成测试: `test/integration/`
- 基准测试: `*_benchmark_test.go`

### 项目结构

```
internal/
├── processors/     # 处理器实现
├── context/       # 上下文管理（含 templates/ 三平台模板）
├── workspace/     # 工作空间管理
├── platform/      # 多平台适配（github/gitlab/cnb/gitea）
├── code/          # AI 执行后端（claude/gemini/codex）
├── proxy/         # 安全代理网关
├── mcp/           # MCP 协议支持
├── config/        # 配置管理
├── container/     # Docker 容器生命周期
└── server/        # HTTP 服务器（Webhook 入口）
```

### 构建和测试

```bash
# 构建主服务
go build ./cmd/codeagent

# 构建所有二进制（含 router/portal/mcp-server）
make build-all

# 单元测试
go test ./...

# 集成测试（验证 webhook → prompt 全链路）
go test ./test/integration/...

# 更新 golden 文件（修改模板后执行）
go test ./test/integration/... -update

# 格式化
go fmt ./...
```

### 大文件注意事项

以下文件超过 1200 行，包含多个独立职责，AI 编辑时需分段阅读，避免遗漏上下文：

| 文件 | 行数 | 内部结构 |
| ---- | ---- | -------- |
| `internal/workspace/manager.go` | ~1580 | workspace 创建、git 操作、fork PR 处理、清理 |
| `internal/portal/service.go` | ~1555 | session 管理、API key 配置、用户权限 |
| `internal/processors/execution/executor.go` | ~1320 | 上下文构建、容器启动、流式输出、日志存储 |
| `internal/processors/slash_processor.go` | ~1207 | 每个命令一个 execute 方法，权限检查命令见文件顶部注释 |

### 同步约束（多文件联动）

以下操作**必须**同时修改多个文件，切勿只改其中一处：

| 操作 | 必须同步修改的文件 |
| ---- | ----------------- |
| 修改 prompt 逻辑 | `templates/default.tmpl` + `gitlab.tmpl` + `cnb.tmpl`（三个同步） |
| 添加新 AI Provider | `internal/config/` 4个文件 + `internal/container/service.go` + `config.example.yaml`（见上方详细清单） |
| 添加新斜杠命令 | `pkg/models/events.go`（常量）+ `slash_processor.go`（case 分支）+ 权限评估 |

## 架构原则

- 不要 fallback，而应该从最佳实践角度解决问题
- 少写代码, 遵循:
  - KISS(Keep It Stupid Simple)
  - YAGNI(You Ain't Gonna Need It)
  - DRY(Don't Repeat Yourself)

- 写好代码, 遵循 SOLID 原则
  - Single Responsibility Principle
  - Open/Closed Principle
  - Liskov Substitution Principle
  - Interface Segregation Principle
  - Dependency Inversion Principle
- 写对代码, 使用 ask-question tool

### 多平台支持原则

- **优先使用平台无关接口，避免从 URL 解析信息**
  - 不要从仓库 URL 中解析 owner/repo 等信息，因为不同平台（GitHub、GitLab、Gitea 等）的 URL 格式可能不同，解析容易失败
  - 应使用 `platform.PullRequest`、`platform.Repository` 等接口提供的方法获取信息
  - 例如：获取 fork 仓库的 owner 应使用 `pr.GetHead().GetRepository().GetOwner().GetLogin()`，而非从 URL 路径解析

### Prompt 模板同步修改原则

修改 `internal/context/templates/` 下的模板文件时，**必须同步修改所有三个模板**：

| 模板文件       | 平台         | 说明                     |
| -------------- | ------------ | ------------------------ |
| `default.tmpl` | GitHub/Gitea | 使用 `gh` CLI，PR 术语   |
| `gitlab.tmpl`  | GitLab       | 使用 `glab` CLI，MR 术语 |
| `cnb.tmpl`     | CNB          | 使用 MCP 工具，MR 术语   |

#### 模板一致性要求

1. **行为逻辑必须一致**：除平台特定差异外，三个模板的执行步骤、条件判断、功能要求必须保持一致
2. **使用 Go 模板语法**：所有模板都使用 `{{.Variable}}` 格式，由 `text/template` 引擎渲染
3. **平台特定差异**（允许不同）：
   - CLI 工具名称：`gh` vs `glab` vs MCP 工具
   - 术语：PR vs MR
   - 评论工具：`github-comments` vs `gitlab-comments` vs `cnb-comments`
   - Token 变量：`GH_TOKEN` vs `GITLAB_TOKEN` vs `CNB_TOKEN`
   - commit message tag：`<gh_commit_message>` vs `<glab_commit_message>` vs `<cnb_commit_message>`

#### 修改模板时的检查清单

- [ ] 三个模板的执行步骤结构一致
- [ ] Fork PR 处理逻辑一致（使用 `{{if .IsForkPR}}` 条件判断）
- [ ] git push 格式一致（使用内联条件避免多余空行）
- [ ] Review footer 格式一致
- [ ] 无重复或遗漏的功能说明
- [ ] 平台特定的 tag 名称正确（如 `<cnb_commit_message>` 而非 `<gh_commit_message>`）

### Skill 与 MCP 的职责边界

- **优先在 skill / prompt 层表达 AI 行为约束，不要先收紧 MCP 工具 schema**
  - MCP 工具描述的是平台能力边界，给 AI 调用；是否允许 AI 使用某个能力，优先在 `.codeagent/skills/*.md` 或提示词中约束
  - 只有当平台能力本身就不应该暴露时，才修改 `pkg/mcp/servers/` 的 schema 或参数校验
  - 这次 `submit_review` 的 `REQUEST_CHANGES` 误判，根因是把“AI 不应发阻塞 review”的策略误当成“平台工具不支持该 event”，导致改错到 MCP 层
- **成对维护同一能力的多份 review skill**
  - 修改 `.codeagent/skills/codeagent-review/SKILL.md` 时，必须同时检查 `.codeagent/skills/codeagent-review-low/SKILL.md`
  - 这两个 skill 承担的是同一类 review 行为约束，只是平台和字段命名不同；除平台差异外，规则应保持一致
  - 提交前要显式核对是否两边都同步更新，避免只改一个 skill 导致行为漂移

### 提示词修改与集成测试同步原则

**核心原则：修改提示词时必须同步更新集成测试，确保提示词质量。**

#### 背景说明

项目在 `/test` 目录下维护了集成测试，用于保障提示词的质量和功能的正确性。这些测试验证了 AI 代理在各种场景下的行为是否符合预期。

#### 修改规则

当修改以下内容时，**必须同步更新** `/test` 目录下的相关集成测试：

1. **Prompt 模板文件** (`internal/context/templates/`)
   - `default.tmpl` (GitHub/Gitea)
   - `gitlab.tmpl` (GitLab)
   - `cnb.tmpl` (CNB)

2. **处理器逻辑** (`internal/processors/`)
   - 影响 AI 代理行为的代码修改
   - 新增或修改处理器功能

3. **上下文构建逻辑** (`internal/context/`)
   - 修改传递给 AI 的上下文信息
   - 调整上下文格式化逻辑

#### 集成测试更新要点

- **验证新行为**：为新增的功能或修改的逻辑添加测试用例
- **保持覆盖率**：确保所有提示词变更都有对应的测试覆盖
- **平台一致性**：如果修改涉及多平台，确保各平台的测试都已更新
- **测试通过**：运行 `go test ./test/...` 确保所有集成测试通过

#### 检查清单

修改提示词相关内容时，请确保：

- [ ] 已识别需要修改的提示词模板（所有平台）
- [ ] 已更新相关的集成测试用例
- [ ] 已运行集成测试并确保通过：`go test ./test/...`
- [ ] 已在 PR 中说明测试更新内容
