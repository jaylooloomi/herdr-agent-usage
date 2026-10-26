# Windows 原生移植工作單

狀態：規劃中，尚未支援 Windows。不可只修改 manifest 的 platforms 後宣稱移植完成。

整合發行包與安裝規格的唯一來源：
https://github.com/jaylooloomi/herdr/blob/integration/windows-agent-bundle/docs/windows-agent-bundle.zh-TW.md

本分支：`integration/windows-support`。上游基準：`511622e976a6c666a429809dd0db7a7b8e6e59d5`。

## 已確認的移植範圍

- `herdr-plugin.toml` 僅列 macOS/Linux，build、startup、events、actions 和 pane 全部依賴 Bash。
- `cmd/usagebar/main.go` 使用 `SIGWINCH`、`SysProcAttr.Setsid`；必須拆成 OS 實作。
- `internal/update/watch.go` 使用 `unix.Flock`、`Futimes`；需 Windows 鎖定與時間戳實作。
- `internal/setup` 產生 Bash statusLine 指令；Windows 需要原生可執行檔或 PowerShell argv。
- 現有測試大量使用 `/bin/sh` 假程式；新增 Windows fixture，保留 Unix 測試。
- Go 最低版本以 `go.mod` 為準，目前是 1.26.0，不能沿用舊 README 的 1.25 說法。

## 實作順序

1. 先讓 `GOOS=windows GOARCH=amd64 go build ./cmd/usagebar` 通過，再加入 Windows CI。
2. OS-specific helper 使用 build tags；provider-neutral 層不分支判斷 Claude/Codex/Agy。
3. 以 registry 為 provider 清單唯一來源；安裝選擇三個 provider 不等於把共享架構寫死成三個。
4. 將 Bash wrapper 的環境與 cwd 契約移到 Go entry point；必要時提供 Windows-only manifest commands。
5. 處理 `.exe`、`.cmd`、含空白/中文路徑、Windows Credential Manager 與用戶目錄差異。
6. setup 保留已有 statusLine，新增 adapter，支援移除並復原；不得破壞 Herdr 官方 hooks。
7. 原生 Windows 驗證 Claude、Codex、Agy 的真實 session matching、quota、context、cache。
8. 通過後才把 Windows 加入 platforms，發佈帶 SHA-256 的預編譯 zip，再供整合包鎖定。

## 驗收

- `gofmt -l .` 無輸出；`go vet ./...`、`go test ./...`、`go build ./...` 通過。
- Linux/macOS/Windows CI，以及既有 lint/security checks 通過。
- Windows 原生執行，不依賴 WSL、Bash、MSYS2。
- 缺登入、過期、網路錯誤、未知 quota 分別顯示；不可把缺資料轉成 0% 或 100%。
- Agy 需收到真實 statusLine payload；CLI 已安裝不代表已取得額度。
- 每個 provider 的原始額度與面板數值吻合；不得把不同帳號或 session 的數據混用。
- 重複 setup、升級、解除安裝不重複 hook/row，不遺失使用者設定。

尚未開始程式移植；這個分支目前提供可交接的工作單。
