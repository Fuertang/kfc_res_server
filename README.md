# 厨侠微信热更资源发布

客户端读取 `https://fuertang.github.io/kfc_res_server/kungfucook/wechat/latest.json`，再从 `kungfucook/wechat/releases/<版本>/` 下载四个 PCK 和 `chunks.json`。仓库根目录的 `hotupdate/` 是 Godot 打包器的本地输出目录，已在 `.gitignore` 中排除；不要强制添加它。`build_report.json` 和 `res_catalog.tres` 用于本地构建核验，不作为 Pages 下载文件。

## 每次资源更新的顺序

1. 如果任何 PCK 内容改变，先在 Godot 的 `wechat_hotupdate.tres` 方案中递增 `release_version`。不要覆盖已经发布的版本目录或以相同版本替换 PCK。只改首包脚本或配置且四包摘要未变时，不必重新发布热更资源。
2. 在 Godot 中用“项目 → 工具 → PCK 打包”选择“微信美术音频热更”，执行**完整构建**。当前方案输出到仓库根目录的 `hotupdate/wechat/<版本>/`，生成四个 PCK、`chunks.json`、`manifest.json`、`res_catalog.tres` 和构建报告。完整构建还会同步游戏工程中的资源归属表和首包内嵌清单。
3. 在资源仓库中先同步远端，再将四个 PCK、`chunks.json` 和 `manifest.json` 复制到 `kungfucook/wechat/releases/<版本>/`。核对文件大小、完整 SHA-256 和清单版本。只暂存该版本目录，提交并推送。
4. 等 GitHub Pages 上该版本的四个 PCK、`chunks.json` 与 `manifest.json` 全部可访问，再把**同一份** `manifest.json` 复制为 `kungfucook/wechat/latest.json`，单独提交并推送。修改 `latest.json` 会让已发布的旧小游戏客户端看到新版本，须先确认兼容性和发布时间。
5. 导出微信小游戏首包，核对整个上传目录小于 20,000,000 字节。在微信开发者工具验证版本检查、下载、断点续下、四包挂载、开始页视频和 Gameplay，然后发布小游戏。

## PowerShell 操作示例

在新的版本号已经设置并且 Godot 完整构建成功后执行。将示例版本改为实际版本：

```powershell
$repo = 'G:\godotProject\KungfuCook\ResServer'
$ver = '1.0.1'
$source = Join-Path $repo "hotupdate/wechat/$ver"
$release = Join-Path $repo "kungfucook/wechat/releases/$ver"
$files = @('common.pck', 'home.pck', 'home_video.pck', 'battle.pck', 'chunks.json', 'manifest.json')

git -C $repo status -sb
if ((git -C $repo status --porcelain).Length -ne 0) { throw '资源仓库有未处理改动，先检查后再发布' }
git -C $repo pull --ff-only origin master
if ($LASTEXITCODE -ne 0) { throw '远端同步失败，停止发布' }
if (Test-Path -LiteralPath $release) { throw "版本目录已存在，禁止覆盖：$release" }
foreach ($name in $files) {
    if (-not (Test-Path -LiteralPath (Join-Path $source $name))) { throw "缺少构建产物：$name" }
}
New-Item -ItemType Directory -Path $release | Out-Null
foreach ($name in $files) {
    $from = Join-Path $source $name
    Copy-Item -LiteralPath $from -Destination (Join-Path $release $name)
}

$manifest = Get-Content -LiteralPath (Join-Path $release 'manifest.json') -Raw | ConvertFrom-Json
if ($manifest.release_version -ne $ver) { throw 'manifest 版本不匹配' }
foreach ($pack in $manifest.packs) {
    $path = Join-Path $release $pack.file
    if ((Get-Item -LiteralPath $path).Length -ne $pack.size_bytes) { throw "大小不符：$path" }
    if ((Get-FileHash -LiteralPath $path -Algorithm SHA256).Hash.ToLowerInvariant() -ne $pack.sha256.ToLowerInvariant()) { throw "摘要不符：$path" }
}
$chunks = Get-Content -LiteralPath (Join-Path $release 'chunks.json') -Raw | ConvertFrom-Json
if ($chunks.release_version -ne $ver -or $chunks.chunk_size -ne 2097152) { throw '分块清单不匹配' }
foreach ($pack in $manifest.packs) {
    $part = $chunks.packs.($pack.pack_id)
    if ($null -eq $part -or $part.sha256 -ne $pack.sha256 -or $part.size_bytes -ne $pack.size_bytes) { throw "分块清单的包信息不匹配：$($pack.pack_id)" }
}

git -C $repo add -- "kungfucook/wechat/releases/$ver"
git -C $repo diff --cached --stat
git -C $repo commit -m "发布厨侠微信热更资源 $ver"
git -C $repo push origin master
```

等待 Pages 部署，确认六个版本文件都可访问（下列检查均应返回 `200`），且客户端兼容后，再发布版本入口：

```powershell
foreach ($name in $files) {
    $url = "https://fuertang.github.io/kfc_res_server/kungfucook/wechat/releases/$ver/$name"
    $response = Invoke-WebRequest -Uri $url -Method Head
    if ($response.StatusCode -ne 200) { throw "Pages 文件不可用：$url" }
}
```

发布版本入口：

```powershell
Copy-Item -LiteralPath (Join-Path $release 'manifest.json') -Destination (Join-Path $repo 'kungfucook/wechat/latest.json') -Force
git -C $repo add -- kungfucook/wechat/latest.json
git -C $repo diff --cached -- kungfucook/wechat/latest.json
git -C $repo commit -m "启用厨侠微信热更版本 $ver"
git -C $repo push origin master
```

如果 `pull --ff-only` 报分叉，先停止操作并检查本地提交；不要强制推送，也不要直接用新的版本覆盖已有目录。
