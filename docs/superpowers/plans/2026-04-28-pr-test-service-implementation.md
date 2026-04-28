# PR Test Service Coordination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement coordinated PR test service startup between datastat-website and APIMagic with automatic branch matching.

**Architecture:** Port ranges separated by service type (APIMagic: 10000+, datastat: 20000+). datastat-website workflow detects APIMagic branch, waits for or triggers corresponding service, and configures VITE_MAGICAPI_TARGET accordingly.

**Tech Stack:** GitHub Actions, shell scripts, GitHub CLI (gh), Docker

---

## Task 1: Update APIMagic pr-test-service.yml

**Files:**
- Modify: `opensourceways-test/APIMagic/.github/workflows/pr-test-service.yml`

**Changes:**
1. Port calculation: `9999 + PR_NUMBER` → `10000 + PR_NUMBER`
2. Status file: `pr-$PR_NUMBER.info` → `apimagic-pr-$PR_NUMBER.info`
3. Add branch info to status file

- [ ] **Step 1: Update port calculation**

Change line 36:
```yaml
          PORT=$((10000 + PR_NUMBER))
```

- [ ] **Step 2: Add branch name to outputs**

After line 38, add branch output:
```yaml
          BRANCH_NAME="${{ github.head_ref || github.ref_name }}"
          echo "branch=$BRANCH_NAME" >> $GITHUB_OUTPUT
```

- [ ] **Step 3: Update status file path and content**

Replace line 81:
```bash
          cat > /tmp/pr-services/apimagic-pr-$PR_NUMBER.info << EOF
port=$PORT
repo=APIMagic
pr_number=$PR_NUMBER
branch=${{ steps.pr-info.outputs.branch }}
started_at=$(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF
```

- [ ] **Step 4: Update comment to show port calculation**

Update comment section (lines 83-96) to reflect new port:
```yaml
      - name: Comment service URL
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ steps.pr-info.outputs.pr_number }}
          PORT: ${{ steps.pr-info.outputs.port }}
        run: |
          gh pr comment $PR_NUMBER --body "## 🧪 测试服务已启动

          **服务地址**: http://159.138.55.6:$PORT

          接口列表可通过 MagicAPI UI 查看: http://159.138.55.6:$PORT/magic/web/index.html

          > 服务将持续运行，如需停止请手动执行 \`Stop PR Test Service\` workflow。
          > 端口计算: 10000 + PR号
          "
```

- [ ] **Step 5: Commit changes**

```bash
cd opensourceways-test/APIMagic
git add .github/workflows/pr-test-service.yml
git commit -m "feat(workflow): update port to 10000+ range and add branch info to status file"
```

---

## Task 2: Update APIMagic stop-pr-test-service.yml

**Files:**
- Modify: `opensourceways-test/APIMagic/.github/workflows/stop-pr-test-service.yml`

- [ ] **Step 1: Update status file path**

Replace line 27:
```bash
          sudo rm -f /tmp/pr-services/apimagic-pr-$PR_NUMBER.info || true
```

- [ ] **Step 2: Commit changes**

```bash
cd opensourceways-test/APIMagic
git add .github/workflows/stop-pr-test-service.yml
git commit -m "fix(workflow): update status file path to match new naming convention"
```

---

## Task 3: Update datastat-website pr-test-service.yml

**Files:**
- Modify: `opensourceways-test/datastat-website/.github/workflows/pr-test-service.yml`

This is the major change. The complete updated file:

- [ ] **Step 1: Add new workflow_dispatch input**

Replace lines 6-10:
```yaml
  workflow_dispatch:
    inputs:
      pr_number:
        description: 'PR number (manual trigger)'
        required: true
      apimagic_ref:
        description: 'APIMagic reference (main, #123, or branch-name)'
        required: false
        default: ''
```

- [ ] **Step 2: Add new env variables at job level**

After line 16 (`jobs:`), before `start-service:`, add:
```yaml
env:
  APIMAGIC_REPO: opensourceways-test/APIMagic
```

