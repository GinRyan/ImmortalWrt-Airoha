# xg040g-wifi-defaults

> Nokia XG-040G-MD（Airoha AN7581）+ USB 无线网卡，把固件配成
> **STA 无线转有线桥接** 的开箱可用状态。绑在 `packages/` 本地 feed 里，
> CI 会自动编进固件；刷完**不需要任何刷后 shell**。

**一句话**：`/etc/uci-defaults/99-xg040g-wifi` 在首次启动时关掉所有 AP 接口、
纠正 `country`/`band`、并补齐 `wwan` 与防火墙归属，然后自删。

---

## 一、为什么必须有这个包（本机 2026-10-09 实测结论）

### 结论 1 ⭐ 这块 MT7921AU 是「单 vif 卡」，**只能做 STA，做 AP 会崩固件**

- 只要 AP 接口存在并启动，STA 接口**根本不会被创建**：`iw dev` 里只有 `phy2-ap0`，没有 `phy2-sta0`。
- AP 启动过程本身会让固件挂掉，随后 USB 整条链路掉线、只能物理重插：

  ```text
  phy1-ap0: failed to set key (4, ff:ff:ff:ff:ff:ff) to hardware (-110)   # 设组密钥超时 ETIMEDOUT
  mt7921u 2-1:1.0: Message 00020001 (seq 8) timeout
  xhci-mtk 1fab0000.usb: Timeout while waiting for setup device command
  usb 2-1: USB disconnect, device number 2
  usb usb2-port1: Cannot enable. Maybe the USB cable is bad?
  usb usb2-port1: unable to enumerate USB device
  ```

- 网卡 USB ID = `0e8d:7961`，挂在 `1fab0000.usb`（完整 USB3 控制器）上。
- 软件救不回来（`unbind/bind xhci-mtk`、`wifi up` 都无效）→ **必须物理重插**。
- ✅ **关掉 AP 接口后，STA 立刻正常，且不再崩**：

  ```sh
  uci set wireless.default_radio0.disabled='1'
  uci commit wireless && wifi reload
  ```

### 结论 2 `band '6g'` 会让 LuCI 的加密列表只剩 WPA3

本卡固件**确实上报 6 GHz 频段**（`5955/5975/5995 MHz`），所以 LuCI 的“波段”下拉里会出现 6 GHz。
一旦选中（或配置里被写成 `option band '6g'`），LuCI 的判定是：

```js
// luci-mod-network / view/network/wireless.js
const is_6ghz = uci.get('wireless', <radio设备>, 'band') == '6g';
if (has_hostapd || has_supplicant) {
    if (!is_6ghz) {                    // ← 6GHz 时这三项被跳过
        crypto_modes.push(['psk2',      'WPA2-PSK', 35]);
        crypto_modes.push(['psk-mixed', 'WPA-PSK/WPA2-PSK Mixed Mode', 22]);
        crypto_modes.push(['psk',       'WPA-PSK', 12]);
    }
}
if (has_ap_sae || has_sta_sae) crypto_modes.push(['sae', 'WPA3-SAE', 31]);
```

→ **WPA / WPA2 / mixed / EAP 全部不出现，只剩 WPA3-SAE 与 OWE**。
（6 GHz 强制 WPA3 是规范要求，不是 bug；错的是把 6 GHz 和 5 GHz 的信道混着配。）

### 结论 3 `country '00'` 会让 hostapd 直接拒绝启动

```text
daemon.err hostapd: Line 7: Invalid country_code '00'
daemon.err hostapd: 2 errors found in configuration file '<inline>'
daemon.notice hostapd: hostapd.add_iface failed for phy phy0 ifname=phy0-ap0
```

### 结论 4 wpad / hostapd / wpa-supplicant 这 21 个变体**互相冲突**，只能留一个

hostapd 的 Makefile 里它们互相 `CONFLICTS`（都 `PROVIDES hostapd` / `wpa-supplicant`）：

```make
define Package/wpad/Default
  PROVIDES:=hostapd wpa-supplicant
  CONFLICTS:=$(HOSTAPD_PROVIDERS) $(SUPPLICANT_PROVIDERS)
```

同时选多个时**哪个生效取决于构建顺序**；而 `CONFIG_SAE` 只在 `-openssl/-mbedtls/-wolfssl`
分支里加，所以内部 TLS 的 `wpad` / `wpad-basic` / `hostapd-basic` 天生**没有 WPA3**，
`-mini` 更是只有 WPA-PSK。实测装上 `hostapd-basic-mbedtls` 时，`ubus call luci getFeatures` 给出：

