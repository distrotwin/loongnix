# 龙芯 Loongnix 桌面操作系统 · 构建与测试镜像

对着龙芯 Loongnix 桌面 25 的公开 apt 源自举出来的容器环境，用于**软件构建、打包与兼容性测试**。只覆盖 **LoongArch 新世界 ABI**（`loong64`）——旧世界的 20 线在托管 runner 上构建不出来，理由见下文——公开在 GHCR。最近一轮 3 个镜像、119 项检查全部通过，零异常。

```bash
docker run --rm --platform linux/loong64 ghcr.io/distrotwin/loongnix:v25-devel \
  bash -c 'grep PRETTY /etc/os-release; ldd --version | head -1; gcc -dumpfullversion'
```

在 x86 机器上**必须带 `--platform linux/loong64`**：这个仓库的 manifest 里只有一个架构，docker 默认按宿主平台挑，挑不到就报 `no matching manifest for linux/amd64`——那句话读起来像镜像坏了，其实是没告诉 docker 要哪个平台。宿主还需要 `qemu-user-static` 与 `binfmt-support`，且 QEMU 不低于 7.1。

## 这是什么，不是什么

镜像里没有内核——这是所有 Linux 容器镜像的常态，容器共享宿主内核。只在启动时才有意义的东西（`initramfs-tools`、驱动）不起作用，`systemd` 不是 PID 1。

所以它适合回答「编出来的东西对不对」，不适合回答「跑起来的系统对不对」。

**该用它**：在 CI 里编出能在龙芯机器上跑的二进制与 `.deb`；检查产物需要的 glibc / libstdc++ 符号版本目标系统能否满足；复现只在 LoongArch 上出现的编译问题。

**别用它**：当生产运行时基础镜像；复现内核相关行为；当作系统的完整替代品做验收测试。

## 先跑一遍

进容器，写个 A+B，编了跑，再看符号天花板。

```bash
docker run -it --rm --platform linux/loong64 ghcr.io/distrotwin/loongnix:v25-devel /bin/bash
```

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

最后那行是这套镜像最有用的一句：**它直接告诉你产物需要目标系统多新的 glibc**。

## 选哪一个

| 版本 | glibc | gcc | libstdc++ | 架构 |
|---|---|---|---|---|
| `v25` | **2.41** | **14.2.0** | 6.0.33 / `GLIBCXX_3.4.33` | `loong64`（新世界） |

三个档位：`micro` 只有 libc 与 shell，不带包管理器；`base` 加上 `apt`、`python3`、网络工具；`devel` 再加 `build-essential`、`pkg-config`。

这是四个仓库里 ABI 最新的一个——银河麒麟 V11、统信 V25、麒麟信安 V6 都停在 glibc 2.38。

## 只覆盖新世界

LoongArch 有两套互不兼容的 ABI，Loongnix 的两条桌面线各用一套。本仓库只做 **25 线（新世界 `loong64`）**；20 线（稻香湖，旧世界 `loongarch64`，glibc 2.28）**在托管 runner 上构建不出来**，卡在 QEMU 用户态模拟对旧世界信号 ABI 的支持缺失：`rt_sigaction` 在旧世界下一律返回 EINVAL，而 dpkg 跑维护者脚本前必须 `signal(SIGQUIT, SIG_IGN)`，拿到 EINVAL 就中止。要做旧世界镜像得用龙芯补丁版 QEMU 或者真机。完整的逐层排查记录在 `buildkit/docs/downstream-repo.md`。

**世代不能靠架构名判。** deb 世界里 `loong64` 是新、`loongarch64` 是旧，而 **rpm 世界两个世界都叫 `loongarch64`**，名字不携带世代信息。龙芯自己的 registry `cr.loongnix.cn` 上那个 `loongnix` 就是旧世界的 20.7，与 25.1 同名只差一个 tag。想确认手上的镜像是哪个世界，看动态链接器：

```bash
docker run --rm --platform linux/loong64 ghcr.io/distrotwin/loongnix:v25-micro \
  readlink -f /lib64/ld-linux-loongarch-lp64d.so.1
```


## 镜像是怎么造的

