# Issue Workflow Refactoring Design

## Overview

将现有的 `issue-workflow.yml` 拆分为两个独立的工作流文件，并使用 Composite Actions 消除重复逻辑。

## Goals

- 清晰分离分析和开发两个阶段
- 消除冗余的 setup、checkout、issue 获取逻辑
- 使用 GitHub Artifacts 传递设计文档
- 保持易于维护和调试

## File Structure

```
.github/
├── workflows/
│   ├── issue-analyzer.yml      # 分析阶段（accept 标签触发）
│   └── issue-developer.yml     # 开发阶段（approved 标签触发）
└── actions/
    ├── setup/
    │   └── action.yml          # opencode 环境初始化
    ├── fetch-issue/
    │   └── action.yml          # 获取 issue 内容
    └── checkout-repos/
        └── action.yml          # 根据 dev_type checkout 目标仓库
```

## Workflow: issue-analyzer.yml

### Trigger
```yaml
on:
  issues:
    types: [labeled]
```

### Job: analyze
**Condition:** `github.event.label.name == 'accept' && github.event.issue.state == 'open'`

**Steps:**
1. Checkout changelog
2. Setup (composite action)
3. Fetch issue content (composite action)
4. Determine dev_type
   - Run opencode to classify: api-only / frontend-only / both
   - Parse output, validate, default to "both"
5. Checkout target repos (composite action with analysis mode)
   - sparse-checkout: APIMagic (api/), datastat-website (views/, components/)
6. Generate design documents
   - Run opencode with repo structure
   - Output: requirements.md, design.md
7. Upload artifacts
   - requirements.md
   - design.md
   - dev_type.txt
8. Add dev_type label
9. Comment design documents

### Outputs
- `dev_type`: api-only / frontend-only / both

## Workflow: issue-developer.yml

### Trigger
```yaml
on:
  issues:
    types: [labeled]
```

### Job: develop
**Condition:** `github.event.label.name == 'approved' && github.event.issue.state == 'open'`

**Steps:**
1. Checkout changelog
2. Setup (composite action)
3. Fetch issue content (composite action)
4. Download artifacts
   - requirements.md
   - design.md
   - dev_type.txt
5. Checkout target repos (composite action with full checkout)
   - fetch-depth: 0 for commits
6. Develop API (if api-only or both)
   - Create branch `issue-{number}-api`
   - Run opencode with design docs
   - Commit and push if changes
7. Develop Frontend (if frontend-only or both)
   - Create branch `issue-{number}-frontend`
   - Run opencode with design docs
   - Commit and push if changes
8. Create PRs

## Composite Actions

### setup/action.yml
```yaml
inputs:
  opencode-api-key:
    required: true
runs:
  using: composite
  steps:
    - name: Setup opencode environment
      run: |
        mkdir -p $HOME/.local/share/opencode/log $HOME/.config/opencode /tmp/opencode
        printf '{"alibaba-cn":{"type":"api","key":"%s"}}\n' "${{ inputs.opencode-api-key }}" > $HOME/.local/share/opencode/auth.json
        printf '{"$schema":"https://opencode.ai/config.json"}\n' > $HOME/.config/opencode/opencode.json
      shell: bash
```

### fetch-issue/action.yml
```yaml
inputs:
  issue-number:
    required: true
  github-token:
    required: true
outputs:
  title:
    value: ${{ steps.fetch.outputs.title }}
  body:
    value: ${{ steps.fetch.outputs.body }}
runs:
  using: composite
  steps:
    - name: Fetch issue content
      id: fetch
      env:
        GH_TOKEN: ${{ inputs.github-token }}
      run: |
        TITLE=$(gh issue view ${{ inputs.issue-number }} --json title -q '.title')
        BODY=$(gh issue view ${{ inputs.issue-number }} --json body -q '.body')
        printf "Issue #%s: %s\n\n%s" "${{ inputs.issue-number }}" "$TITLE" "$BODY" > /tmp/opencode/issue.txt
        echo "title<<EOF" >> $GITHUB_OUTPUT
        echo "$TITLE" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT
        echo "body<<EOF" >> $GITHUB_OUTPUT
        echo "$BODY" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT
      shell: bash
```

### checkout-repos/action.yml
```yaml
inputs:
  dev-type:
    required: true
    description: "api-only, frontend-only, or both"
  mode:
    required: false
    default: "full"
    description: "analysis (sparse) or full"
  cross-repo-token:
    required: true
  repository-owner:
    required: true
runs:
  using: composite
  steps:
    - name: Checkout APIMagic
      if: inputs.dev-type == 'api-only' || inputs.dev-type == 'both'
      uses: actions/checkout@v4
      with:
        repository: ${{ inputs.repository-owner }}/APIMagic
        token: ${{ inputs.cross-repo-token }}
        path: APIMagic
        sparse-checkout: ${{ inputs.mode == 'analysis' && 'magic-api/api' || '' }}
        fetch-depth: ${{ inputs.mode == 'analysis' && 1 || 0 }}
    
    - name: Checkout datastat-website
      if: inputs.dev-type == 'frontend-only' || inputs.dev-type == 'both'
      uses: actions/checkout@v4
      with:
        repository: ${{ inputs.repository-owner }}/datastat-website
        token: ${{ inputs.cross-repo-token }}
        path: datastat-website
        sparse-checkout: ${{ inputs.mode == 'analysis' && 'src/views,src/components' || '' }}
        fetch-depth: ${{ inputs.mode == 'analysis' && 1 || 0 }}
```

## Artifacts

### Analyzer uploads:
- `requirements.md` - 需求分析文档
- `design.md` - 技术设计方案
- `dev_type.txt` - 开发类型

### Developer downloads:
- Same artifacts from analyzer run
- Match by issue number via workflow_run or manual storage

Note: Since workflows are triggered independently by labels, artifacts will be stored with issue number in the name for easy retrieval.

## Migration Path

1. Create `.github/actions/` directory and composite actions
2. Create `issue-analyzer.yml`
3. Create `issue-developer.yml`
4. Test with a new issue
5. Remove old `issue-workflow.yml`

## Redundancy Eliminated

| Current (issue-workflow.yml) | Refactored |
|------------------------------|------------|
| Setup logic duplicated in both jobs | One setup action |
| Issue fetch logic duplicated | One fetch-issue action |
| Repo checkout logic duplicated | One checkout-repos action with mode parameter |
| Design docs passed via issue comments | Passed via artifacts (cleaner) |

## Testing Strategy

1. Create test issue with `accept` label → verify analyzer runs
2. Verify artifacts are uploaded correctly
3. Add `approved` label → verify developer runs
4. Verify correct repos are checked out based on dev_type
5. Verify PRs are created in target repositories