- [ ] **Step 3: Update port calculation**

Replace line 36:
```yaml
          PORT=$((20000 + PR_NUMBER))
```

- [ ] **Step 4: Add branch name output**

After line 38, add:
```yaml
          BRANCH_NAME="${{ github.head_ref || github.ref_name }}"
          echo "branch=$BRANCH_NAME" >> $GITHUB_OUTPUT
```

- [ ] **Step 5: Add step to get PR description for Depends-On parsing**

After the checkout step (line 40), add new step:
```yaml
      - name: Get PR info
        id: pr-desc
        if: github.event_name == 'pull_request'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          PR_DESC=$(gh pr view ${{ steps.pr-info.outputs.pr_number }} --json body --jq '.body' 2>/dev/null || echo "")
          DEPENDS_ON=$(echo "$PR_DESC" | grep -oP 'Depends-On:\s*APIMagic#\K[\w-]+' || true)
          echo "depends_on=${DEPENDS_ON}" >> $GITHUB_OUTPUT
          echo "pr_desc_fetched=true" >> $GITHUB_OUTPUT
```

- [ ] **Step 6: Add step to resolve APIMagic reference**

Add after the pr-desc step:
```yaml
      - name: Resolve APIMagic reference
        id: apimagic
        env:
          GH_TOKEN: ${{ github.token }}
          MANUAL_REF: ${{ github.event.inputs.apimagic_ref }}
          DEPENDS_ON: ${{ steps.pr-desc.outputs.depends_on }}
          BRANCH_NAME: ${{ steps.pr-info.outputs.branch }}
        run: |
          # Priority: manual input > Depends-On > auto-detect > main
          
          resolve_apimagic_pr() {
            local ref="$1"
            
            # If it's a PR number (starts with #)
            if [[ "$ref" =~ ^#([0-9]+)$ ]]; then
              echo "pr_number=${BASH_REMATCH[1]}"
              echo "port=$((10000 + BASH_REMATCH[1]))"
              return 0
            fi
            
            # If it's a branch name, find corresponding PR
            local pr_num=$(gh api repos/${{ env.APIMAGIC_REPO }}/pulls \
              --jq ".[] | select(.head.ref == \"$ref\") | .number" 2>/dev/null | head -1)
            
            if [ -n "$pr_num" ]; then
              echo "pr_number=$pr_num"
              echo "port=$((10000 + pr_num))"
              return 0
            fi
            
            return 1
          }
          
          check_branch_exists() {
            local branch="$1"
            gh api repos/${{ env.APIMAGIC_REPO }}/branches/"$branch" > /dev/null 2>&1
          }
          
          check_service_running() {
            local port="$1"
            curl -s --max-time 2 "http://localhost:$port/" > /dev/null 2>&1
          }
          
          # 1. Manual input takes priority
          if [ -n "$MANUAL_REF" ]; then
            echo "Using manual reference: $MANUAL_REF"
            if resolve_apimagic_pr "$MANUAL_REF"; then
              echo "mode=resolved"
              exit 0
            fi
          fi
          
          # 2. Depends-On from PR description
          if [ -n "$DEPENDS_ON" ]; then
            echo "Using Depends-On reference: APIMagic#$DEPENDS_ON"
            if resolve_apimagic_pr "$DEPENDS_ON"; then
              echo "mode=resolved"
              exit 0
            fi
          fi
          
          # 3. Auto-detect: check if same branch exists in APIMagic
          echo "Checking for matching branch: $BRANCH_NAME"
          if check_branch_exists "$BRANCH_NAME"; then
            echo "Branch $BRANCH_NAME exists in APIMagic"
            
            # Find PR for this branch
            PR_NUM=$(gh api repos/${{ env.APIMAGIC_REPO }}/pulls \
              --jq ".[] | select(.head.ref == \"$BRANCH_NAME\") | .number" 2>/dev/null | head -1)
            
            if [ -n "$PR_NUM" ]; then
              APIMAGIC_PORT=$((10000 + PR_NUM))
              
              # Check if service is already running
              if check_service_running "$APIMAGIC_PORT"; then
                echo "pr_number=$PR_NUM"
                echo "port=$APIMAGIC_PORT"
                echo "mode=running"
                exit 0
              fi
              
              # Need to trigger APIMagic workflow
              echo "pr_number=$PR_NUM"
              echo "port=$APIMAGIC_PORT"
              echo "mode=trigger"
              exit 0
            fi
          fi
          
          # 4. Default to main
          echo "Using APIMagic main branch"
          echo "pr_number="
          echo "port=9999"
          echo "mode=main"
```

