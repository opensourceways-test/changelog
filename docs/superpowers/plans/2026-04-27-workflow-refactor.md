# Issue Workflow Refactoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor issue-workflow.yml into two independent workflows with composite actions to eliminate redundancy.

**Architecture:** Split single workflow into analyzer and developer workflows triggered by different labels. Extract common setup, issue fetch, and repo checkout logic into reusable composite actions. Use GitHub Artifacts for design document transfer between phases.

**Tech Stack:** GitHub Actions, Composite Actions, GitHub CLI (gh), opencode CLI

---

## File Structure

```
.github/
├── workflows/
│   ├── issue-analyzer.yml      # 分析阶段（accept 标签触发）
│   ├── issue-developer.yml     # 开发阶段（approved 标签触发）
│   └── issue-workflow.yml      # 将在最后删除
└── actions/
    ├── setup/
    │   └── action.yml          # opencode 环境初始化
    ├── fetch-issue/
    │   └── action.yml          # 获取 issue 内容
    └── checkout-repos/
        └── action.yml          # 根据 dev_type checkout 目标仓库
```

---

### Task 1: Create setup composite action

**Files:**
- Create: `.github/actions/setup/action.yml`

- [ ] **Step 1: Create actions directory structure**

```bash
mkdir -p .github/actions/setup .github/actions/fetch-issue .github/actions/checkout-repos
```

- [ ] **Step 2: Write setup action.yml**

Create `.github/actions/setup/action.yml`:

```yaml
name: 'Setup OpenCode Environment'
description: 'Initialize opencode environment with API key configuration'
inputs:
  opencode-api-key:
    description: 'OpenCode API key for authentication'
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

- [ ] **Step 3: Commit**

```bash
git add .github/actions/setup/action.yml
git commit -m "feat: add setup composite action"
```

---

### Task 2: Create fetch-issue composite action

**Files:**
- Create: `.github/actions/fetch-issue/action.yml`

- [ ] **Step 1: Write fetch-issue action.yml**

Create `.github/actions/fetch-issue/action.yml`:

```yaml
name: 'Fetch Issue Content'
description: 'Fetch issue title and body using GitHub CLI'
inputs:
  issue-number:
    description: 'Issue number to fetch'
    required: true
  github-token:
    description: 'GitHub token for API access'
    required: true
outputs:
  title:
    description: 'Issue title'
    value: ${{ steps.fetch.outputs.title }}
  body:
    description: 'Issue body'
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

- [ ] **Step 2: Commit**

```bash
git add .github/actions/fetch-issue/action.yml
git commit -m "feat: add fetch-issue composite action"
```

---

### Task 3: Create checkout-repos composite action

**Files:**
- Create: `.github/actions/checkout-repos/action.yml`

- [ ] **Step 1: Write checkout-repos action.yml**

Create `.github/actions/checkout-repos/action.yml`:

```yaml
name: 'Checkout Target Repositories'
description: 'Checkout APIMagic and/or datastat-website based on dev_type'
inputs:
  dev-type:
    description: 'Development type: api-only, frontend-only, or both'
    required: true
  mode:
    description: 'Checkout mode: analysis (sparse) or full'
    required: false
    default: 'full'
  cross-repo-token:
    description: 'Token for cross-repository access'
    required: true
  repository-owner:
    description: 'Repository owner/organization'
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

- [ ] **Step 2: Commit**

```bash
git add .github/actions/checkout-repos/action.yml
git commit -m "feat: add checkout-repos composite action"
```

---

### Task 4: Create issue-analyzer.yml workflow

**Files:**
- Create: `.github/workflows/issue-analyzer.yml`

- [ ] **Step 1: Write issue-analyzer.yml**

Create `.github/workflows/issue-analyzer.yml`:

```yaml
name: Issue Analyzer

on:
  issues:
    types: [labeled]

concurrency:
  group: issue-analyzer-${{ github.event.issue.number }}
  cancel-in-progress: false

permissions:
  issues: write
  contents: read

