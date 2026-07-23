# Claude Code Headless CI/CD 鎸囧崡

## 鏍稿績姒傚康

| 妯″紡 | 鍛戒护 | 鏄惁鎵ц宸ュ叿 | 閫傜敤鍦烘櫙 |
|------|------|-------------|---------|
| **Print 妯″紡** | `claude -p --output-format json` | 鉂?鍙緭鍑烘枃鏈?| 绾枃鏈敓鎴愩€侀棶绛斻€佸垎鏋?|
| **Stream 妯″紡** | `claude --output-format stream-json --verbose` | 鉁?鎵ц宸ュ叿 | 鏂囦欢鍒涘缓銆佷唬鐮佷慨鏀广€侀」鐩搷浣?|

> **鍏抽敭鍙戠幇**: `-p` (print) 妯″紡涓嶆墽琛屾枃浠舵搷浣滐紝鍙繑鍥炴枃鏈€侰I 涓渶瑕佹搷浣滄枃浠舵椂蹇呴』鐢?stream 妯″紡銆?
## 鍓嶇疆鏉′欢

```bash
# 1. 瀹夎 Claude Code (npm)
npm install -g @anthropic-ai/claude-code

# 2. 璁剧疆 API Key (鎴栦娇鐢?OAuth)
export ANTHROPIC_API_KEY="sk-ant-..."

# 3. 楠岃瘉瀹夎
claude --version
```

---

## 鍦烘櫙 1: 绾枃鏈敓鎴?(Print 妯″紡)

閫傜敤浜庯細鐢熸垚鏂囨。銆佷唬鐮佸鏌ユ剰瑙併€佸垎鏋愭姤鍛婄瓑涓嶉渶瑕佸啓鏂囦欢鐨勪换鍔°€?
### GitHub Actions

```yaml
# .github/workflows/generate-docs.yml
name: Generate Documentation
on:
  push:
    branches: [main]

jobs:
  generate-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm install -g @anthropic-ai/claude-code

      - name: Generate API docs
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p --output-format json \
            --dangerously-skip-permissions \
            "Read src/api.ts and generate API documentation in markdown format" \
            > docs-output.json

      - name: Extract and save docs
        run: |
          cat docs-output.json | jq -r '.result' > docs/API.md

      - name: Commit docs
        run: |
          git config user.name "claude-bot"
          git config user.email "bot@example.com"
          git add docs/
          git diff --staged --quiet || git commit -m "Update API docs"
          git push
```

### GitLab CI

```yaml
# .gitlab-ci.yml
generate-docs:
  image: node:20
  script:
    - npm install -g @anthropic-ai/claude-code
    - |
      claude -p --output-format json \
        --dangerously-skip-permissions \
        "Review the code changes in this MR and write a summary to mr-summary.md" \
        > output.json
    - cat output.json | jq -r '.result' > mr-summary.md
  artifacts:
    paths:
      - mr-summary.md
```

---

## 鍦烘櫙 2: 鏂囦欢鎿嶄綔 (Stream 妯″紡)

閫傜敤浜庯細鍒涘缓鏂囦欢銆佷慨鏀逛唬鐮併€侀噸鏋勭瓑闇€瑕佸疄闄呮搷浣滅殑浠诲姟銆?
### GitHub Actions

```yaml
# .github/workflows/auto-refactor.yml
name: Auto Refactor
on:
  workflow_dispatch:
    inputs:
      task:
        description: 'Refactoring task'
        required: true

jobs:
  refactor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm install -g @anthropic-ai/claude-code

      - name: Run Claude refactor
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # Stream 妯″紡 - 瀹為檯鎵ц宸ュ叿鎿嶄綔
          echo "${{ github.event.inputs.task }}" | \
          claude --output-format stream-json \
                 --verbose \
                 --dangerously-skip-permissions \
                 > claude-events.jsonl

      - name: Check results
        run: |
          echo "=== Events summary ==="
          cat claude-events.jsonl | jq -r 'select(.type=="assistant") | .message.content[] | select(.type=="tool_use") | .name' | sort | uniq -c

      - name: Commit changes
        run: |
          git config user.name "claude-bot"
          git config user.email "bot@example.com"
          git add -A
          git diff --staged --quiet || git commit -m "Auto-refactor: ${{ github.event.inputs.task }}"
          git push
```

---

## 鍦烘櫙 3: 浠ｇ爜瀹℃煡

```yaml
# .github/workflows/code-review.yml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm install -g @anthropic-ai/claude-code

      - name: Get diff
        id: diff
        run: echo "diff=$(git diff origin/main...HEAD)" >> $GITHUB_OUTPUT

      - name: Review with Claude
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          DIFF: ${{ steps.diff.outputs.diff }}
        run: |
          echo "Review this PR diff and provide feedback:
          $DIFF" | \
          claude -p --output-format json \
                 --dangerously-skip-permissions \
                 > review.json

      - name: Post review comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = JSON.parse(fs.readFileSync('review.json', 'utf8'));
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Claude Code Review\n\n${review.result}`
            });
