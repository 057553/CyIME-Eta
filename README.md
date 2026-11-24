# CyIME Eta

自用的 CyIME 构建配置。**不含源码**——构建时 checkout 上游 [Cyletix/CyIME](https://github.com/Cyletix/CyIME)，再套本仓库的补丁。

产物与官方包共存：包名 `com.cyletix.cyime.eta`，应用名 **CyIME Eta**。

---

## 这个仓库做什么

| 补丁 | 作用 | 必需 |
|---|---|---|
| `pr575-stroke-filter-106.patch` | 候选栏笔画筛选：拼音出候选后从空格键起笔，按笔形过滤当前候选 | ✅ |
| `fuzzylock.patch` | 锁定读音时保留同段近音候选（修「开模糊音、点读音后金消失」） | ✅ |
| `coexist-package.patch` | `applicationIdSuffix = ".eta"`，与官方包共存 | ✅ |
| `slim-manifest-sync.patch` | 词库清单与精简资产同步（不同步会导致安装后准备词库死循环） | ✅ |
| `eta-brand.patch` | 应用名改 CyIME Eta | |
| `eta-icon.patch` | 换成 Eta 版图标（浅紫底 + 环 + η） | |
| `slim-assets.patch` | 排掉五笔/仓颉/速成/自然码/KaTeX，约 −20 MB | |
| `skip-install-integrity.patch` | 自签包跳过签名自检，避免每次进设置弹警告 | |
| `skip-offline-translate.patch` | 上游离线翻译库资产 404，清空 sha 表 | |
| `arm64-only.patch` | 只构建 arm64-v8a | |

**注意**：`ChaiPUA-0.2.7-snow.ttf` 不能排进精简列表。它是按键标签字体的默认回退，缺了键盘首次渲染就崩、键盘弹不出来。

---

## 两个「笔画」别搞混

- **本仓库要的**：候选栏笔画筛选（`CandidateStrokeFilter.kt`）。拼音打完后从空格起笔过滤**当前候选**。
- **上游 1.15.106 加的**：独立的笔画输入键盘（`StrokeKeyboardLayout` / `StrokePreedit` / `StrokeGlyphInline` / `StrokeCellDictionary`），用 `hspnz` 一个字一个字打，不碰拼音候选。

两者互补，上游没有候选筛选功能。

---

## 同步上游更新

本仓库是独立仓库（不是 fork），上游同步用 remote，不依赖 GitHub 的 fork 机制。

### 查上游有什么新的

```bash
git remote add upstream https://github.com/Cyletix/CyIME.git   # 只需一次
git fetch upstream --tags
git log --oneline HEAD..upstream/main          # 上游新增的 commit
git log --oneline HEAD..upstream/main --grep='^feat'   # 只看新功能
```

或直接看 GitHub：`https://github.com/Cyletix/CyIME/compare/1.15.106...main`

### 升级版本

1. 把 `.github/workflows/cyime-eta.yml` 里的 `PINNED_UPSTREAM` 改成新版本号
2. **逐个验证补丁**（关键步骤，上游常改到冲突文件）：

```bash
V=1.15.107
curl -sL https://codeload.github.com/Cyletix/CyIME/tar.gz/refs/tags/$V | tar xz
cd CyIME-$V && git init -q . && git add -A && git commit -qm base

for p in pr575-stroke-filter-106 coexist-package fuzzylock slim-assets \
         slim-manifest-sync eta-brand eta-icon skip-install-integrity \
         skip-offline-translate arm64-only; do
  printf '%-30s ' "$p"
  git apply -p1 <path/to/$p.patch 2>/dev/null && echo OK || echo FAIL
done
```

3. 失败的补丁按下面「重新生成笔画筛选补丁」处理，其它补丁通常只需调 hunk 行号
4. 全部通过后再推 workflow 改动，触发构建

### 重新生成笔画筛选补丁

上游改到 `CandidateBar.kt` / `CandidatePage.kt` / `KeyboardView.kt` / `SpaceKeyButton.kt` 时，这个补丁会失效。重新生成：

```bash
# 1. 取 PR #575 最新 diff，按 hunk 过滤只留 stroke 相关的
#    （不能整体套用：PR 捆了 75 个文件、剪贴板收藏/底部留白/符号编辑等无关改动）
python3 extract_stroke_only.py

# 2. 在旧版本上应用子补丁，得到「旧版+笔画」
# 3. 三方合并到新版本
git merge-file -p <新版文件> <旧版本文件> <旧版+笔画文件> > merged.kt

# 4. 逐块检查 diff，确认没有引入无关改动
diff -u <新版文件> merged.kt
```

三个已知坑（前两次都踩过，务必检查）：

1. **三方合并会丢上游新增行**：曾经丢掉 `BottomLiftSpacer(...)`，合并后必须核对上游这行的出现次数
2. **hunk 边界会带进无关改动**：曾有 `directEnglishInput` 进入调用处但函数签名没有该参数，导致编译失败
3. **冲突处要保留上游的新逻辑**：`barAssociationWords` 用上游的 `cs.showsTypedEnglishWord` 判断，值换成笔画过滤后的 `visibleAssociations`

无法本地编译时的最低验证：干净树上按 workflow 顺序累积 `git apply` 全部补丁 + 符号 grep 核对 + 逐块 diff 审查。

---

## 签名

- keystore：`eta.jks`，alias `eta`
- 证书 SHA-256：`6D:A1:9C:63:A8:97:31:17:39:BE:03:59:EE:1D:FD:07:3C:41:B3:FD:1C:3C:CE:70:C0:49:44:A1:01:D0:50:C1`
- 密钥与密码存在本仓库 Actions secrets（`SIGNING_KEY` / `KEY_STORE_PASSWORD` / `KEY_PASSWORD` / `ALIAS`）
- **换 keystore 后必须卸载重装**，不能覆盖安装

---

## 手动构建

```bash
gh workflow run cyime-eta.yml -f force=true
```

`upstream_ref` 留空 = 用 `PINNED_UPSTREAM` 锁定的版本；填其它版本会打 warning 提示补丁可能失效。`force=true` 才会在 release 已存在时重建。