jobs:
  analyze:
    if: github.event.label.name == 'accept' && github.event.issue.state == 'open'
    runs-on: self-hosted
    container:
      image: ghcr.io/${{ github.repository_owner }}/opencode-dev:latest
      env:
        HOME: /home/runner
    timeout-minutes: 10
    outputs:
      dev_type: ${{ steps.parse.outputs.dev_type }}
    
    steps:
      - name: Checkout changelog
        uses: actions/checkout@v4

      - name: Setup
        uses: ./.github/actions/setup
        with:
          opencode-api-key: ${{ secrets.OPENCODE_API_KEY }}

      - name: Fetch issue
        uses: ./.github/actions/fetch-issue
        with:
          issue-number: ${{ github.event.issue.number }}
          github-token: ${{ github.token }}

      - name: Determine dev type
        id: dev-type
        run: |
          opencode run "判断以下 Issue 需要开发哪些仓库（只写一行：api-only/frontend-only/both）：

          $(cat /tmp/opencode/issue.txt)" --model alibaba-cn/glm-5 --agent build --dangerously-skip-permissions --thinking=false || true
          cat /tmp/opencode/dev_type.txt 2>/dev/null || echo "both" > /tmp/opencode/dev_type.txt

      - name: Parse dev type
        id: parse
        run: |
          DEV_TYPE=$(head -1 /tmp/opencode/dev_type.txt 2>/dev/null || echo "both")
          case "$DEV_TYPE" in api-only|frontend-only|both) ;; *) DEV_TYPE="both";; esac
          echo "dev_type=$DEV_TYPE" >> $GITHUB_OUTPUT

      - name: Checkout target repos for analysis
        uses: ./.github/actions/checkout-repos
        with:
          dev-type: ${{ steps.parse.outputs.dev_type }}
          mode: analysis
          cross-repo-token: ${{ secrets.CROSS_REPO_TOKEN }}
          repository-owner: ${{ github.repository_owner }}

      - name: Generate design documents
        id: design
        env:
          DEV_TYPE: ${{ steps.parse.outputs.dev_type }}
        run: |
          REPO_INFO=""
          [ "$DEV_TYPE" = "api-only" ] || [ "$DEV_TYPE" = "both" ] && REPO_INFO+="APIMagic API目录: $(ls APIMagic/magic-api/api/ 2>/dev/null)\n"
          [ "$DEV_TYPE" = "frontend-only" ] || [ "$DEV_TYPE" = "both" ] && REPO_INFO+="datastat-website views: $(ls datastat-website/src/views/ 2>/dev/null)\n"
          
          opencode run "根据 Issue 和目标仓库结构生成设计文档：

          Issue: $(cat /tmp/opencode/issue.txt)

          仓库结构: $REPO_INFO

          写入：
          1. /tmp/opencode/requirements.md（需求分析）
          2. /tmp/opencode/design.md（设计方案，含具体文件路径和代码示例）" --model alibaba-cn/glm-5 --agent build --dangerously-skip-permissions --thinking=false || true

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: issue-${{ github.event.issue.number }}-requirements
          path: /tmp/opencode/requirements.md
          retention-days: 30

      - name: Upload design artifact
        uses: actions/upload-artifact@v4
        with:
          name: issue-${{ github.event.issue.number }}-design
          path: /tmp/opencode/design.md
          retention-days: 30

      - name: Upload dev_type artifact
        uses: actions/upload-artifact@v4
        with:
          name: issue-${{ github.event.issue.number }}-dev-type
          path: /tmp/opencode/dev_type.txt
          retention-days: 30

      - name: Add dev_type label
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          DEV_TYPE: ${{ steps.parse.outputs.dev_type }}
        run: gh issue edit $ISSUE_NUMBER --add-label "$DEV_TYPE"

      - name: Comment design documents
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          DEV_TYPE: ${{ steps.parse.outputs.dev_type }}
        run: |
          gh issue comment $ISSUE_NUMBER --body "✅ **需求分析完成** (开发类型: $DEV_TYPE)
          
          请审核后添加 \`approved\` 标签触发开发。"
          
          [ -f /tmp/opencode/requirements.md ] && gh issue comment $ISSUE_NUMBER --body "$(cat /tmp/opencode/requirements.md)"
          [ -f /tmp/opencode/design.md ] && gh issue comment $ISSUE_NUMBER --body "$(cat /tmp/opencode/design.md)"
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/issue-analyzer.yml
git commit -m "feat: add issue-analyzer workflow"
```

---

### Task 5: Create issue-developer.yml workflow

**Files:**
- Create: `.github/workflows/issue-developer.yml`

