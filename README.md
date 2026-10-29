# Chaysu（小米 6.12 / android16-6.12 适配）

基于 [baizhou04/chaysu](https://github.com/baizhou04/chaysu)（KernelSU 深度定制版）的适配分支，
面向 **小米 6.12 内核**（KMI `android16-6.12`）设备。

## 这个分支做了什么

- **合并上游 `e55669a`**：`KSU_VERSION` 环境变量覆盖、`/data/adb/ksud → kud` 兼容链接
- **小米 6.12 适配**：KMI 兜底列表、内核版本串推断（`Linux version 6.12` → `android16-6.12`）、
  内置 `android16-6.12_kernelsu.ko`（按目标设备 release 编 vermagic）
- **修复模块安装必然失败**：`userspace/ksud/src/installer.sh` 在 Windows 上被检出成 CRLF，
  经 `include_str!` 编进 ksud 后 busybox ash 直接退出（3 ms、零输出 → `Failed to install module script`）。
  修复：`.gitattributes` 强制 `*.sh` 为 LF + 全部脚本转 LF
- **修复版本号静默回退**：`kernel/Kbuild` 在没有 git 历史且未指定 `KSU_VERSION` 时不再回退到 16，
  直接 `$(error)`；ksud 在模块安装前自检内核版本
- **容错**：开机搬运 `modules_update → modules` 时单个模块失败不再中断整批

版本：**32684 / v1.1.3-mi612**

## 构建

管理器：

```bash
cd manager
KEYSTORE_FILE=... KEYSTORE_PASSWORD=... KEY_ALIAS=... KEY_PASSWORD=... \
  ./gradlew assembleRelease -PKSU_PACKAGE_NAME=com.chayu.ksu -PKSU_NAME=Chaysu
```

ksud（会把 `userspace/ksud/bin/aarch64/*_kernelsu.ko` 一起内嵌）：

```bash
export KSU_VERSION=32684
cargo build --target aarch64-linux-android --release --manifest-path userspace/ksud/Cargo.toml
```

内核模块（android16-6.12，需要 DDK 容器）：

```bash
docker run --rm --privileged -v "$PWD":/work -w /work/kernel \
  ghcr.io/ylarod/ddk-min:android16-6.12-20260828 \
  bash -c 'CONFIG_KSU=m KSU_VERSION=32684 CC=clang make \
    KSU_PACKAGE_NAME=com.chayu.ksu \
    KSU_EXPECTED_SIZE=0x0377 \
    KSU_EXPECTED_HASH=<你的证书 SHA-256>'
```

> KO、ksud、管理器三处的 `KSU_VERSION` 必须一致；`KSU_EXPECTED_SIZE`/`HASH` 必须与你自己的签名证书一致。

## 说明

本仓库仅包含**源码**；编译产物（APK / `kernelsu.ko`）见 Releases。
内核部分 GPL-2.0，管理器部分 GPL-3.0，遵循上游授权。
