# Syncthing Fork - Global .stignore Support

此專案是 Syncthing v2.0.10 的 fork,新增全域 `.stignore` 支援。

## 專案資訊

- **原始版本**: v2.0.10 (官方穩定版)
- **Fork 版本**: v2.0.10+global1
- **修改日期**: 2025-11-04
- **Fork 來源**: https://github.com/syncthing/syncthing
- **主要功能**: 支援全域 `.stignore` 檔案

## 新增功能: Global .stignore

### 1. 使用方式

**設定檔位置**: `~/.config/syncthing/.stignore`

**建立全域設定檔**:
```bash
mkdir -p ~/.config/syncthing
nano ~/.config/syncthing/.stignore
```

**範例內容**:
```
// macOS 系統檔案
(?d).DS_Store
(?d).AppleDouble

// Node.js
node_modules
dist

// Python
__pycache__
.venv
venv

// 敏感資料
.env
.env.*
!.env.example

// 憑證
*.key
*.pem
credentials.json

// IDE
.vscode
.idea

// Git
.git
```

### 2. 規則載入機制

**載入順序**: Global → Project Local

**規則合併**:
- Global patterns 優先載入
- Project .stignore 可補充或覆蓋
- 兩者的規則會合併使用

**變更偵測**:
- 每個 folder 有獨立的 ChangeDetector
- 修改 global .stignore 後,**不會自動套用**
- 需要手動觸發重新載入

**讓變更生效的方法**:

