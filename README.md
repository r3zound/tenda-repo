# tenda-repo — Tenda BE12 Pro 自建 apk 源

> **这个仓库只存构建产物，不存代码。**
> 构建逻辑在 [Tenda-BE12-Pro-OpenWrt](https://github.com/r3zound/Tenda-BE12-Pro-OpenWrt)
> 的 `scripts/fetch-proxy-packages.sh` 和 `scripts/build-repo.sh` 里，
> 由固件的 CI 每周一自动推送到这里。

## 为什么需要自建源

内核模块的 **vermagic 绑死内核版本**。固件走 SNAPSHOT，内核号天天变，
任何外部源的 `kmod-*.apk` 都是「上一版内核编的」，装上去必然失败。

实测别人的源给的是 `kmod-nft-tproxy-6.18.52-r1`，而固件已经是 `6.18.54`。
而 `kmod-nft-tproxy` 是透明代理的命门 —— 没有它，PassWall / OpenClash /
HomeProxy / Nikki-RS 的透明代理模式**全起不来**。

所以这里的每一个 `.apk` 都和固件**在同一次 CI、同一个 build root 里产出**，
ABI 天然 100% 匹配。

## 目录结构

按分支分目录，两套索引互不干扰（包同名但编译基线不同，不能混）：

```
/main/                  OpenWrt SNAPSHOT（内核 6.18，滚动）
  packages.adb
  packages.adb.sig
  *.apk
/immortalwrt-25.12/     ImmortalWrt 25.12（内核 6.12，稳定）
  packages.adb
  packages.adb.sig
  *.apk
```

## 设备怎么用

公钥已经打进固件（`/etc/apk/keys/public-key.pem`），
**不需要** `--allow-untrusted`，也不需要任何额外操作。

```sh
# 1. 加自建源（按你的固件分支二选一）
mkdir -p /etc/apk/repositories.d
echo 'https://r3zound.github.io/tenda-repo/main/packages.adb' \
  >> /etc/apk/repositories.d/customfeeds.list

# 2. 拉索引
apk update

# 3. 装面板（全是 noarch，装哪个都行）
apk add luci-app-passwall      # 或 luci-app-homeproxy / luci-app-openclash
apk add luci-app-nikki-rs
```

想回退就把 `customfeeds.list` 里那一行删掉，再 `apk update`。

## 签名

索引用 RSA 3072 签名，**私钥只存在于 GitHub Actions secrets**，从不进任何仓库。
公钥（`public-key.pem`，指纹 `74ea4b93…`）随固件发布。

这是刻意的安全边界：**只有刷了本项目固件的设备才信任这个源**。

## 内容

| 类别 | 包 |
|------|-----|
| 代理面板 | PassWall、PassWall2、SSR-Plus、HomeProxy、OpenClash、Nikki-RS、Momo、NeKoBox、MosDNS、Hijpass、v2rayA、AdGuardHome |
| 代理内核 | sing-box、mihomo、xray-core、clash-rs |
| DNS/去广告 | adblock-fast |
| **内核模块** | `kmod-nft-tproxy`、`kmod-nft-socket`（**只有这里有别人没有的东西**） |

## 发布保证

构建时有一道硬校验，不过就**拒绝发布**（GitHub Pages 保留上一版好内容）：

1. 每个 `kmod-*.apk` 的内核号必须等于本次固件的内核号
2. `kmod-nft-tproxy` 必须在产物里，缺了就拒发
3. 面板缺失只告警不阻断（面板是可选的，固件本身不受影响）

## 上游

面板来自这些仓库（由 `fetch-proxy-packages.sh` 挂进 `package/`）：
`Openwrt-Passwall/*`、`coolsnowwolf/luci`、`immortalwrt/homeproxy`、
`vernesong/OpenClash`、`CHKayanami/OpenWrt-nikki-rs`、`sbwml/luci-app-mosdns` 等 13 个。
它们都不是本项目的一部分，仅作构建输入。
