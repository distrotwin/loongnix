# 开发指引

这个仓库把龙芯 Loongnix 桌面 25 的公开 apt 源变成可用于软件构建与测试的容器镜像。它只覆盖 **LoongArch 新世界 ABI**；旧世界不做，理由见 README 的「为什么只有新世界」。定位边界见 README 前两节。

## 硬性约定

- commit **不允许带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务
- 写进文档的版本号一律来自跑镜像实测，不取源索引里的元包版本

## 这个仓库放什么

```
loongnix/
├── buildkit/                          # submodule，钉住一个 commit
├── distros/v25.conf                   # 一个版本一个，文件名即 DID
├── keys/loongnix-archive-keyring.gpg  # 信任根，指纹钉在 conf 里
├── .github/workflows/build.yml        # 只定义矩阵，调用 buildkit 的可复用 workflow
├── README.md
├── CLAUDE.md                          # 本文件
└── AGENTS.md -> CLAUDE.md             # 符号链接，不是副本
```

只跟这个系统版本自己的事实有关（源地址、suite、包列表、ABI 基线、这一版特有的怪癖）就进 `distros/*.conf`；跟怎么构建、怎么测、怎么发有关就进 buildkit。

## 必知事实

- **全线只有 LoongArch，没有 amd64/arm64。** 两条桌面线每个 suite 的 `Release` 都核过：25 线四个 suite 全是 `loong64`，20 线八个 suite 全是 `loongarch64`；服务器线（rpm 系）的架构目录也只有 `loongarch64` 与 `source`
- **世代判据落在动态链接器上，不在架构名上。** `/lib64/ld-linux-loongarch-lp64d.so.1` 是新世界，`/lib64/ld.so.1` 是旧世界。deb 世界里 `loong64` = 新、`loongarch64` = 旧，但 rpm 世界两个世界都叫 `loongarch64`，名字不携带世代信息
- **旧世界在托管 runner 上造不出来。** 实测同一个 QEMU 8.2.2 下，25 线的 `bash` 正常跑出 `MACHTYPE`，20 线的报 `Unknown syscall 80`（ENOSYS）——上游 QEMU 不实现旧世界那两个调用
- **multiarch 三元组是 `loongarch64-linux-gnu`,而 deb 架构名是 `loong64`**，两者不同名，`lib/arch.sh` 已做映射
- **`Release` 自述 `Origin`/`Label`/`Suite` 都是 `Debian`**，只有 `Codename` 是 `loongnix-stable`。所以 conf 的 `SUITE` 要写 Codename 那一个
- **没有托管 runner，三个档位全靠 QEMU 模拟**，runner 必须 `ubuntu-24.04`（22.04 只带 QEMU 6.2，binfmt 里没有 `qemu-loongarch64` 条目）

## 本地构建与验证

```bash
docker build --build-arg http_proxy= --build-arg https_proxy= \
  -f buildkit/Dockerfile.builder -t lnbuild:latest buildkit/
docker run --rm --privileged -v "$PWD:/w" \
  -e http_proxy= -e https_proxy= -e HTTP_PROXY= -e HTTPS_PROXY= \
  lnbuild:latest bash -c 'ROOT=/w BK=/w/buildkit ARCH=loong64 /w/buildkit/build/build.sh v25 micro'
```

代理变量**大小写四个都要清**：容器会从 docker daemon 的 proxy drop-in 继承设置，而那个代理在容器里往往连不上，症状是取 `InRelease` 五轮重试全失败。宿主还需要 `qemu-user-static` 与 `binfmt-support`；不要引入 `tonistiigi/binfmt` 容器，实测它会破坏本来可用的 binfmt 注册。

## 跑 CI

`gh workflow run build.yml --repo distrotwin/loongnix -f publish=true`

三个档位全在 QEMU 下构建，一轮比其他仓库慢得多。

## 发布与验收

发布后清 GHCR 的无 tag 版本。日期钉版 tag 是**天粒度**，同一天重复发布会把 tag 移到新 digest、让上一次同日构建变成孤儿。删之前必须先把所有 tag 的 manifest 展开、收集成员 digest 做白名单。

## 排错

先看构建环境记录那一步：本 job 的输入、conf 解出的值、宿主信息都在那里。读日志读不出来时把制品下下来直接看。