- [ ] **Step 7: Add step to trigger APIMagic workflow if needed**

Add after the resolve step:
```yaml
      - name: Trigger APIMagic service if needed
        if: steps.apimagic.outputs.mode == 'trigger'
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ steps.apimagic.outputs.pr_number }}
        run: |
          echo "Triggering APIMagic workflow for PR #$PR_NUMBER..."
          gh workflow run pr-test-service.yml \
            --repo ${{ env.APIMAGIC_REPO }} \
            -f pr_number=$PR_NUMBER
          
          # Wait for service to be ready
          PORT=$((10000 + PR_NUMBER))
          echo "Waiting for APIMagic service on port $PORT..."
          
          for i in $(seq 1 60); do
            if curl -s --max-time 2 "http://localhost:$PORT/" > /dev/null 2>&1; then
              echo "APIMagic service ready on port $PORT"
              exit 0
            fi
            echo "Waiting... ($i/60)"
            sleep 5
          done
          
          echo "::warning::APIMagic service did not start within timeout"
```

- [ ] **Step 8: Replace Start test service step**

Replace the entire "Start test service" step (lines 42-77):
```yaml
      - name: Start test service
        env:
          PR_NUMBER: ${{ steps.pr-info.outputs.pr_number }}
          PORT: ${{ steps.pr-info.outputs.port }}
          APIMAGIC_PORT: ${{ steps.apimagic.outputs.port }}
        run: |
          mkdir -p /tmp/pr-services
          
          # Wait for APIMagic if it was just triggered
          if [ "${{ steps.apimagic.outputs.mode }}" = "trigger" ]; then
            echo "APIMagic should be ready on port $APIMAGIC_PORT"
          fi
          
          sudo docker run -d \
            --name datastat-test-$PR_NUMBER \
            --network host \
            -v $GITHUB_WORKSPACE:/app \
            -v /tmp/pr-services:/tmp/pr-services \
            -w /app \
            -e VITE_MAGICAPI_TARGET=http://localhost:$APIMAGIC_PORT/ \
            ghcr.io/${{ github.repository_owner }}/opencode-dev:latest \
            sh -c "npm install && pnpm dev --port $PORT --host 0.0.0.0"
          
          echo "Starting datastat-website on port $PORT..."
          echo "Connecting to APIMagic on port $APIMAGIC_PORT"
          
          for i in {1..30}; do
            if curl -s --max-time 2 http://localhost:$PORT/ > /dev/null 2>&1; then
              echo "Service ready!"
              break
            fi
            if [ $i -eq 30 ]; then
              echo "Service failed to start"
              sudo docker logs datastat-test-$PR_NUMBER 2>&1 | tail -100
              sudo docker stop datastat-test-$PR_NUMBER || true
              sudo docker rm datastat-test-$PR_NUMBER || true
              exit 1
            fi
            sleep 2
          done
          
          cat > /tmp/pr-services/datastat-pr-$PR_NUMBER.info << EOF
          port=$PORT
          repo=datastat-website
          pr_number=$PR_NUMBER
          branch=${{ steps.pr-info.outputs.branch }}
          apimagic_port=$APIMAGIC_PORT
          started_at=$(date -u +%Y-%m-%dT%H:%M:%SZ)
          EOF
```

- [ ] **Step 9: Update Comment service URL step**

