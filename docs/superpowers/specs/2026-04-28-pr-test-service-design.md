# PR Test Service 协调启动设计

## 背景

datastat-website 依赖 APIMagic 提供的接口进行功能测试。当前两个仓库各自独立启动测试服务，存在以下问题：

1. 端口冲突：两个仓库的 PR 使用相同公式计算端口
2. 无法关联：修改 datastat-website 时无法自动连接到对应的 APIMagic 测试服务
3. 手动配置：需要手动配置 APIMagic 服务地址

## 目标

实现两种场景的自动化测试服务启动：

- **场景1**：仅修改 datastat-website，自动连接 APIMagic main 分支服务
- **场景2**：同时修改两个仓库（同名分支），自动检测并启动对应的 APIMagic 服务

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                     datastat-website PR                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ PR 描述 (可选):                                              ││
│  │ Depends-On: APIMagic#123  (覆盖自动检测)                      ││
│  └─────────────────────────────────────────────────────────────┘│
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                pr-test-service.yml (datastat-website)            │
│                                                                   │
│  1. 解析 Depends-On 标记 (如有则覆盖后续逻辑)                       │
│  2. 获取当前 PR 分支名                                             │
│  3. 通过 GitHub API 检查 APIMagic 是否有同名分支                    │
│     - 有同名分支 → 检测/启动对应 PR 服务                            │
│     - 无同名分支 → 使用 main 服务 (端口 9999)                       │
│  4. 启动 datastat-website，配置 VITE_MAGICAPI_TARGET              │
└───────────────────────────────────────────────────────────────────┘
```

## 端口分配规则

| 服务类型 | 端口计算 | 示例 |
|---------|---------|------|
| APIMagic main | 9999 (固定) | 9999 |
| APIMagic PR #N | `10000 + N` | PR #123 → 10123 |
| datastat-website PR #M | `20000 + M` | PR #45 → 20045 |

**范围规划:**
- APIMagic PR 服务: 10001 - 19999 (支持约 10000 个 PR)
- datastat-website PR 服务: 20001 - 29999 (支持约 10000 个 PR)

**端口范围分离优势:**
- 避免端口冲突
- 通过端口号可快速识别服务类型

## 场景详细设计

### 场景1: 仅修改 datastat-website

**触发方式:**
- PR 创建/更新自动触发
- 手动触发 workflow_dispatch

**处理流程:**
```
1. 获取 PR 分支名 (如: feat-issue-123)
        ↓
2. 检查 APIMagic 是否有同名分支
        ↓
┌──────────────────┬──────────────────┐
│ 有同名分支        │ 无同名分支        │
├──────────────────┼──────────────────┤
│ 检测服务是否运行   │ 使用 main 服务    │
│ ├─ 已运行 → 使用  │ (端口 9999)      │
│ └─ 未运行 → 启动  │                  │
└──────────────────┴──────────────────┘
        ↓
3. 启动 datastat-website
   - PORT = 20000 + PR_NUMBER
   - VITE_MAGICAPI_TARGET = http://localhost:<apimagic_port>/
```

### 场景2: 同时修改两个仓库（同名分支）

**前提:** 两个仓库的 PR 来自同一个 issue，使用相同分支名

**处理流程:**
```
1. 获取 PR 分支名 (如: feat-issue-123)
        ↓
2. 检查 APIMagic 是否有同名分支 → 是
        ↓
3. 检测 APIMagic 服务是否已运行
   - 检查 /tmp/pr-services/apimagic-branch-feat-issue-123.info
   - 或通过端口探测
        ↓
┌──────────────────┬──────────────────┐
│ 服务已运行        │ 服务未运行        │
├──────────────────┼──────────────────┤
│ 获取端口信息      │ 触发 APIMagic     │
│                  │ workflow 并等待   │
└──────────────────┴──────────────────┘
        ↓
4. 启动 datastat-website
   - PORT = 20000 + PR_NUMBER
   - VITE_MAGICAPI_TARGET = http://localhost:<apimagic_port>/
```

### 手动覆盖

通过 PR 描述或 workflow_dispatch 参数指定 APIMagic 引用：

**PR 描述标记:**
```markdown
Depends-On: APIMagic#123
```

**workflow_dispatch 参数:**
```yaml
inputs:
  pr_number:
    description: 'PR number'
    required: true
  apimagic_ref:
    description: 'APIMagic reference (main, #123, or branch-name)'
    required: false
    default: ''
