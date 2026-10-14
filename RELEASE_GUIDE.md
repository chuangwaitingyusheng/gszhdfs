# 发版操作手册

「格式转换大法师」新版本发布流程。

## 核心原则

- **本仓库只放文档，安装包一律作为 Release 附件上传。**
- `.gitignore` 已忽略 `*.exe`，从机制上防止把安装包提交进 git 历史。
- 每个版本的安装包约 70 MB，若提交进 git 会永久留在提交历史里，十几次发版后
  仓库会膨胀到 GB 级且无法瘦身；Release 附件不计入仓库体积，下载也更快。

## 一、准备安装包

在源码仓库（`F:\Projects\9. GoWeb\Code\WorkflowHub`）构建：

```powershell
cd F:\Projects\9. GoWeb\Code\WorkflowHub
npx tauri build
```

产物：
```
WorkflowHub\src-tauri\target\release\bundle\nsis\格式转换大法师_<版本>_x64-setup.exe
```

## 二、把安装包复制成 ASCII 文件名（重要）

GitHub 的 uploads 接口对 URL 里的中文支持不好，直接用中文名上传会把文件名截断成
`_0.1.0_x64-setup.exe` 这种残缺形式。

**上传前先复制成纯英文名：**

```powershell
Copy-Item "格式转换大法师_0.1.0_x64-setup.exe" "$env:TEMP\geshizhuanhuan_dafashi_0.1.0_x64-setup.exe"
```

## 三、更新 README 的更新日志

编辑本仓库 `README.md`，在「更新日志」下加一节：

```markdown
### 0.2.0

- 修复 xxx
- 新增 xxx
```

提交推送：

```powershell
cd F:\Projects\gszhdfs-release
git add README.md
git commit -m "docs: 更新 0.2.0 更新日志"
git push
```

## 四、创建 Release 并上传安装包

### 方式 A：网页操作（推荐，最简单）

1. 打开 https://github.com/chuangwaitingyusheng/gszhdfs/releases/new
2. **Choose a tag** 填 `v0.2.0` → 点 **Create new tag**
3. **Release title** 填 `格式转换大法师 v0.2.0`
4. **Describe this release** 填更新说明
5. 把 ASCII 改名的 exe **拖到**「Attach binaries by dropping them here」区域
6. 点 **Publish release**

### 方式 B：命令行（用 PowerShell 调 GitHub API）

先取已保存的令牌：

```powershell
$out = "protocol=https`nhost=github.com`n`n" | git credential fill 2>&1
$token = ($out | Select-String '^password=').Line -replace '^password=',''
$headers = @{ Authorization = "token $token"; Accept = "application/vnd.github+json"; "User-Agent" = "gszhdfs-release" }
```

创建 Release：

```powershell
$body = @{
  tag_name = "v0.2.0"
  name     = "格式转换大法师 v0.2.0"
  body     = "## 更新内容`n`n- 修复 xxx"
} | ConvertTo-Json

$rel = Invoke-RestMethod -Method Post `
  -Uri "https://api.github.com/repos/chuangwaitingyusheng/gszhdfs/releases" `
  -Headers $headers -Body $body -ContentType "application/json; charset=utf-8" -TimeoutSec 60
$rel.id    # 记下这个 id
```

上传附件（**建议放后台跑，69 MB 视网速需几分钟**）：

```powershell
$relId = $rel.id
$exe = "$env:TEMP\geshizhuanhuan_dafashi_0.2.0_x64-setup.exe"
$uri = "https://uploads.github.com/repos/chuangwaitingyusheng/gszhdfs/releases/$relId/assets?name=$(Split-Path $exe -Leaf)"

Invoke-RestMethod -Method Post -Uri $uri -Headers $headers -InFile $exe `
  -ContentType "application/octet-stream" -TimeoutSec 3600
```

> **注意**：上传耗时较长，PowerShell 的 `Invoke-RestMethod` 如果超时会报
> `操作超时`，但此时附件可能已经上传完成。**先查再重传**，避免重复：
> ```powershell
> Invoke-RestMethod "https://api.github.com/repos/chuangwaitingyusheng/gszhdfs/releases/$relId/assets" -Headers $headers
> ```

## 五、验证发布结果

```powershell
# 看附件列表与状态（state 应为 uploaded）
Invoke-RestMethod "https://api.github.com/repos/chuangwaitingyusheng/gszhdfs/releases/tags/v0.2.0" -Headers $headers |
  Select-Object -ExpandProperty assets | Select-Object name, size, state
```

再打开 Release 页面确认能下载：
https://github.com/chuangwaitingyusheng/gszhdfs/releases

## 常见问题

| 问题 | 原因与处理 |
|---|---|
| 附件名变成 `_0.1.0_x64-setup.exe` | 用了中文文件名上传。删掉附件改用 ASCII 名重传 |
| 上传报「操作超时」 | 网络慢，但附件可能已传完。先查附件列表再决定是否重传 |
| 想删掉传错的附件 | `Invoke-RestMethod -Method Delete "https://api.github.com/repos/.../releases/assets/<asset_id>" -Headers $headers` |
| 误把 exe 提交进 git | 立刻 `git reset --soft HEAD~1` 撤回提交，**不要 push** |
| 国内下载慢 | GitHub 直链在国内较慢，可考虑同时传到腾讯云 COS / 对象存储，配合官网分发 |

## 仓库隔离说明

- **发版目录**：`F:\Projects\gszhdfs-release` —— 只含 README.md 和 .gitignore
- **源码目录**：`F:\Projects\9. GoWeb\Code` —— 另一个独立 git 仓库，连的是私有 Gitea，**不要**往里加 GitHub remote

两个目录物理隔离，所以在发版目录里执行任何 git 命令都不可能碰到源码。
