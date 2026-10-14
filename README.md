# iStarChip NPU 模型模板仓库

NPU-IDE 打包版按需下载模板工程的公开托管仓库。客户端（`electron/main.js` 的 `ensureTemplateRoot`）先拉取根目录的 `templates.json` 清单，再从 Release 下载对应 zip，校验 sha256 后解压使用。

## 模板来源

- `babycry`（婴儿哭声检测）：快照自 NPU-IDE 主仓 `projects_audio/babycry`，快照日期 2026-10-14，整目录打 zip（含 `data/`、`configs/`、`models/`、`output/`、`babycry.npupproj`），解压后根即模板根（无 wrapper 层）。
- `sirendetection`：快照自 NPU-IDE 主仓 `projects_audio/sirendetection`，快照日期 2026-10-14，整目录打 zip（含 `data/`（1259 files）、`configs/`、`models/`、`output/`、`siren.npupproj`），解压后根即模板根（无 wrapper 层）。

## 版本对照（tag ↔ templates.json version）

| Release tag | 模板 | version |
|---|---|---|
| `babycry-v1` | babycry | `"1"` |
| `sirendetection-v1` | sirendetection | `"1"` |

`version` 为不透明字符串，客户端只做相等比较；tag 与 version 靠本表人工对齐。

## 升级步骤

1. 在仓库外对新快照打 zip（解压后根即模板根），算字节数与 sha256。
2. 发新 Release（如 tag `babycry-v2`），zip 作 binary asset 上传。
3. 改 `templates.json` 对应模板的三行（`file` / `version` / `size` / `sha256`），push 到 `main`。旧 Release 保留，可回滚。

## 仓库布局

根目录只放索引与说明（`templates.json` + `README.md`）；模板按条目扩展，不建子目录（zip 进不了 git 树且只能居 Release assets，目录结构管不着它；每个模板用独立 tag（如 `kws-v1`）+ 独立 asset，版本各自演进）。