- [ ] **Step 1: Write issue-developer.yml**

Create `.github/workflows/issue-developer.yml`:

```yaml
name: Issue Developer

on:
  issues:
    types: [labeled]

concurrency:
  group: issue-developer-${{ github.event.issue.number }}
  cancel-in-progress: false

permissions:
  issues: write
  contents: read

jobs:
  develop:
    if: github.event.label.name == 'approved' && github.event.issue.state == 'open'
    runs-on: self-hosted
    container:
      image: ghcr.io/${{ github.repository_owner }}/opencode-dev:latest
      env:
        HOME: /home/runner
    timeout-minutes: 30
    
    steps:
      - name: Checkout changelog
        uses: actions/checkout@v4

      - name: Setup
        uses: ./.github/actions/setup
        with:
          opencode-api-key: ${{ secrets.OPENCODE_API_KEY }}

      - name: Configure git
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
        shell: bash

      - name: Fetch issue
        uses: ./.github/actions/fetch-issue
        with:
          issue-number: ${{ github.event.issue.number }}
          github-token: ${{ github.token }}

      - name: Download requirements artifact
        uses: actions/download-artifact@v4
        with:
          name: issue-${{ github.event.issue.number }}-requirements
          path: /tmp/opencode
        continue-on-error: true

      - name: Download design artifact
        uses: actions/download-artifact@v4
        with:
          name: issue-${{ github.event.issue.number }}-design
          path: /tmp/opencode
        continue-on-error: true

      - name: Download dev_type artifact
        uses: actions/download-artifact@v4
        with:
          name: issue-${{ github.event.issue.number }}-dev-type
          path: /tmp/opencode
        continue-on-error: true

      - name: Determine dev type
        id: dev-type
        run: |
          DEV_TYPE=$(head -1 /tmp/opencode/dev_type.txt 2>/dev/null || echo "both")
          case "$DEV_TYPE" in api-only|frontend-only|both) ;; *) DEV_TYPE="both";; esac
          echo "dev_type=$DEV_TYPE" >> $GITHUB_OUTPUT

      - name: Checkout target repos for development
        uses: ./.github/actions/checkout-repos
        with:
          dev-type: ${{ steps.dev-type.outputs.dev_type }}
          mode: full
          cross-repo-token: ${{ secrets.CROSS_REPO_TOKEN }}
          repository-owner: ${{ github.repository_owner }}

      - name: Develop API
        if: steps.dev-type.outputs.dev_type == 'api-only' || steps.dev-type.outputs.dev_type == 'both'
        id: api
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          BRANCH="issue-$ISSUE_NUMBER-api"
          cd APIMagic
          git checkout -b $BRANCH 2>/dev/null || git checkout $BRANCH
          git config --global --add safe.directory $GITHUB_WORKSPACE/APIMagic
          
          DESIGN=$(cat /tmp/opencode/design.md 2>/dev/null || echo "")
          REQ=$(cat /tmp/opencode/requirements.md 2>/dev/null || echo "")
          TITLE=$(cat /tmp/opencode/title.txt 2>/dev/null || echo "")
          BODY=$(cat /tmp/opencode/body.txt 2>/dev/null || echo "")
          
          [ -n "$DESIGN" ] && [ -n "$REQ" ] && opencode run "根据设计文档开发 API：

          Issue: $TITLE
          $BODY
          
          设计方案: $DESIGN
          需求分析: $REQ

          参考 CLAUDE.md 和现有 .ms 文件格式，创建/修改 magic-api/api/ 下的接口文件。" --model alibaba-cn/glm-5 --agent build --dangerously-skip-permissions --thinking=false || true
          
          git checkout -- .github/workflows/ 2>/dev/null || true
          if [ -n "$(git status --porcelain)" ]; then
            git add -A
            git commit -m "API for issue #$ISSUE_NUMBER"
            git push -u origin $BRANCH
            echo "has_changes=true" >> $GITHUB_OUTPUT
          else
            echo "has_changes=false" >> $GITHUB_OUTPUT
          fi
        shell: bash

      - name: Develop Frontend
        if: steps.dev-type.outputs.dev_type == 'frontend-only' || steps.dev-type.outputs.dev_type == 'both'
        id: frontend
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          BRANCH="issue-$ISSUE_NUMBER-frontend"
          cd datastat-website
          git checkout -b $BRANCH 2>/dev/null || git checkout $BRANCH
          git config --global --add safe.directory $GITHUB_WORKSPACE/datastat-website
          
          DESIGN=$(cat /tmp/opencode/design.md 2>/dev/null || echo "")
          REQ=$(cat /tmp/opencode/requirements.md 2>/dev/null || echo "")
          TITLE=$(cat /tmp/opencode/title.txt 2>/dev/null || echo "")
          BODY=$(cat /tmp/opencode/body.txt 2>/dev/null || echo "")
          
          [ -n "$DESIGN" ] && [ -n "$REQ" ] && opencode run "根据设计文档开发前端：

          Issue: $TITLE
          $BODY
          
          设计方案: $DESIGN
          需求分析: $REQ

          参考 CLAUDE.md 目录结构，修改对应的 Vue 组件。" --model alibaba-cn/glm-5 --agent build --dangerously-skip-permissions --thinking=false || true
          
          git checkout -- .github/workflows/ 2>/dev/null || true
          if [ -n "$(git status --porcelain)" ]; then
            git add -A
            git commit -m "Frontend for issue #$ISSUE_NUMBER"
            git push -u origin $BRANCH
            echo "has_changes=true" >> $GITHUB_OUTPUT
          else
            echo "has_changes=false" >> $GITHUB_OUTPUT
          fi
        shell: bash

      - name: Create PRs
        env:
          GH_TOKEN: ${{ secrets.CROSS_REPO_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          DEV_TYPE: ${{ steps.dev-type.outputs.dev_type }}
          API_HAS: ${{ steps.api.outputs.has_changes }}
          FRONTEND_HAS: ${{ steps.frontend.outputs.has_changes }}
        run: |
          RESULT=""
          if [ "$DEV_TYPE" = "api-only" ] || [ "$DEV_TYPE" = "both" ]; then
            if [ "$API_HAS" = "true" ]; then
              cd APIMagic
              PR=$(gh pr create --title "API for issue #$ISSUE_NUMBER" --body "来自 changelog issue #$ISSUE_NUMBER" --base main)
              cd ..
              RESULT+="APIMagic PR: $PR\n"
            else
              RESULT+="APIMagic: 无变更\n"
            fi
          fi
          if [ "$DEV_TYPE" = "frontend-only" ] || [ "$DEV_TYPE" = "both" ]; then
            if [ "$FRONTEND_HAS" = "true" ]; then
              cd datastat-website
              PR=$(gh pr create --title "Frontend for issue #$ISSUE_NUMBER" --body "来自 changelog issue #$ISSUE_NUMBER" --base main)
              cd ..
              RESULT+="datastat-website PR: $PR\n"
            else
              RESULT+="datastat-website: 无变更\n"
            fi
          fi
          gh issue comment $ISSUE_NUMBER --body "🚀 **开发完成**

$RESULT"
        shell: bash
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/issue-developer.yml
git commit -m "feat: add issue-developer workflow"
```

---

### Task 6: Remove old workflow

**Files:**
- Delete: `.github/workflows/issue-workflow.yml`

- [ ] **Step 1: Delete old workflow**

```bash
git rm .github/workflows/issue-workflow.yml
git commit -m "refactor: remove old issue-workflow.yml"
```

---

### Task 7: Final verification

- [ ] **Step 1: Verify file structure**

```bash
find .github -type f -name "*.yml" | sort
```

Expected output:
```
.github/actions/checkout-repos/action.yml
.github/actions/fetch-issue/action.yml
.github/actions/setup/action.yml
.github/workflows/issue-analyzer.yml
.github/workflows/issue-developer.yml
```

- [ ] **Step 2: Verify YAML syntax**

```bash
for f in .github/actions/*/action.yml .github/workflows/*.yml; do
  echo "Checking $f"
  python3 -c "import yaml; yaml.safe_load(open('$f'))" && echo "OK" || echo "FAILED"
done
```

---

## Testing Checklist

After deployment, test with:

1. Create a new issue, add `accept` label → verify analyzer runs and uploads artifacts
2. Check issue comments for design documents
3. Add `approved` label → verify developer runs and downloads artifacts
4. Verify correct repos are checked out based on dev_type
5. Verify PRs are created in target repositories