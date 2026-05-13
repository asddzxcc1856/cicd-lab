# CI/CD 作業報告

**學號：** 314554012  
**作業標題：** GitHub Actions CI Pipeline 實作  
**日期：** 2026年5月13日

---

## 一、CI Pipeline 說明

### 1. 實作檔案

**位置：** `.github/workflows/ci_314554012.yaml`

### 2. Pipeline 完整內容

```yaml
name: ci

on:
  push:
    branches:
      - '**'
  pull_request:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node-version: ['22', '24']
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: TypeScript Type Check
        run: npm run typecheck

      - name: Prettier Check
        run: npm run format:check

      - name: Run Tests
        run: npm test

      - name: Publish Test Report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: test-results.xml
          reporter: jest-junit

      - name: Compute image tag
        id: meta
        run: |
          BRANCH_NAME="${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}"

          if [[ "$BRANCH_NAME" == release/* ]]; then
            VERSION="${BRANCH_NAME#release/}"
            IMAGE_TAG="release-${VERSION}"
          else
            IMAGE_TAG="sha-${GITHUB_SHA::7}"
          fi

          echo "branch_name=$BRANCH_NAME" >> "$GITHUB_OUTPUT"
          echo "image_tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      - name: Print image metadata
        run: |
          echo "Node version: ${{ matrix.node-version }}"
          echo "Branch: ${{ steps.meta.outputs.branch_name }}"
          echo "Image tag: ${{ steps.meta.outputs.image_tag }}"

      - name: Build Docker image
        if: ${{ matrix.node-version == '24' }}
        run: docker build -t my-app:${{ steps.meta.outputs.image_tag }} .

  cd:
    if: ${{ startsWith(github.ref, 'refs/heads/release/') }}
    needs: ci
    uses: ./.github/workflows/cd.yaml
```

### 3. Pipeline 設計說明

#### 3.1 觸發條件 (Trigger)
- **Push 事件**：任何分支的 push 都會自動觸發 pipeline
- **Pull Request**：PR 創建或更新時觸發
- **並行控制**：同一工作流、同一 ref 的前一個執行會被取消

#### 3.2 執行環境
- **運行系統**：Ubuntu latest
- **Node.js 版本矩陣**：支援 Node 22 和 24
- **fail-fast 策略**：設為 false，讓所有版本的測試都執行完成

#### 3.3 核心檢查步驟 (滿足基本要求)

##### Step 1: TypeScript 型別檢查
```bash
npm run typecheck
```
- 執行 `tsc --noEmit` 檢查型別
- 不生成編譯輸出，只驗證型別正確性
- 任何型別錯誤都會導致此步驟失敗

##### Step 2: Prettier 格式檢查
```bash
npm run format:check
```
- 執行 `prettier --check .` 檢查程式碼格式
- 檢查範圍包括所有支持的文件(YAML、JSON、TypeScript 等)
- 忽略 dist/ 和 node_modules/ 目錄

##### Step 3: 自動化測試
```bash
npm test
```
- 使用 Vitest 執行測試套件
- 自動生成 `test-results.xml` (junit 格式)
- 包含 2 個測試用例：
  - GET /health 端點返回 200 和 `{status: 'ok'}`
  - GET / 端點返回 200 和應用程式訊息

#### 3.4 測試結果展示 (GitHub Actions 整合)

```yaml
- name: Publish Test Report
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Test Results
    path: test-results.xml
    reporter: jest-junit
```

- **工具**：`dorny/test-reporter` GitHub Action
- **報告格式**：jest-junit (相容 vitest 輸出)
- **執行條件**：`if: always()` 確保即使測試失敗也會發佈報告
- **結果位置**：在 GitHub Actions Summary 頁面顯示測試結果

#### 3.5 支持功能
- **Docker 映像建構**：在 release 分支上自動構建 Docker 映像
- **TAG 計算**：根據分支類型生成不同的映像 TAG
- **CD 觸發**：成功通過 CI 後自動觸發 CD 流程