```

---

## 鍦烘櫙 4: 鑷姩娴嬭瘯鐢熸垚

```yaml
# .github/workflows/generate-tests.yml
name: Generate Tests
on:
  push:
    paths:
      - 'src/**/*.ts'

jobs:
  test-gen:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm install -g @anthropic-ai/claude-code
      - run: npm ci

      - name: Generate tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # 瀵规瘡涓慨鏀圭殑鏂囦欢鐢熸垚娴嬭瘯
          for file in $(git diff --name-only HEAD~1 | grep '\.ts$'); do
            test_file="tests/$(echo $file | sed 's|src/||; s|\.ts$|.test.ts|')"
            
            echo "Generating test for $file..."
            echo "Create unit tests for $file, output to $test_file" | \
            claude --output-format stream-json \
                   --verbose \
                   --dangerously-skip-permissions \
                   >> claude-events.jsonl
          done

      - name: Run generated tests
        run: npx vitest run --reporter=verbose
```

---

## 鍦烘櫙 5: Docker 涓繍琛?
```dockerfile
# Dockerfile
FROM node:20-slim

RUN npm install -g @anthropic-ai/claude-code

WORKDIR /workspace
COPY . .

# 绾枃鏈ā寮?CMD ["claude", "-p", "--dangerously-skip-permissions", \
     "Analyze the codebase and suggest improvements"]

# 鎴?Stream 妯″紡
# CMD ["claude", "--output-format", "stream-json", "--verbose", \
#      "--dangerously-skip-permissions"]
```

```yaml
# docker-compose.yml
services:
  claude:
    build: .
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - ./src:/workspace/src
```

---

## 杈撳嚭瑙ｆ瀽

### Print 妯″紡 (JSON)

```json
{
  "result": "鐢熸垚鐨勬枃鏈唴瀹?..",
  "total_cost_usd": 0.12,
  "duration_ms": 54000,
  "usage": { "input_tokens": 15000, "output_tokens": 200 }
}
```

瑙ｆ瀽锛歚cat output.json | jq -r '.result'`

### Stream 妯″紡 (JSONL)

姣忚涓€涓?JSON 浜嬩欢锛?
```jsonl
{"type":"system","subtype":"init","tools":["Write","Bash","Read"]}
{"type":"assistant","message":{"content":[{"type":"tool_use","name":"Write","input":{"file_path":"..."}}]}}
{"type":"user","message":{"content":[{"type":"tool_result","content":"File created"}]}}
{"type":"result","result":"浠诲姟瀹屾垚","total_cost_usd":0.15}
```

瑙ｆ瀽浜嬩欢锛?```bash
# 鎻愬彇宸ュ叿浣跨敤
cat events.jsonl | jq -r 'select(.type=="assistant") | .message.content[] | select(.type=="tool_use") | .name' | sort | uniq -c

# 鎻愬彇缁撴灉
cat events.jsonl | jq -r 'select(.type=="result") | .result'

# 妫€鏌ラ敊璇?cat events.jsonl | jq -r 'select(.type=="error")'
```

---

## 璐圭敤鎺у埗

```bash
# 璁剧疆鏈€澶ч绠?claude -p --max-budget-usd 1.00 "your prompt"

# 鍦?CI 涓缃幆澧冨彉閲?export CLAUDE_MAX_BUDGET_USD=5.00
```

---

## 甯歌闂

### Q: `-p` 妯″紡涓轰粈涔堜笉鍒涘缓鏂囦欢锛?A: `-p` (print) 鏄潪浜や簰妯″紡锛屽彧杈撳嚭鏂囨湰銆傞渶瑕佹枃浠舵搷浣滄椂鐢?`--output-format stream-json --verbose`锛堜笉鍔?`-p`锛夈€?
### Q: CI 涓繀椤荤敤 `--dangerously-skip-permissions` 鍚楋紵
A: 鏄殑锛孋I 鏄棤浜哄€煎畧鐜锛屾棤娉曞鐞嗕氦浜掑紡鏉冮檺纭銆?
### Q: 濡備綍閬垮厤 Claude 鎵ц鍗遍櫓鎿嶄綔锛?A: 
- 浣跨敤 `--allowedTools` 闄愬埗鍙敤宸ュ叿
- 鍦ㄤ唬鐮佷粨搴撲腑閰嶇疆 `.claude/settings.json`
- 璁剧疆 `--max-budget-usd` 鎺у埗璐圭敤

### Q: Stream 妯″紡鐨勮緭鍑哄浣曡В鏋愶紵
A: 姣忚涓€涓?JSON锛岀敤 `jq` 杩囨护銆傚叧閿簨浠剁被鍨嬶細`system`(鍒濆鍖?銆乣assistant`(妯″瀷杈撳嚭)銆乣user`(宸ュ叿缁撴灉)銆乣result`(鏈€缁堢粨鏋?銆?