1. **手動 Rescan** (推薦):
   - 在 Web UI (http://localhost:8384) 點擊資料夾 → **Rescan**
   - 只需要 Rescan 任一個 folder,該 folder 立即生效
   - 其他 folder 在下次 Rescan 時也會獲得最新規則

2. **重啟服務**:
   ```bash
   brew services restart syncthing
   ```

3. **API 觸發**:
   ```bash
   curl -X POST -H "X-API-Key: YOUR_API_KEY" \
     http://localhost:8384/rest/db/scan?folder=FOLDER_ID
   ```

### 3. 技術實現

**修改檔案**: `lib/ignore/ignore.go`

**核心邏輯**:
- 在 `parseLocked()` 函數中載入 global .stignore
- 為每個 folder 創建獨立的臨時 ChangeDetector
- 避免跨 folder 的 "multiple include" 錯誤
- 確保每次 Rescan 都能獲得最新的 global patterns

**關鍵函數**:
```go
// 取得 global .stignore 路徑
func getGlobalIgnorePath() string

// 建立 filesystem 物件
func newGlobalFilesystem(path string) fs.Filesystem

// 合併 global 和 project patterns
func (m *Matcher) parseLocked(r io.Reader, file string) error
```

---

## 編譯與部署

### 1. 版本號管理

**Tag 命名規則**: `v2.0.10+global1`
- 基於官方 v2.0.10
- `+global1` 表示自訂功能的迭代版本
- 未來更新使用 `+global2`, `+global3` 等

**建立 Annotated Tag** (必須):
```bash
# Syncthing 的 git describe 只認 annotated tag
git tag -a v2.0.10+global1 -m "Fork v2.0.10 with global .stignore support"
```

**驗證版本**:
```bash
git describe --always --dirty --abbrev=8
# 輸出: v2.0.10+global1 (正確)
```

### 2. 編譯步驟

**環境需求**:
- Go 1.25.3+ (darwin/arm64)
- 安裝: `arch -arm64 brew install go`

**編譯指令**:
```bash
cd /path/to/syncthing

# 清理並編譯
rm -f syncthing
go run build.go build

# 確認版本
./syncthing --version
# 輸出: syncthing v2.0.10+global1 "Hafnium Hornet" ...
```

**跨平台編譯**:
```bash
# 編譯 Intel Mac 版本
GOARCH=amd64 go run build.go build

# 編譯 Universal Binary (同時支援 Intel 和 Apple Silicon)
GOARCH=arm64 go run build.go build
mv syncthing syncthing-arm64

GOARCH=amd64 go run build.go build
mv syncthing syncthing-amd64

lipo -create -output syncthing syncthing-arm64 syncthing-amd64
```

### 3. 部署到 macOS

#### Homebrew 目錄結構
```
/opt/homebrew/
├── Cellar/syncthing/2.0.10/bin/syncthing  # 實體檔案 ← 替換這個
├── opt/syncthing → ../Cellar/...          # 目錄 symlink
└── bin/syncthing → ../opt/...             # 檔案 symlink
```

#### 本機部署

```bash
# 1. 停止服務
brew services stop syncthing

# 2. 備份原版本
sudo cp /opt/homebrew/Cellar/syncthing/2.0.10/bin/syncthing \
        /opt/homebrew/Cellar/syncthing/2.0.10/bin/syncthing.bak

# 3. 替換執行檔 (只需替換 Cellar 中的實體檔案)
sudo cp ./syncthing /opt/homebrew/Cellar/syncthing/2.0.10/bin/syncthing

# 4. 重啟服務
brew services start syncthing

# 5. 驗證版本
/opt/homebrew/bin/syncthing --version
```

#### 遠端部署 (scp)

**前提**: 目標機器也是 arm64 架構

```bash
# 1. 複製到遠端 (在本機執行)
scp ./syncthing user@remote-mac:/tmp/syncthing

# 2. 在遠端機器執行
ssh user@remote-mac

# 3. 停止服務
brew services stop syncthing

# 4. 替換並重新簽名 (重要!)
sudo cp /tmp/syncthing /opt/homebrew/Cellar/syncthing/2.0.10/bin/syncthing
sudo codesign --remove-signature /opt/homebrew/Cellar/syncthing/2.0.10/bin/syncthing
sudo codesign -s - /opt/homebrew/Cellar/syncthing/2.0.10/bin/syncthing

# 5. 重啟服務
brew services start syncthing

# 6. 驗證
/opt/homebrew/bin/syncthing --version
```

**重要**: macOS 的代碼簽名保護機制會拒絕未簽名或簽名被污染的執行檔 (顯示 `Killed: 9`)。
必須在目標機器上重新簽名: `sudo codesign -s -`

### 4. 重要注意事項

#### 代碼簽名問題

**錯誤現象**: 執行檔顯示 `Killed: 9`

**原因**: macOS 的代碼簽名保護,拒絕執行未正確簽名的二進制檔案

**解決方案**:
```bash
# 移除舊簽名並重新簽名
sudo codesign --remove-signature /path/to/syncthing
sudo codesign -s - /path/to/syncthing

# 驗證簽名
codesign -dvv /path/to/syncthing
```

#### Homebrew 升級影響

**問題**: `brew upgrade syncthing` 會覆蓋自訂版本

**解決方案**:

1. **鎖定版本** (推薦):
   ```bash
   brew pin syncthing
   ```

2. **升級後重新部署**:
   ```bash
   # 如果不小心升級了,重新編譯並部署
   cd /path/to/syncthing-fork
   go run build.go build
   sudo cp ./syncthing /opt/homebrew/Cellar/syncthing/NEW_VERSION/bin/syncthing
   brew services restart syncthing
   ```

#### 架構相容性

**檢查遠端機器架構**:
```bash
ssh user@remote 'uname -m'
# arm64 = Apple Silicon (M1/M2/M3)
# x86_64 = Intel
```

**不同架構需要重新編譯**:
- 不能將 arm64 的執行檔用在 Intel Mac 上
- 必須使用 `GOARCH=amd64` 重新編譯
- 或編譯 Universal Binary 同時支援兩種架構

---

## 測試驗證

### 功能測試

1. **確認 global .stignore 存在**:
   ```bash
   ls -la ~/.config/syncthing/.stignore
   ```

2. **測試忽略規則**:
   ```bash
   # 在同步資料夾建立應被忽略的檔案
   cd /path/to/synced/folder
   mkdir node_modules
   echo "test" > node_modules/test.js

   # Rescan folder
   # 在 Web UI 點擊 folder → Rescan

   # 確認未同步到遠端機器
   ```

3. **查看日誌** (macOS):
   ```bash
   tail -f /opt/homebrew/var/log/syncthing.log
   ```

### 測試結果

**測試日期**: 2025-11-04

**測試環境**:
- MacBook Air (arm64)
- Mac mini (arm64)
- 兩台都運行 v2.0.10+global1

**驗證項目**: ✅
- Global .stignore 正確載入
- 忽略規則生效 (node_modules, .DS_Store 等未同步)
- Rescan 後變更立即生效
- 向後相容 (沒有 global .stignore 的 folder 正常運作)

---

## Git 記錄

**Commit**: d8fe41d02
```
feat: add global .stignore support

- Load ~/.config/syncthing/.stignore before project .stignore
- Global rules are applied first, project rules can override
- Silently ignore if global .stignore does not exist
- Maintains backward compatibility with existing projects
- Fix: Load global patterns in parseLocked() for correct merging
- Fix: Use independent ChangeDetector to avoid cross-folder conflicts
```

**Tag**: v2.0.10+global1

---

## 參考資料

- [Syncthing 官方文件](https://docs.syncthing.net/)
- [.stignore 語法](https://docs.syncthing.net/users/ignoring.html)
- [Syncthing GitHub](https://github.com/syncthing/syncthing)
- [Semantic Versioning](https://semver.org/)

---

## 相關檔案

- `~/.config/syncthing/.stignore` - 全域忽略規則模板
- `lib/ignore/ignore.go` - 核心修改檔案
- `docs/syncthing-global-stignore-modification.md` - 詳細技術文件
- `docs/syncthing-setup.md` - 基本設定指南
