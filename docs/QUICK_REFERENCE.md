# GitHub Workflow 快速參考

## branch 架構
```
main (繳交版本) <- dev (開發) <- test (測試) <- feat/xxx (功能)
```

## branch命名規範

| 前綴 | 用途 | 範例 |
|------|------|------|
| `feat/` | 新功能 | `feat/RAG_KG` |
| `fix/` | 錯誤修復 | `fix/agent_overtime` |
| `docs/` | 文檔更新 | `docs/api-guide` |
| `refactor/` | 程式碼重構 | `refactor/api-calls` |
| `perf/` | 性能優化 | `perf/time-loading` |
| `test/` | 測試相關 | `test/unit-tests` |
| `security/` | 安全修復 | `security/xss-fix` |
| `project/` | 個人專案 | `project/john/feature` |

**規則:** 小寫 + 連字符 + 描述清楚

---

## 完整工作流程

### 1. 建立branch
```bash
# 從test branch開新branch
git checkout test
git pull origin test
git checkout -b feat/your-feature
```

### 2. 開發與提交
```bash
# 開發程式碼...
git add file_name1 file_name2
git commit -m "feat: add user authentication"
git push origin feat/your-feature
```

### 3. 合併到 Test (測試)
```bash
# 在 GitHub 上創建 PR
# 標題格式: [TEST] feat: add user authentication
# 審查人數: 1人
# 合併方式: Squash and merge
```

### 4. 測試通過後合併到 Dev
```bash
# 在 GitHub 上創建 PR: test -> dev
# 標題格式: [DEV] Merge test - user authentication
# 審查人數: 1人
# 合併方式: Merge commit
```

### 5. 準備發布時合併到 Main
```bash
# 在 GitHub 上創建 PR: dev -> main
# 標題格式: [RELEASE] v1.2.0 - Feature release
# 審查人數: 1人
# 合併方式: Merge commit
```

---

## Commit Message 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type 類型
- `feat`: 新功能
- `fix`: 錯誤修復
- `docs`: 文檔變更
- `style`: 代碼格式
- `refactor`: 重構
- `perf`: 性能優化
- `test`: 測試相關
- `chore`: 構建過程

### 範例
```bash
feat(auth): add JWT token validation

- Implement token expiry check
- Add refresh token mechanism

Closes #123
```

---

## PR 標題格式

### 到 Test 分支
```
[TEST] <type>: <description>
```
範例: `[TEST] feat: add payment integration`

### 到 Dev 分支
```
[DEV] Merge test - <description>
```
範例: `[DEV] Merge test - payment features`

### 到 Main 分支
```
[RELEASE] v<version> - <description>
```
範例: `[RELEASE] v1.3.0 - Q4 feature release`

---

## Code Review 檢查清單

### 審查者檢查
- [ ] 代碼風格一致
- [ ] 有錯誤處理
- [ ] 包含測試
- [ ] 文檔已更新
- [ ] 無安全問題
- [ ] 性能合理

### 提交者檢查
- [ ] CI 全部通過
- [ ] 本地測試通過
- [ ] 代碼已自我審查
- [ ] Issue 已連結
- [ ] PR 描述完整

---

## 分支保護規則

| 分支 | 審查人數 | Status Checks | 備註 |
|------|---------|---------------|------|
| **main** | 1人 | 全部檢查 | 最嚴格 |
| **dev** | 1人 | 全部檢查 | 標準 |
| **test** | 1人 | 基本檢查 | 寬鬆 |

---

## 常用命令

### 創建分支
```bash
git checkout test && git pull && git checkout -b feat/description
```

### 同步最新代碼
```bash
git checkout feat/description
git pull origin test --rebase
```

### 推送分支
```bash
git push origin feat/description
```

### 查看所有分支
```bash
git branch -a
```

### 刪除本地分支
```bash
git branch -d feat/description
```

### 強制同步遠端分支 (小心使用!)
```bash
git push origin feat/description --force-with-lease
```

---

## 解決衝突

```bash
# 1. 更新目標分支
git checkout test
git pull origin test

# 2. Rebase 功能分支
git checkout feat/your-feature
git rebase test

# 3. 解決衝突 (編輯檔案)

# 4. 完成 rebase
git add .
git rebase --continue

# 5. 推送 (使用 force-with-lease 更安全)
git push origin feat/your-feature --force-with-lease
```

---

## 快速檢查（僅限最小化版本）

**注意**: 最小化版本不需要本地檢查，直接推送即可。CI 會自動檢查。

如果使用完整版本：
```bash
npm run lint          # 檢查程式碼風格
npm run test          # 執行測試
npm run build         # 確保可以建置
```

---

## CI 狀態說明

| 狀態 | 說明 | 處理方式 |
|------|------|---------|
| [通過] | 所有檢查通過 | 可以合併 |
| [失敗] | 檢查失敗 | 查看日誌並修復 |
| [進行中] | 檢查進行中 | 等待完成 |
| [跳過] | 檢查被跳過 | 確認是否正常 |

### 常見 CI 失敗原因
1. **分支命名錯誤** -> 重新建立正確命名的分支
2. **PR 格式錯誤** -> 修正 PR 標題
3. **Commit 格式問題** -> 修正 commit message

---

## 緊急情況處理

### Hotfix 流程 (生產環境緊急修復)
```bash
# 1. 從 main 創建 hotfix
git checkout main
git pull origin main
git checkout -b fix/critical-bug

# 2. 修復並測試
# ... 修復程式碼 ...
git commit -m "fix: critical security issue"

# 3. 直接 PR 到 main
# 標題: [HOTFIX] fix: critical security issue
# 需要 2 人快速審查

# 4. 合併後同步到其他分支
git checkout dev && git pull origin main && git push origin dev
git checkout test && git pull origin main && git push origin test
```

### 錯誤合併回退
```bash
# 使用 revert (推薦)
git revert <commit-hash>
git push origin <branch-name>

# 緊急情況聯繫管理員
```

---

## 最佳實踐

### 好習慣
- 每天開始前同步 test 分支
- 頻繁提交小的變更
- 及時推送避免衝突
- 仔細審查他人的 PR
- 寫清楚的 commit message
- 提交前本地測試（完整版）

### 壞習慣
- 直接推送到 main/dev/test
- 大量程式碼一次提交
- 不寫 commit message
- 不執行本地測試（完整版）
- 忽略 CI 失敗
- 不回應 review 意見

---

## 需要幫助?

### 相關文檔
- 完整文檔: `MINIMAL_SETUP.md` 或 `GITHUB_WORKFLOW_SETUP.md`
- 實作指南: `IMPLEMENTATION_GUIDE.md`
- GitHub Flow: https://docs.github.com/en/get-started/quickstart/github-flow

---

## 記住這些重點

1. **三分支策略**: main (穩定) <- dev (整合) <- test (測試)
2. **分支命名**: `type/description` (小寫 + 連字符)
3. **PR 標題**: `[TARGET] type: description`
4. **審查人數**: test(1) -> dev(1) -> main(2)
5. **合併策略**: test 用 Squash, dev/main 用 Merge commit
6. **測試優先**: 本地測試 -> CI 檢查 -> Code Review (完整版)
7. **及時溝通**: 遇到問題立即提問

---