Replace lines 79-90:
```yaml
      - name: Comment service URL
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ steps.pr-info.outputs.pr_number }}
          PORT: ${{ steps.pr-info.outputs.port }}
          APIMAGIC_PORT: ${{ steps.apimagic.outputs.port }}
          APIMAGIC_MODE: ${{ steps.apimagic.outputs.mode }}
        run: |
          COMMENT_BODY="## 🧪 测试服务已启动

          **服务地址**: http://159.138.55.6:$PORT

          **APIMagic 接口地址**: http://159.138.55.6:$APIMAGIC_PORT"
          
          if [ "$APIMAGIC_MODE" = "trigger" ]; then
            COMMENT_BODY="$COMMENT_BODY

          ⚠️ APIMagic 服务已自动启动"
          elif [ "$APIMAGIC_MODE" = "running" ]; then
            COMMENT_BODY="$COMMENT_BODY

          ✅ APIMagic 服务已就绪"
          fi
          
          COMMENT_BODY="$COMMENT_BODY

          > 服务将持续运行，如需停止请手动执行 \`Stop PR Test Service\` workflow。
          > 端口计算: 20000 + PR号"
          
          gh pr comment $PR_NUMBER --body "$COMMENT_BODY"
```

- [ ] **Step 10: Commit changes**

```bash
cd opensourceways-test/datastat-website
git add .github/workflows/pr-test-service.yml
git commit -m "feat(workflow): add APIMagic service coordination with auto branch detection"
```

---

## Task 4: Update datastat-website stop-pr-test-service.yml

**Files:**
- Modify: `opensourceways-test/datastat-website/.github/workflows/stop-pr-test-service.yml`

- [ ] **Step 1: Update status file path**

Replace line 27:
```bash
          sudo rm -f /tmp/pr-services/datastat-pr-$PR_NUMBER.info || true
```

- [ ] **Step 2: Commit changes**

```bash
cd opensourceways-test/datastat-website
git add .github/workflows/stop-pr-test-service.yml
git commit -m "fix(workflow): update status file path to match new naming convention"
```

---

## Task 5: Verification

- [ ] **Step 1: Verify APIMagic workflow syntax**

```bash
cd opensourceways-test/APIMagic
# Check YAML syntax
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/pr-test-service.yml'))"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/stop-pr-test-service.yml'))"
```

- [ ] **Step 2: Verify datastat-website workflow syntax**

```bash
cd opensourceways-test/datastat-website
# Check YAML syntax
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/pr-test-service.yml'))"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/stop-pr-test-service.yml'))"
```

- [ ] **Step 3: Push changes to changelog repo**

```bash
cd /home/wcp/opensourceways-test/changelog
git add docs/superpowers/plans/2026-04-28-pr-test-service-implementation.md
git commit -m "docs: add implementation plan for PR test service coordination"
```

---

## Summary of Changes

| File | Change Type | Key Changes |
|------|-------------|-------------|
| APIMagic/pr-test-service.yml | Modify | Port: 10000+PR, status file with branch info |
| APIMagic/stop-pr-test-service.yml | Modify | Status file path update |
| datastat-website/pr-test-service.yml | Major | Port: 20000+PR, APIMagic coordination, branch detection |
| datastat-website/stop-pr-test-service.yml | Modify | Status file path update |

## Testing Checklist

After deployment, test the following scenarios:

- [ ] **Scenario 1a**: Create datastat-website PR with no matching APIMagic branch → should connect to APIMagic main (port 9999)
- [ ] **Scenario 1b**: Create datastat-website PR with matching APIMagic branch (service already running) → should connect to existing APIMagic PR service
- [ ] **Scenario 2a**: Create PRs in both repos with same branch name → datastat should auto-detect and wait for APIMagic
- [ ] **Scenario 2b**: Use Depends-On in PR description → should override auto-detection
- [ ] **Manual override**: Use workflow_dispatch with apimagic_ref → should use specified reference
- [ ] **Port verification**: Verify APIMagic PR uses 10000+ range, datastat uses 20000+ range