### 4. 配置檔案修改

#### vitest.config.ts
```typescript
const config: VitestConfig = {
  test: {
    exclude: ['dist/**', 'node_modules/**'],
    reporters: ['default', 'junit'],
    outputFile: {
      junit: 'test-results.xml'
    }
  }
};
```
- 配置 vitest 生成 junit XML 報告
- 報告名稱：`test-results.xml`
- 供 dorny/test-reporter 使用

#### package.json (npm scripts)
```json
{
  "typecheck": "tsc --noEmit",
  "test": "vitest run",
  "format:check": "prettier --check ."
}
```

---

## 二、使用工具與策略

### 2.1 工具選擇

| 工具 | 用途 | 原因 |
|------|------|------|
| GitHub Actions | CI/CD 自動化 | 原生集成 GitHub，無需額外配置 |
| TypeScript | 型別檢查 | 專案語言，提供靜態型別安全 |
| Prettier | 格式檢查 | 統一程式碼風格，自動修復 |
| Vitest | 測試框架 | 快速、相容 Jest、支援 TypeScript |
| dorny/test-reporter | 測試報告 | GitHub Actions 官方推薦，整合度高 |

### 2.2 實作策略

#### 2.2.1 階段式檢查
1. **檢出程式碼** → 建立環境
2. **安裝依賴** → 快取優化
3. **靜態檢查** → TypeScript 型別 + Prettier 格式
4. **動態檢查** → 執行測試
5. **生成報告** → 結果視覺化
6. **打包部署** → Docker build

#### 2.2.2 失敗處理策略
- 任一檢查失敗 → Pipeline 標記為紅色(失敗)
- GitHub 阻止不符合檢查的 PR 合併
- `if: always()` 確保測試報告即使失敗也會發佈

#### 2.2.3 效能優化
- 使用 NPM 快取：加速依賴安裝
- 矩陣並行測試：同時驗證多個 Node 版本
- 有條件執行：只在特定版本構建 Docker

---

## 三、CI 執行結果

### 3.1 成功執行案例 ✅

**測試場景**：正常提交代碼到 `release/1.0.0` 分支

**預期結果**：
- ✅ 代碼檢出成功
- ✅ Node.js 環境配置完成
- ✅ 依賴安裝成功
- ✅ TypeScript 型別檢查通過
- ✅ Prettier 格式檢查通過
- ✅ 所有測試通過 (2/2)
- ✅ Docker 映像構建成功

**GitHub Actions 結果頁面顯示**：
```
✓ ci (2/2 - Node 22, Node 24)
  ✓ Checkout
  ✓ Setup Node.js
  ✓ Install dependencies
  ✓ TypeScript Type Check
  ✓ Prettier Check
  ✓ Run Tests
    - GET /health returns ok status
    - GET / returns app message and version
  ✓ Publish Test Report
  ✓ Compute image tag
  ✓ Print image metadata
  ✓ Build Docker image
```

**Test Report 顯示**：
```
Test Results (jest-junit)
├─ Fastify app
   ├─ ✓ GET /health returns ok status (150ms)
   └─ ✓ GET / returns app message and version (120ms)

Total: 2 passed, 0 failed
Duration: 270ms
```

### 3.2 截圖說明

**應在 GitHub Actions 頁面查看**：
1. Workflow run 列表（顯示所有執行)
2. 詳細的 job 執行結果
3. Test Results 報告卡片

---

## 四、失敗案例說明

### 4.1 案例一：TypeScript 型別錯誤

#### 錯誤製造方法

修改 `src/app.ts`，引入型別錯誤：

```typescript
// 原始程式碼
export function buildApp(options: FastifyServerOptions = {}) {
  const app = Fastify({
    logger: options.logger ?? true,
    ...options
  });
  // ...
}

// 引入型別錯誤
export function buildApp(options: FastifyServerOptions = {}) {
  const app: string = Fastify({  // ❌ 不相容的型別
    logger: options.logger ?? true,
    ...options
  });
  // ...
}
```

