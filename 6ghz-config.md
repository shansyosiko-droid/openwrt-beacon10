# 6 GHz 强开：三层配置

固件里只是"允许"6 GHz，真正让它变成可用的 AP，还要过三关。
下面按顺序做，每关都有验证命令。

---

## 第一关：regdb 生效

固件已经把你的 `regulatory.db` 打进 `/lib/firmware/`。刷完之后先确认内核认了：

```sh
# 触发一次监管域加载
iw reg set CN

# 看监管域
iw reg get
```

期望输出里出现类似：

```
country CN: DFS-ETSI
	(5945 - 6425 @ 160), (N/A, 23), (N/A), NO-OUTDOOR
	(6425 - 7125 @ 160), (N/A, 23), (N/A), NO-OUTDOOR
```

如果 6 GHz 两段没出现，说明 `regulatory.db` 没被加载——检查：

```sh
ls -l /lib/firmware/regulatory.db
dmesg | grep -i regulatory
```

---

## 第二关：ath11k 暴露 Band 4

```sh
for p in /sys/class/ieee80211/phy*; do
  echo "=== $p ==="
  iw phy $(basename $p) info | sed -n '/Band 4/,/Band/p' | head -30
done
```

期望看到 `Band 4:` 和 5955 / 6135 / 7115 这类频率。

如果三个 phy 都只有 Band 1/2，说明：
- regdb 没生效（回第一关）
- 或者 BDF 没被正确加载：`dmesg | grep -i "board-2.bin\|bdf\|ath11k.*fail"`

---

## 三关：hostapd 起 6 GHz BSS

6 GHz 不是改个信道就行，**协议层有硬性要求**：

| 要求 | 原因 |
|---|---|
| 必须 `ieee80211ax=1` | 6 GHz 只允许 Wi-Fi 6E 及以上 |
| 不能开 `ieee80211ac` / `ieee80211n` | 只在 5 GHz / 2.4 GHz 有效 |
| 必须 WPA3-SAE（或 OWE） | 6 GHz 禁止 WPA2 和开放网络 |
| 必须 `ieee80211w=2` | 强制管理帧保护 |
| 必须指定 `op_class` | 6 GHz 信道编号与 5 GHz 不同 |
| 建议从 PSC 信道起步 | 客户端靠 PSC 扫描发现 AP |

### 用 UCI 配置（OpenWrt 常规方式）

```sh
# 建立一个 6 GHz 的 SSID
uci add wireless wifi-iface
uci set wireless.@wifi-iface[-1].device='radio2'      # 确认哪一个是 6G phy
uci set wireless.@wifi-iface[-1].mode='ap'
uci set wireless.@wifi-iface[-1].ssid='6G-test'
uci set wireless.@wifi-iface[-1].encryption='sae'
uci set wireless.@wifi-iface[-1].key='你的密码'
uci set wireless.@wifi-iface[-1].ieee80211w='2'
uci set wireless.@wifi-iface[-1].channel='37'          # 6135 MHz，PSC 信道
uci set wireless.@wifi-iface[-1].htmode='HE160'
uci set wireless.@wifi-iface[-1].disabled='0'
uci commit wireless
wifi reload
```

### 直接看 hostapd 配置（调试用）

`wpad-openssl` 生成的配置文件在 `/var/run/hostapd-phyX.conf`，正常应该包含：

```
hw_mode=a
channel=37
op_class=131
ieee80211ax=1
ieee80211ac=0
ieee80211n=0
he_6ghz_reg_power_mode=1
wpa=2
wpa_key_mgmt=SAE
ieee80211w=2
sae_pwe=2
```

起不来时看日志：

```sh
logread -f | grep -i hostapd
```

常见错误：

| 报错 | 原因 |
|---|---|
| `Failed to set frequency` | regdb 不允许该信道，或 phy 不支持 |
| `Invalid op_class` | `htmode` 和 `op_class` 不匹配 |
| `SAE is required` | 用了 `psk2` 而不是 `sae` |
| `nl80211: kernel reports: Channel is disabled` | regdb 没生效 |

---

## 信道速查

| PSC 信道 | 中心频率 | 建议 |
|---|---|---|
| 5 | 5975 | 靠近 UNII-5 下沿，干扰少 |
| 37 | 6135 | **首选**，多数设备默认扫这里 |
| 69 | 6295 | — |
| 101 | 6455 | 跨入 UNII-6 |
| 149 | 6695 | UNII-7 |
| 197 | 6935 | UNII-8 |

带宽 160 MHz 时，中心信道要选能被 160 覆盖的位置，用 37 最稳。

---

## 第四关：客户端（这关最容易翻车）

**AP 强开成功 ≠ 你的设备能连上。**

| 客户端 | 默认能不能扫到 6 GHz |
|---|---|
| 国行 Windows 笔记本（AX210/AX211） | ❌ 监管域是 CN，不扫 6 GHz |
| 国行 iPhone | ❌ 硬件层面就不支持 6E |
| 国行安卓旗舰 | 部分支持，但默认关闭 |
| 外版 Windows / Linux | 取决于系统监管域设置 |

### 让客户端也放开

**Linux**：
```sh
sudo iw reg set US      # 或 CA / BR / KR
sudo iw dev wlan0 scan | grep -i 6ghz
```

**Windows**：改注册表（需重启网卡驱动）
```
HKLM\SYSTEM\CurrentControlSet\Control\Class\{4d36e972-...}\00xx
  -> "CountryRegion" = 0（world roaming）
  -> 或直接把驱动区域设为 US
```
具体键值随驱动版本不同，稳妥做法是装 Linux 双系统或用外版设备验证。

**手机**：基本无解，除非是外版机型。

### 一个自检办法

在 AP 上起 6 GHz 后，用另一台**已经放开监管域的 Linux 设备**扫：

```sh
sudo iw dev wlan0 scan | grep -E "freq:|SSID:"
```

如果能扫到 6135 MHz 且 SSID 正确，说明 AP 侧完全成功，剩下就是客户端的事了。

---

## 功率与合法性提醒

`db-cn-6ghz.txt` 里给的 23 dBm 是按 LPI（低功率室内）取的保守值。改大之前注意：

- BDF 里的功率表是按 FCC/CE 校准的，和 regdb 的上限对不上时，可能起不来或掉线
- 6 GHz 在国内没有任何 Wi-Fi 频段分配（5925–6425 未开放，6425–7125 划给 IMT）
- 也就是说，**这一整套是"能跑通"，不是"合规可部署"**

技术验证没问题，但长期开着发射要自己想清楚。
