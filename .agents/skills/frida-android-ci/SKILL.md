---
name: frida-android-ci
description: GitHub Actions 只构建 Frida 的 Android gadget/server，按 ABI 可选，打 tag 自动发 Release。
---

# Frida Android 构建

## 硬约束

**tag 必须裸 semver**（`17.0.1`、`17.0.1-fork.1`）。带 `v` 会让 `releng/frida_version.py`
抛 `VersionParseError`，而 `meson.build` 用 `check: true` 调它，configure 直接失败。
过滤器写 `tags: ["[0-9]*"]`。

**不自算 FRIDA_VERSION**：meson 自己算；手算只会在 tag 非法时提前挂掉，掩盖真实错误。

**enable/disable 由 `meson.options` 生成**（`meson_configure.py: register_meson_options`，`_`→`-`）：

| 默认值 | 可用参数 |
| --- | --- |
| enabled | `--disable-x` |
| auto | `--enable-x` / `--disable-x` |
| disabled | 仅 `--enable-x` |

默认 disabled：`portal` `frida_net` `frida_node` `frida_swift` `frida_qml`。

## 构建

```yaml
python3 tools/ensure-submodules.py          # releng + gum + core
# 前置：NDK r29 → ANDROID_NDK_ROOT；Go（compiler backend）
./configure --prefix="$FRIDA_PREFIX" --host=android-<arch> \
  --enable-gadget --enable-server \
  --disable-inject --disable-graft-tool --disable-frida-tools --disable-frida-python
make -j"$(nproc)" && make install
```

## 产物

| 目标 | 路径 |
| --- | --- |
| server | `$PREFIX/bin/frida-server` |
| gadget 64（arm64/x86_64） | `$PREFIX/lib/frida-1.0/64/frida-gadget.so` |
| gadget 32（arm/x86） | `$PREFIX/lib/frida-1.0/32/frida-gadget.so` |

## 按 ABI 省额度

```yaml
prepare:
  outputs:
    matrix: ${{ steps.compute.outputs.matrix }}   # {"arch":["arm64","x86_64"]}
build:
  needs: prepare
  strategy:
    matrix: ${{ fromJSON(needs.prepare.outputs.matrix) }}
```

未选中的 ABI 不分配 runner。

## 压包与发布

```bash
xz -T0 -6 out/*     # frida-server-android-arm64.xz / frida-gadget-....so.xz
```

```bash
gh release view "$TAG" >/dev/null 2>&1 \
  && gh release upload "$TAG" dist/* --clobber \
  || gh release create "$TAG" dist/* --generate-notes
```

需 `permissions: contents: write`；tag 含 `-` 加 `--prerelease`。

## 坑

| 现象 | 原因 |
| --- | --- |
| configure 报 `VersionParseError` | tag 带 `v` |
| `unrecognized arguments: --disable-portal` | 该 feature 默认 disabled |
| Release 无产物 | 构建早失败，看 job 日志 |
| 网页无 "Run workflow" | workflow 不在默认分支，用 tag 或 `gh workflow run --ref` |
| push 即全量 CI | 自带 `ci.yml` 为 `on: push`，改名或 Disable |

## 验收

- tag 能过 `frida_version.py`
- 无硬编码 `FRIDA_VERSION`
- 每个 enable/disable 与默认值组合相符
- 32/64 分支正确
- Release 含 `.xz` 与 `SHA256SUMS.txt`