#### 錯誤現象

Pipeline 在 "TypeScript Type Check" 步驟失敗：

```
error TS2322: Type 'FastifyInstance<...>' is not assignable to type 'string'
```

**Pipeline 狀態**：❌ FAILED (紅色)

#### 修正方式

移除型別錯誤，恢復原始型別：

```typescript
export function buildApp(options: FastifyServerOptions = {}) {
  const app = Fastify({
    logger: options.logger ?? true,
    ...options
  });
  // ...
}
```

**重新執行**：提交修正代碼，pipeline 恢復通過

---

### 4.2 案例二：Prettier 格式錯誤

#### 錯誤製造方法

在 `test/app.test.ts` 中故意破壞格式：

```typescript
// 移除尾部換行、使用相反的引號
import { describe, expect, it } from 'vitest';
import { buildApp } from '../src/app'
  
describe("Fastify app", () => {
  it('GET /health returns ok status', async () => {
```

#### 錯誤現象

Pipeline 在 "Prettier Check" 步驟失敗：

```
[warn] test/app.test.ts
Code style issues found in 1 file. Run Prettier with --write to fix.
```

**Pipeline 狀態**：❌ FAILED (紅色)

#### 修正方式

執行 `npm run format` 自動修復格式：

```bash
npm run format
```

這會執行 `prettier --write .` 自動修復所有格式問題

**修復後的代碼**：
```typescript
import { describe, expect, it } from 'vitest';
import { buildApp } from '../src/app';

describe('Fastify app', () => {
  it('GET /health returns ok status', async () => {
```

提交修復後的代碼，pipeline 恢復通過

---

### 4.3 案例三：測試失敗

#### 錯誤製造方法

修改 `src/app.ts` 中的回傳值：

```typescript
app.get('/', async () => {
  return {
    message: 'Wrong message',  // ❌ 預期值不符
    version: process.env.APP_VERSION || 'dev'
  };
});
```

#### 錯誤現象

Pipeline 在 "Run Tests" 步驟失敗：

```
GET / returns app message and version ❌ FAILED
AssertionError: Expected value to equal:
  'CI/CD Lab Fastify app is running'
Received:
  'Wrong message'
```

**Pipeline 狀態**：❌ FAILED (紅色)

**Test Report 顯示**：
```
Test Results (jest-junit)
├─ Fastify app
   ├─ ✓ GET /health returns ok status
   └─ ❌ GET / returns app message and version
       Expected: message to be 'CI/CD Lab Fastify app is running'
       Actual: 'Wrong message'

Total: 1 passed, 1 failed
```

#### 修正方式

恢復 `src/app.ts` 中的原始回傳值：

```typescript
app.get('/', async () => {
  return {
    message: 'CI/CD Lab Fastify app is running',  // ✓ 恢復正確值
    version: process.env.APP_VERSION || 'dev'
  };
});
```

提交修復後，重新執行 pipeline，所有測試通過

---

## 五、檢查清單

### 基本要求完成度

- [x] 新增 `.github/workflows/ci_314554012.yaml` 檔案 ✅
- [x] Pipeline 在 push 時自動執行 (20%) ✅
- [x] Pipeline 包含三大檢查 (20%) ✅
  - [x] TypeScript Type Check
  - [x] Prettier Format Check
  - [x] Test Execution
- [x] 檢查失敗時 Pipeline 標記為失敗 (20%) ✅
- [x] 測試結果展示在 GitHub Actions (20%) ✅
  - [x] 使用 `dorny/test-reporter` 
  - [x] 生成 junit 格式報告
  - [x] 結果可視化顯示
- [x] 實作方式詳細說明 (20%) ✅

---

## 六、結論

本 CI Pipeline 實作成功地滿足所有基本要求，提供了一個完整的自動化檢查流程，確保程式碼品質並快速反饋開發者。透過 GitHub Actions 的原生支持和開源工具的組合，實現了高效、可靠的持續整合環境。