```json
"hostapd": { "sae": true, "owe": true, "eap": false, "suiteb192": false, "wep": false, ... }
```

→ WPA-Enterprise / WPA2-Enterprise / WPA3-EAP/192-bit **永远不会出现在列表里**。
**要全都有，就只装 `wpad-openssl`**（full + OpenSSL）。

### 结论 5 `nft` 里搜不到 `masquerade` 不代表没有 NAT

本固件的 wan zone 启用了 **fullcone NAT**（`fw4 check` 会打印
`Section @zone[1] (wan) IPv4 fullcone enabled for zone 'wan'`），fullcone 模式**不用**
`masquerade` 关键字。别据此判断 NAT 坏了。

### 结论 6 不要用多网卡 PC 直接测「LAN 能不能出网」

测试机同时有 `192.168.1.x` 与上游 `192.168.2.x` 时，`ping -S` 的源地址选择会走错出口，
看起来像路由器没转发。用路由器自身强制源地址测最可靠：

```sh
ping -I 192.168.1.1 1.1.1.1        # 强制用 LAN 网段源地址 → 0% 丢包说明 NAT 正常
```

---

### 结论 7 ⭐ `ra_default='1'` 会让 Windows 显示「IPv6：无 Internet 访问」，**尽管 IPv6 完全通**

**症状**：路由器已经拿到 IPv6（有 PD 委派、能 ping 通 IPv6 外网），但 PC 的网络状态里
IPv6 显示「无 Internet 访问」。

**先证伪**（数据面全通，问题不在连通性）：

```text
# 本机（Windows，走路由器那块网卡）
curl -6 http://ipv6.msftconnecttest.com/connecttest.txt   → HTTP 200
curl -6 --interface <路由器委派前缀地址> https://www.taobao.com → HTTP 200
ping -6 -S <委派前缀地址> 2400:3200::1                    → 0% loss
# 路由器自身
ping -6 -I 2409:8a3c:b51:d714::1 2400:3200::1             → 0% loss（LAN 前缀作源也通）
```

**真因**：`dhcp.lan.ra_default` 被设成了 `1`。官方 `/etc/config/dhcp` 文档的语义是：

| 值 | 何时才通告默认路由器寿命 |
| --- | --- |
| `0`（默认） | 有默认路由 **且** 接口有全局地址 |
| `1` | 有默认路由 **但** 接口没有全局地址 |
| `2` | 两者都没有 |

br-lan **有全局地址**（`2409:8a3c:b51:d714::1/62`，来自上游 PD），所以 `1` 的条件不成立，
odhcpd 把 RA 里的 **router lifetime 设成 0**，并在日志刷：

```text
daemon.warn odhcpd[2908]: No default route present, setting ra_lifetime to 0!
```

router lifetime = 0 的意思是"**我（这个路由器）不是默认路由器**"。于是主机即使有地址、
数据面也通，操作系统仍会把该网卡判成没有 IPv6 Internet。

**修复与验证**：

```sh
uci set dhcp.lan.ra_default='0'    # 或直接删掉该选项，回到文档默认值
uci commit dhcp
/etc/init.d/odhcpd restart
```

修复后 Windows 自己的判定字段立刻变好：

```text
InterfaceAlias  IPv4Connectivity  IPv6Connectivity
Ethernet        Internet          Internet          ← 修复前是「无 Internet 访问」
```

**顺带发现的两点（不影响使用，了解即可）**：

- `ubus call dhcp ipv6leases` 显示 **br-lan 一个 DHCPv6 租约都没有**；PC 在路由器这块网卡上
  只有 SLAAC 地址、**没有 IPv6 DNS**（`ra_flags` 里带 `managed-config`，即"地址和 DNS 都去问
  DHCPv6"，而 DHCPv6 没成）。因为 Windows 还能用 IPv4 的 DNS（192.168.1.1）解析 AAAA，
  所以功能上无感。若想让 LAN 走"纯 SLAAC + RDNSS"，可把 `dhcp.lan.ra_flags` 设为 `none`
  （`ra_slaac` 与 `ra_dns` 默认都开）。
- 如果测试机**同时**直连上游 WiFi（本机 `WLAN 6` 就是），它的 IPv6 默认路由会被 metric 更低的
  无线网卡抢占（`WLAN 6` metric 10 < `Ethernet` metric 25），`tracert` 第一跳会是上游网关。
  这会让"到底走没走路由器"变得难以判断 —— 判断时请用 `curl --interface <源地址>` 强制指定源。

