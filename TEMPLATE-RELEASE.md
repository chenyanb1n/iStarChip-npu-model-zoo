# 模板手动打包发布手册

> 适用：iStarChip-npu-model-zoo 仓库。向 NPU-IDE 打包版用户交付模板工程
> （音频/图像）的完整手动流程。客户端（`electron/main.js` 的 `ensureTemplateRoot`）
> 只认本仓根的 `templates.json` + Release assets，不需改 IDE 代码。

## 0. 命名铁律（先看，错了客户端认不出）

| 项 | 规则 | 例 |
|---|---|---|
| manifest 条目 key | 必须等于 IDE 侧 `TEMPLATE_REGISTRY` 里的模板 id，否则报 `unknown template` | `babycry`、`sirendetection` |
| Release tag | `<id>-vN`（模板名前缀即命名空间，各模板版本各自演进） | `babycry-v1`、`sirendetection-v1` |
| asset 文件名 | 与 tag 同名：`<id>-vN.zip` | `sirendetection-v1.zip` |
| manifest `version` | 不透明字符串，客户端只做相等比较；推荐从 `"1"` 起按整数递增 | `"1"`、`"2"` |
| manifest `file` | asset 完整地址：`https://github.com/chenyanb1n/iStarChip-npu-model-zoo/releases/download/<tag>/<asset>` | 见 §3 |

硬限：单个文件**进不了 git 树**（100MB 硬限）→ zip 只能走 Release asset（上限 2GB）；
`templates.json`/`README.md`/本手册是小文件，走 git 树。

## 1. 打 zip（仓库外操作）

源目录 = NPU-IDE 主仓对应模板目录（如 `projects_audio/sirendetection`），整目录全量，
什么都不删（生成物才占几 MB，瘦身不值；babycry/siren 两批实测都是 data 占 99%+）。

要求：**解压后根即模板根**（`data/`、`.npupproj` 在顶层，不要多包一层；
万一多了一层客户端有拆包兜底，但首选无 wrapper）。

```powershell
# 进源目录后对内容打 zip（无 wrapper 层）；TAG 按 §0 规则定
$src = 'D:\0_WorkingProject\3_NPU\5_projects\npu-ide\projects_audio\sirendetection'
$zip = "$env:TEMP\sirendetection-v1.zip"
Compress-Archive -LiteralPath (Join-Path $src '*') -DestinationPath $zip -Force
```

有隐藏文件（点开头）时 `*` 会漏——先 `Get-ChildItem -Force` 看一眼，有就显式加进去。

## 2. 算哈希/体积 + 验包（必做，防坏包发版）

```powershell
(Get-FileHash $zip -Algorithm SHA256).Hash.ToLower()
(Get-Item $zip).Length
# 解压验包（只解到 TEMP，不进任何仓库）：
$d = Join-Path $env:TEMP 'zipcheck'
Expand-Archive -LiteralPath $zip -DestinationPath $d -Force
@(Get-ChildItem -LiteralPath $d -Recurse -File).Count          # 总文件数，记下来
Test-Path -LiteralPath (Join-Path $d '<模板>.npupproj')        # 必须 True
@(Get-ChildItem -LiteralPath (Join-Path $d 'data') -Recurse -File).Count  # data 数，记下来
Remove-Item -LiteralPath $d -Recurse -Force                    # 验完即删
```

## 3. 改 `templates.json`（根目录，网页编辑或上传覆盖）

保留已有条目不动，追加新条目（`size`/`sha256` 填 §2 实测值）：

```json
{
  "templates": {
    "babycry": { "...": "原样保留" },
    "sirendetection": {
      "file": "https://github.com/chenyanb1n/iStarChip-npu-model-zoo/releases/download/sirendetection-v1/sirendetection-v1.zip",
      "version": "1",
      "size": 555682980,
      "sha256": "73a5fd091d5c9fec0ffd51ddf4b205c35785b0b5d7b5f10198ae87a1636eb538"
    }
  }
}
```

## 4. 改 `README.md`（根目录）

版本对照表加一行（`| <tag> | <模板id> | <version> |`），模板来源节加一条
（快照自哪个目录、日期、zip 内容、data 文件数——照 §2 记的数字写）。

## 5. 发 Release（网页）

1. 仓库主页 → Releases → Draft a new release。
2. Choose a tag 输入 `<id>-vN` → Create new tag on publish；Target `main`；Title 同 tag。
3. Attach binaries 把 §1 的 zip 拖进去（几百 MB，传几分钟，页面别关，看到文件名+体积才算完）。
4. Publish release。

