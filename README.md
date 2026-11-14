# Akile DNS 解锁怎么用：免费吗？能解 Netflix 和 ChatGPT 吗？脚本会不会改坏系统 DNS？（附实测结果与套餐价格）

AKDNS 现在免费公测，不买 Akile 的机器也能免费绑 10 个 IP。用法是把服务器的 resolver 指向它，解析 Netflix、Disney+、ChatGPT 这类域名时返回对应地区的可解锁节点，其余流量照常直连，不装客户端也不建隧道。

下面按实际会遇到的顺序讲：它靠什么工作、六步怎么开、那条 `wget | bash` 到底会改你系统的哪些文件，以及 2026 年 3 月一份香港测评里它解成了什么样、哪些平台翻了车。

> **想快速决定的话**
> 机器已经有了、只想解锁：直接用 AKDNS 免费公测，不必为解锁再买一台。
> 想要开箱即用、不想动系统 DNS：买自带解锁的入门套餐，日本 JPHyper-Lite ¥14.99/月、洛杉矶 LAXLite-One ¥8.88/月起、新加坡 SGLite-One ¥10.42/月起（2026 年 10 月比价资料口径，结算页为准）。
> 👉 [查看 AkileCloud 当前套餐与结算价](https://akile.io/register?aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)

## AKDNS 是怎么把内容「解」出来的

路由发生在 DNS 层。你的服务器去解析 `netflix.com`，AKDNS 返回一个该地区可解锁节点的 IP，这条请求就从那个出口出去。没配置过的域名按正常解析走本地网络，所以其他业务的延迟基本不受影响，也没有任何东西被代理。

规则按 IP 配置，每台机器独立。你在控制台里给某个 IP 勾了哪些服务、指向哪个地区，只对这个 IP 生效。

支持的平台目前 60 多个，按用途分：

- 流媒体：Netflix、Disney+、HBO Max、YouTube、Prime Video、Hulu、Paramount+、HotStar、DAZN
- AI 服务：ChatGPT、Claude、Gemini、Sora、Copilot、Meta AI
- 日本本地：ABEMA、U-NEXT、TVer
- 台湾本地：巴哈姆特动画疯、KKTV
- 香港本地：MyTVSuper、ViuTV
- 其他：TikTok、Instagram、Reddit、Spotify

节点覆盖香港、日本、美国、新加坡、韩国、台湾、德国、英国 8 个地区，走多区域 BGP。安装脚本会先把这些节点测一遍速，再自动选最优的那个。

## 开通六步：哪一步最容易漏

1. **在控制台添加机器 IP**。非 Akile 用户有 10 个免费额度，Akile 用户不限。
2. **跑解锁基线测试**。先看这台机器原本能解什么，后面才有对比。菜单里的流媒体解锁检测调的是第三方开源脚本 1-stream/RegionRestrictionCheck，脚本文案自己写着「检测脚本与结果非 AKDNS 官方」，看到 Yes/No 别当成官方承诺。
3. **DNS 测速**。菜单里有个「测速并设置」把第 3、4 步合并了，嫌麻烦可以直接选它。
4. **设为系统 DNS**。这一步动作最大，下一节单独讲。
5. **回控制台勾选这个 IP 要解锁的服务和方向**。最容易漏的就是这步，脚本检测到未解锁时提示的也是「回控制台勾服务再复测」。
6. **复测**。确认改动生效。

第 5 步是逐服务勾选，不是给整台机器选一个国家。这一点很关键，后面实测部分会看到同一台机器上 Netflix 走日本区、Disney+ 走美国区可以同时成立。

## 「设为系统 DNS」到底改了什么：会停掉 systemd-resolved，还会锁文件

这个菜单项的名字听起来温和，实际操作是接管。脚本会把系统上管理 DNS 的组件停掉，写死 `/etc/resolv.conf`，再 `chattr +i` 锁定，防止 dhclient 或 cloud-init 在重启时覆盖回来。

具体动了这些：

- **停用** systemd-resolved、NetworkManager、resolvconf、netplan 中负责 DNS 的那部分
- 写入静态 `/etc/resolv.conf`，指向测速选出的最优节点
- 用 `chattr +i` 锁住该文件，防止被动态覆盖

这么干的原因不难理解。很多发行版的 `/etc/resolv.conf` 是软链接，或者由上面的管理器动态生成，你手动追加一行 nameserver，重启一次就没了。

**临时模式与永久模式**：菜单里两种都能选。临时模式重启后失效，想先试水的用这个，验完再决定要不要转永久。

**强制 TCP**：只有在你选择「手动输入 DNS 地址」时才会问要不要强制走 TCP，选了会往配置里写 `options use-vc`。适用场景是 UDP 53 全部超时，一些网络会直接把 UDP 的 DNS 查询拦掉。

**还原路径**：菜单里的「还原 DNS 配置」会解锁文件、重新启用原来的管理器、恢复原配置。还原之前它会再备份一次当前配置，备份失败就中止还原。还原范围覆盖 `/etc/resolv.conf`、`/etc/systemd/resolved.conf` 及其 `resolved.conf.d` 下的 drop-in、NetworkManager 的配置文件与按 UUID 保存的连接级 DNS 设置、netplan、`/etc/nsswitch.conf`，并重新启用 systemd-resolved 和 resolvconf，同时清掉自己写进去的 NetworkManager `dns=none`。还原完会做校验，结果分「已还原并验证通过」和「已还原但验证未通过，请手动检查」两种。

对一台只跑网站或代理的干净机器，这些操作是可逆的，回滚路径也写得清楚。下面几类机器建议先读完脚本再决定：

- **生产机**，尤其 DNS 还被内网域名解析、域控、K8s 节点依赖的
- **跑 Docker 的机器**。容器默认继承宿主的 `/etc/resolv.conf`，宿主一改，容器内的解析路径跟着变
- **自己搭了 DNS 的机器**，AdGuard Home、Pi-hole、dnsmasq、CoreDNS 这类
- **用 netplan 管网络的 Ubuntu**，还原后要确认 netplan 有没有被重新 apply

脚本是公开的，就在 `akile-network/aktools` 仓库的 `akdns.sh` 里，官方在公告帖里也写了「如果你对 `wget | bash` 有顾虑，这个直觉是对的，先读再跑」。装的时候是在 root 或 sudo 下跑那一行管道命令，进交互菜单。

这台机器要是生产机，先拿免费额度在备用机上把流程走一遍，确认还原正常，再去动主力。👉 [查看 AkileCloud 入门套餐价格](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)

## 实测能解什么、解不了什么

以下结果来自 2026 年 3 月 30 日发布的一篇香港 HKLite 测评，机器配置 1 核 / 512M / 5G SSD / 200G@1000Mbps，当时的月付价是 3.99 元。作者在同一台机器上分别切了新加坡、日本、美国三个方向跑检测，价格和库存现在可能已经变了，结果只代表那个时点。

| 平台 | 首轮（默认方向） | 切日本方向 | 切美国方向 |
| --- | --- | --- | --- |
| Netflix | Yes (SG) | Yes (JP) | Yes (US) |
| Disney+ | Yes (SG) | Yes (JP) | Yes (US) |
| HBO Max | Yes (SG) | — | Yes (US) |
| YouTube Premium | Yes (HK) | — | Yes (HK) |
| Prime Video | Yes (HK) | — | Yes (HK) |
| HotStar | Yes (SG) | — | — |
| Dazn | No (Banned) | — | — |
| Hulu | — | No | No |
| U-NEXT | — | Failed | — |
| Peacock TV | — | — | No |
| Crunchyroll | — | — | No |

日区方向解开的还有一大串本地服务：NHK+、DMM TV、Abema.TV、Niconico、Telasa、TVer、WOWOW、VideoMarket、D Anime Store、FOD、Radiko、Lemino、Rakuten TV JP。美区方向 Yes 的包括 FOX、NFL+、ESPN+、Fubo TV、Pluto TV、SHOWTIME、BritBox、Acorn TV。

三个能直接拿去用的判断：

**一台机器可以按服务分地区。** Netflix 在日本方向显示 JP、Disney+ 在美国方向显示 US，两组结果同时成立。所以控制台里是逐服务勾节点，不是整机选一个国家。你想让 Netflix 走日本、Disney+ 走美国，技术上没问题，前提是这些方向在你这个 IP 上都测得通。

**这套解锁走 IPv4。** 同一篇测评的 IPv6 段几乎全线「IPv6 Not Support」，Netflix、Disney+、HotStar 直接 Failed (Network Connection)，HBO Max 是 No。机器只有 IPv6 出口的话，这套方案基本用不上。

**不是万能。** Dazn 直接 Banned，Hulu 两个方向都不通，Peacock、Crunchyroll、Tubi、Sling 在美国方向也是 No。买之前先确认你要看的平台在不在支持列表里，在列表里的也可能因为平台风控当天失效。测评里 Netflix Preferred CDN 一栏显示 `Hijacked with [Akile]`，说明 Netflix 流量被 DNS 接管到了 Akile 自己的 CDN 节点，这条路径是商家侧维护的，平台一收紧就得等他们更新。

## 套餐与价格

AKDNS 本身免费，不需要为它买机器。如果你想要一台 DNS 解锁已经内置、开箱即用的 VPS，下面是公开渠道能核到的在售方案。价格整理自 2026 年 10 月更新的第三方比价资料，AkileCloud 的库存和促销调整频繁，下单页的数字才是准的。

| 地区 | 套餐 | CPU / 内存 / 硬盘 | 月流量 / 带宽 | 月付价格 | 购买 |
| --- | --- | --- | --- | --- | --- |
| 洛杉矶 | LAXLite-One | 1 核 / 1GB / 5GB | 1TB / 5Gbps | ¥8.88 起 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=2&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 洛杉矶 | LAXLite-Mini | 1 核 / 2GB / 10GB | 2TB / 5Gbps | ¥15.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=2&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 洛杉矶 | LAXLite-Starter | 2 核 / 2GB / 10GB | 5TB / 5Gbps | ¥34.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=2&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 洛杉矶 | LAXPro-Light | 1 核 / 1GB / 20GB | 300GB / 300Mbps | ¥43.74 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=2&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 洛杉矶 | LAXPro-Standard | 1 核 / 1GB / 20GB | 1TB / 500Mbps | ¥74.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=2&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 日本 | JPHyper-Lite | 1 核 / 2GB / 10GB | 1TB / 2.5Gbps | ¥14.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 日本 | JPHyper-Mini | 2 核 / 2GB / 10GB | 2.5TB / 2.5Gbps | ¥25.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 日本 | JPPro-Light | 1 核 / 1GB / 10GB | 700GB / 300Mbps | ¥24.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 日本 | JPPro-Starter | 1 核 / 1GB / 20GB | 500GB / 1Gbps | ¥37.35 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 日本 | JPPro-Standard | 1 核 / 1GB / 20GB | 1TB / 1.2Gbps | ¥62.25 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 新加坡 | SGLite-One | 1 核 / 1GB / 5GB | 1TB / 500Mbps | ¥10.42 起 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=4&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 新加坡 | SGLite-Mini | 1 核 / 1GB / 5GB | 1TB / 1Gbps | ¥24.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=4&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 新加坡 | SGLite-Starter | 1 核 / 1GB / 10GB | 3TB / 1Gbps | ¥43.74 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=4&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 新加坡 | SGLite-Standard | 1 核 / 1GB / 10GB | 6TB / 1.5Gbps | ¥87.49 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=4&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |
| 香港 | HKLite-Mini | 1 核 / 1GB / 10GB | 2.5TB / 1Gbps | ¥24.99 | [ 选购](https://akile.io/shop/server?type=traffic&areaId=3&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811) |

上表只收录了公开比价资料里出现过的型号。台湾、韩国、英国、德国的方案在官网商店页有单独分区，本文没拿到可核验的当前价，配置和价格以下单页为准。另外洛杉矶还有一组主打原生 IP 的 ISP / 住宅双 ISP 方案，资料里出现过的入门价在 ¥49.99 到 ¥54.99 之间、流量标注不一，这里不并进表，需要的话去商店页看当期档位。

从这张表能看出两件事。

日本的 JPHyper 系列比 JPPro 便宜，带宽反而高（2.5Gbps 对 300Mbps 至 1.2Gbps），流量也多。官方套餐说明只写了「包含直连和路由调优型号」，没指明哪个型号走哪种路由。你只是要看日区内容的话，JPHyper-Lite ¥14.99 就够了，多花二十多块买 JPPro 之前先看清下单页的线路说明。

洛杉矶的 LAXPro 比 LAXLite 贵出一大截，带宽反而低（300 到 500Mbps 对 5Gbps），流量也少。LAXPro 卖的是线路优化，不是配置。只用来解锁的话，LAXLite-One ¥8.88 起更合适；需要三网优化的场景才轮到 LAXPro。

## 退款、流量和试错成本

退款是阶梯式的：新购 3 天内退 80%，7 天内退 50%，10 天内退 20%，超过 10 天不退。也就是说，最划算的反悔窗口只有头 3 天，而且无论如何都会扣掉 20%。

流量超了不会停机，会限速到共享 10Mbps。要恢复可以重置流量，¥7.00 一次。支付支持支付宝、FPS、USDT 和信用卡。

各方案页面都有 Looking Glass。真买了，顺序应该是：先测延迟，再跑 AKDNS 的解锁基线测试，确认你要的平台在这个 IP 上真的能解开，最后才把主力业务的 DNS 切过去。反过来做，出问题时你已经把生产环境的解析改掉了。👉 [挑一台入门配置先验证](https://akile.io/shop/server?type=traffic&areaId=1&aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)

## 该选哪种

按需求对号入座：

- **已经有机器，只想要解锁**：用 AKDNS 免费公测，别买新机器。10 个免费 IP 额度对个人够用了。
- **机器是生产机、跑 Docker 或自建 DNS**：先用免费额度在备用机上把开通和还原各走一遍，确认没问题再动主力。
- **主要看日区内容**（ABEMA、TVer、DMM、Niconico）：日本节点。JPHyper-Lite ¥14.99/月，比 JPPro 便宜且带宽高。
- **主要看美区**（HBO Max、ESPN+、Fubo、SHOWTIME）：洛杉矶。LAXLite-One ¥8.88/月起，只解锁不上高配。
- **要港区内容**（MyTVSuper、ViuTV）或需要低延迟回国：香港节点，但这一档在 ¥24.99/月，明显贵过日本和美国，先确认是不是非港区不可。
- **只是跑脚本、跑 bot、要便宜**：入门档就行。

下单前确认两件事。一是结算页的实际价格和库存，上表是 2026 年 10 月的第三方口径，AkileCloud 的促销和补货节奏很快；二是你要看的平台在不在支持列表里、那个地区方向在你的 IP 上能不能解，拿免费额度先测，测通了再掏钱买机器。

有读者会问优惠码，第三方整理里流传较广的是 `AkileCloudLsStart`，号称轻量云永久 8 折、续费同享。这类码能不能用、能不能叠加，以结算页实际抵扣为准，我没法替它担保。

👉 [查看 AkileCloud 全部套餐与当前结算价](https://akile.io/register?aff_code=24a6f6bd-f995-4158-a9fd-f4b6aef27811)