```

**优先级:** Depends-On 标记 > 同名分支自动匹配 > main 服务

## 服务状态存储

### 状态文件格式

**路径:** `/tmp/pr-services/<service-type>-<identifier>.info`

**文件命名规则:**
| 服务类型 | 文件名格式 | 示例 |
|---------|-----------|------|
| APIMagic PR | `apimagic-pr-<number>.info` | `apimagic-pr-123.info` |
| APIMagic Branch | `apimagic-branch-<name>.info` | `apimagic-branch-feat-api.info` |
| datastat PR | `datastat-pr-<number>.info` | `datastat-pr-45.info` |

**文件内容:**
```
port=10123
repo=APIMagic
pr_number=123
branch=feat-issue-123
started_at=2024-01-15T10:30:00Z
```

### 用途

1. 检测服务是否运行
2. 获取服务端口
3. 支持跨仓库查找（通过 branch 名称）

## 实现细节

### APIMagic workflow 变更

**文件:** `.github/workflows/pr-test-service.yml`

**变更点:**
1. 端口计算: `10000 + PR_NUMBER`
2. 容器名: `apimagic-test-$PR_NUMBER`
3. 状态文件路径更新
4. 新增分支信息到状态文件

### datastat-website workflow 变更

**文件:** `.github/workflows/pr-test-service.yml`

**变更点:**
1. 端口计算: `20000 + PR_NUMBER`
2. 容器名: `datastat-test-$PR_NUMBER`
3. 新增步骤：解析 Depends-On 标记
4. 新增步骤：检测 APIMagic 同名分支
5. 新增步骤：触发/等待 APIMagic 服务
6. 新增步骤：配置 VITE_MAGICAPI_TARGET
7. 新增 workflow_dispatch 参数: `apimagic_ref`

### 分支检测实现

```bash
check_apimagic_branch() {
  local branch_name=$1
  
  # 通过 GitHub API 检查分支是否存在
  if gh api repos/opensourceways-test/APIMagic/branches/$branch_name > /dev/null 2>&1; then
    echo "exists"
    return 0
  else
    echo "not_found"
    return 1
  fi
}
```

### 服务就绪检测

```bash
wait_for_apimagic_service() {
  local port=$1
  local max_wait=300  # 5分钟
  local interval=10
  
  for i in $(seq 1 $((max_wait/interval))); do
    if curl -s --max-time 2 http://localhost:$port/ > /dev/null 2>&1; then
      echo "APIMagic ready on port $port"
      return 0
    fi
    echo "Waiting for APIMagic... ($((i*interval))s)"
    sleep $interval
  done
  
  echo "Timeout waiting for APIMagic on port $port"
  return 1
}
```

### 触发 APIMagic workflow

```bash
trigger_apimagic_workflow() {
  local branch_name=$1
  
  # 获取分支对应的 PR 号
  local pr_number=$(gh api repos/opensourceways-test/APIMagic/pulls \
    --jq ".[] | select(.head.ref == \"$branch_name\") | .number")
  
  if [ -z "$pr_number" ]; then
    echo "No PR found for branch $branch_name"
    return 1
  fi
  
  # 触发 workflow
  gh workflow run pr-test-service.yml \
    --repo opensourceways-test/APIMagic \
    -f pr_number=$pr_number
  
  echo "Triggered APIMagic workflow for PR #$pr_number"
  
  # 返回期望的端口
  echo $((10000 + pr_number))
}
```

## 文件变更清单

### 需要修改的文件

1. `opensourceways-test/APIMagic/.github/workflows/pr-test-service.yml`
   - 更新端口计算公式
   - 更新状态文件路径和内容

2. `opensourceways-test/APIMagic/.github/workflows/stop-pr-test-service.yml`
   - 更新端口和容器名引用

3. `opensourceways-test/datastat-website/.github/workflows/pr-test-service.yml`
   - 更新端口计算公式
   - 新增分支检测和服务协调逻辑
   - 新增 workflow_dispatch 参数

4. `opensourceways-test/datastat-website/.github/workflows/stop-pr-test-service.yml`
   - 更新端口和容器名引用

## 测试计划

### 场景1测试

| 测试项 | 验证点 |
|-------|-------|
| datastat PR 无同名分支 | 连接 APIMagic main (9999) |
| datastat PR 有同名分支 | 自动连接同名分支的 APIMagic 服务 |
| 手动指定 apimagic_ref | 使用指定的 APIMagic 引用 |

### 场景2测试

| 测试项 | 验证点 |
|-------|-------|
| 两个 PR 同时存在 | datastat 自动检测 APIMagic 服务 |
| APIMagic 未启动 | 自动触发 APIMagic workflow |
| Depends-On 标记 | 覆盖自动检测逻辑 |

### 端口冲突测试

| 测试项 | 验证点 |
|-------|-------|
| APIMagic PR #5 | 端口 10005 |
| datastat PR #5 | 端口 20005 |
| 两服务同时运行 | 无端口冲突 |

## 回滚方案

如需回滚，恢复以下文件的原始版本：
- `APIMagic/.github/workflows/pr-test-service.yml`
- `APIMagic/.github/workflows/stop-pr-test-service.yml`
- `datastat-website/.github/workflows/pr-test-service.yml`
- `datastat-website/.github/workflows/stop-pr-test-service.yml`

原始版本已备份在 `APIMagic/.github/old/` 目录。