## 6. 验证（发完必做）

1. 浏览器打开 `https://raw.githubusercontent.com/chenyanb1n/iStarChip-npu-model-zoo/main/templates.json`
   —— 200 且新条目在。
2. 浏览器打开 manifest 里新条目的 `file` 地址 —— 跳转后开始下载即通。
3. （可选但推荐）找一台装了打包版 NPU-IDE 的机器：新建工程选该模板，
   走完一次冷下载（看进度条），核对数据管理数量 = §2 记的 data 数；
   断网重进应命中缓存 + 黄字提示。

## 7. 升级模板（新版本）

1. 新快照按 §1–§2 重打 zip（新 tag，如 `sirendetection-v2`）→ 新 Release 传 asset。
2. `templates.json` 改该模板三行（`file`/`version`/`size`/`sha256`）→ push。
3. 客户端以 `version` 比对本地 `.version`，不一致自动重下。旧 Release 保留可回滚；
   回滚 = 把 manifest 改回旧三行再 push（客户端下次比对即回退）。

## 8. 排障

| 现象 | 查哪里 |
|---|---|
| 新建模板报 `unknown template: xxx` | manifest key 与 IDE `TEMPLATE_REGISTRY` id 对不上，按 §0 改 key |
| `模板清单获取失败` | raw 地址打不开：分支不是 `main` / 文件名大小写 / 刚 push 等几十秒 CDN |
| `模板校验失败（sha256 不匹配）` | 传错包或 manifest 填错：重算 §2、重传或改 manifest |
| `下载失败：download -> HTTP 404` | tag/asset 名与 manifest `file` 对不上，或 Release 还在草稿没 Publish |
| 离线/慢网 | 正常：有旧缓存用旧缓存 + 黄字提示；有缓存的机器不受影响 |

## 9. 把下载指向其他服务器（长期容量方案）

> GitHub 只适合起步：单文件 2GB 上限、Release 存几 GB 就显重，
> 模板一多（6 个已 ~3.4GB）必然搬家。好消息：**搬家只改 manifest，
> 不改客户端、不发版**——当初 `file` 就是按绝对 URL 设计的。

### 9.1 新服务器要满足什么（静态文件服务即可，无需写接口）

- HTTPS GET 能拿到文件；HTTP 301/302 跳转 ≤3 次（客户端跟 3 跳，对象存储的
  签名 URL 跳转一般 1~2 次，够用）；响应带 `Content-Length`（进度条用）。
- URL 稳定（一旦写入 manifest 就别换）；建议带版本号的路径，
  如 `https://dl.example.com/npu-templates/sirendetection-v1.zip`。
- 备选形态都行：Nginx/Apache 目录、OSS/COS/S3 + CDN、自建 MinIO、NAS 前挂 HTTPS。
- `templates.json` 本身（几百字节）可以**永远留在 GitHub**——只有 zip 要搬；
  双域混用是设计内行为（babycry 首发就是 raw + releases 双域）。

### 9.2 搬家步骤（零停机，用户无感）

1. 把各模板 zip **原样**传到新服务器（文件名建议不变；字节不动，
   `size`/`sha256` 就不用改）。
2. 先验证：浏览器/命令行 GET 新地址能下，且字节数 = manifest 里的 `size`。
3. 改 `templates.json`：只换各条目的 `file` URL，其余不动 → push。
   `version` 没变 → 用户本地缓存继续有效，**零重下**。
4. 观察一轮（新建模板工程成功）后再决定旧 Release assets 删不删；
   回滚 = 把 `file` 改回旧地址再 push，一分钟生效。

### 9.3 搬家前先试点（不建议一次全切）

- 先搬一个模板（如 siren），babycry 留 GitHub：manifest 天然支持逐条目混布，
  哪个条目稳了再搬下一个。
- 试点前可用环境变量让某台机器先行：
  `NPU_TEMPLATE_SERVER` 覆盖只影响 manifest 地址；要试新 zip 地址，
  直接改一份本地 manifest 指向它测一遍 §6 即可。

### 9.4 什么时候必须动客户端

- 只有 `templates.json` 本身也要搬家时，才需要改 `TEMPLATE_SERVER_BASE`
  默认值（`electron/main.js` 一行）并跟 App 发版——而它才几百字节，
  长期放 GitHub 完全没问题，所以这一天大概率不来。