从公开 apt 源 `https://pkg.loongnix.cn/loongnix/25/` 直接 `mmdebstrap` 自举,不经过 ISO。

这个源是标准 Debian 形状:`dists/` + `pool/`,`Origin: Debian`、`Label: Debian`、`Suite: Debian`,只有 `Codename` 是 `loongnix-stable`——所以配置里 suite 要写 Codename 那一个。同时配了 `loongnix-security`,它与 Debian 的 security 同性质、滚动更新。

信任根是厂商公钥,指纹 `D1B8F4D3241F015CACF733D3A8C7C20CEDF1B817`(RSA 3072,2020-11-16,UID `loongsonos <service@loongnix.cn>`)。它有两个互相印证的来源:厂商把它打进了自己改的 `debian-archive-keyring_2019.1.lnd.3` 的 `/usr/share/keyrings/debian-archive-buster-loongarch64-stable.gpg`;`keys.openpgp.org` 上同一指纹的 UID 域名也是 `loongnix.cn` 与 `loongson.cn`。构建时逐条核对指纹再验 `InRelease` 签名。

## 认出自己在哪个系统上

```bash
cat /etc/os-release                  # PRETTY_NAME 与 VERSION
dpkg --print-architecture            # loong64
readlink -f /lib64/ld-linux-loongarch-lp64d.so.1   # 存在即新世界
```

## tag 与钉版

- 滚动 tag：`v25`、`v25-devel`、`v25-base`、`v25-micro`、`latest`（指向 `v25-devel`）
- 日期钉版 tag：`v25-devel-YYYYMMDD`，内容不变，用于复现

钉版 tag 是**天粒度**。同一天重复发布会把这个 tag 移到新 digest。

## 镜像自带的溯源信息

```bash
docker inspect ghcr.io/distrotwin/loongnix:v25-devel --format '{{json .Config.Labels}}' | python3 -m json.tool
```

`cn.internal.repo-commit` 是构建时这个仓库的 commit；`cn.internal.suite` 是取材的 suite；`cn.internal.tier` 是档位。

## 已知的怪癖与期望失败

- **`micro` 档没有包管理器**，这是设计：它不装 apt，出厂也不写 sources.list 与 keyring——多一把没用的 key 就是多一份可被滥用的授权
- **低地板符号检查不适用**：`manylinux2014` 没有 LoongArch 架构，而它最早的 glibc 已高于 2.17，所以「产物是否依赖过新符号」那一项在这个架构上无低地板可比，检查集里标为不适用而不是通过
- **`Release` 自述 `Origin: Debian`**，不是 Loongnix 自己的名字。这是厂商现状，如实记录

## 本地构建

```bash
git clone --recurse-submodules https://github.com/distrotwin/loongnix.git && cd loongnix
docker build --build-arg http_proxy= --build-arg https_proxy= \
  -f buildkit/Dockerfile.builder -t lnbuild:latest buildkit/
docker run --rm --privileged -v "$PWD:/w" \
  -e http_proxy= -e https_proxy= -e HTTP_PROXY= -e HTTPS_PROXY= \
  lnbuild:latest bash -c 'ROOT=/w BK=/w/buildkit ARCH=loong64 /w/buildkit/build/build.sh v25 micro'
```

代理变量**大小写四个都要清**：容器会从 docker daemon 的 proxy drop-in 继承设置，而那个代理在容器里往往连不上，症状是取 `InRelease` 五轮重试全失败、报错却指向「软件源不可达」。

## CI

`gh workflow run build.yml --repo distrotwin/loongnix -f publish=true`

构建、测试、报告、发布四个阶段。测试在**干净机器**上装载并真正启动镜像——构建阶段的机器状态会掩盖镜像自身的缺陷。三个档位全在 QEMU 下跑，一轮比其他仓库慢得多。最近一轮 3 个镜像、119 项检查：全部通过、零异常。报告与完整日志在每次 run 的 artifact 里。

## 仓库结构

```
distros/v25.conf   # 源地址、suite、包列表、ABI 基线、这一版特有的怪癖
keys/              # 厂商公钥，指纹钉在 conf 里
.github/workflows/build.yml
buildkit/          # submodule，构建与测试的全部实现
```