## 二、实测通过的桥接状态

| 项 | 值 |
| --- | --- |
| 关联 | `phy2-sta0` Client 模式，SSID `pfc2022`，ch36 / VHT80 / 1080 Mbit/s / -27 dBm |
| 上游地址 | `192.168.2.112/24`，`default via 192.168.2.1 dev phy2-sta0` |
| LAN | `br-lan` = lan1-4，`192.168.1.1/24`，STA 网段经 fullcone NAT 出网 |
| 无线接口 | `iw dev` 里只有 `phy2-sta0`，**没有 `-ap0`** ← 稳定不崩的关键 |

---

## 三、用法

1. 包已在 `packages/xg040g-wifi-defaults/`（本地 feed `src-link loong` 自动收录），
   并在 workflow 的 `extra_packages` 默认值里预填了 `xg040g-wifi-defaults`。
2. 「刷完自动连上游」：改 `files/99-xg040g-wifi` 顶部

   ```sh
   UPLINK_SSID="..."          # 上游 SSID
   UPLINK_ENCRYPTION="psk2"   # 或 sae-mixed / sae ...
   UPLINK_KEY="..."
   ```

   ⚠️ **本仓库是公开的，不要把 WiFi 密码提交进来**；要固化就放在自己的私有分支/私有 fork，
   或者留空、刷完在 LuCI 里连一次（会写进设备的 `/etc/config/wireless`，不进仓库）。
3. 其余可调项：`COUNTRY`（默认 `CN`）、`DEMOTE_6G`、`DISABLE_AP`、`SETUP_FIREWALL`、`WAN_IFACE`、
   `FIX_IPV6_RA`（默认 `1`：把 `dhcp.lan.ra_default` 钉回 `0`，见结论 7）。

## 四、刷完自查

```sh
iw dev                        # 应只有 xxx-sta0，没有 xxx-ap0
ip -4 addr show               # sta0 上应有上游网段地址
ip route                      # default via <上游网关> dev <sta0>
ping -I 192.168.1.1 1.1.1.1   # 强制 LAN 源地址验证 NAT
```

## 五、症状 → 原因 对照表

| 症状 | 原因 | 处置 |
| --- | --- | --- |
| 加密列表为空 | LuCI 认为 hostapd/wpa_supplicant 没装（`has_hostapd`/`has_supplicant` 全 false） | 确认 `hostapd-common` 与无线主体版本一致；别混装 |
| 加密列表只剩 WPA3-SAE / OWE | `band == '6g'` | 改回 `5g`（本包自动做） |
| 没有 WPA3 选项 | 装的是内部 TLS 的 `wpad`/`wpad-basic`（无 `CONFIG_SAE`） | 换 `wpad-openssl` |
| 没有 Enterprise 选项（WPA-EAP…） | 装的是 `*-basic*`（`eap:false`） | 换 `wpad-openssl` |
| AP 起不来 + hostapd 报 `Invalid country_code '00'` | `country '00'` | 设有效国家（本包自动做，默认 `CN`） |
| 起 AP 就把网卡搞掉线、只能重插 | 单 vif 卡 + AP 路径崩固件 | **别用 AP**，关掉 AP 接口（本包自动做） |
| STA 拿到 IP 但 LAN 出不了网 | `wwan` 不在 wan zone / 没 masq / 没 forwarding | 本包自动补齐（`SETUP_FIREWALL=1`） |
| IPv6 有地址但 Windows 显示「无 Internet 访问」 | `dhcp.lan.ra_default='1'` → RA 的 router lifetime=0 | 改回 `0`（本包 `FIX_IPV6_RA=1` 自动做） |

## 六、上游依据（便于日后核对）

- LuCI 加密列表逻辑：`luci-mod-network` → `htdocs/luci-static/resources/view/network/wireless.js`
  （`is_6ghz`、`hasSystemFeature('hostapd', …)`、`crypto_modes`）
- 特性表数据源：`/usr/share/rpcd/ucode/luci`，运行时用 `ubus call luci getFeatures` 查看
- 变体能力与冲突：`openwrt/openwrt` → `package/network/services/hostapd/Makefile`
- 本机固件：ImmortalWrt SNAPSHOT r41528-41d7b64f93 / 内核 6.18.52 / `target airoha/an7581` /
  `aarch64_cortex-a53` / 设备符号 `nokia_xg-040g-md-ubi`
