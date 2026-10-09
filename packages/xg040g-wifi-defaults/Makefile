#
# XG-040G-MD 无线默认值纠正包（本地 feed）
# ---------------------------------------------------------------------------
# 放在 packages/ 下即可 —— 本目录被工作流挂成 src-link 本地 feed，
# 见 packages/README.md。CI 会在 feeds install 阶段把它编进固件。
#
# 本包只装一个 /etc/uci-defaults 脚本，在【首次启动】把固件配成
# **STA 无线转有线桥接** 的开箱可用状态，刷完不需要任何人再 ssh 进去敲命令。
#
# 为什么需要它（都是实机踩过的坑，细节见同目录 README.md）：
#   1) MT7921AU 是单 vif 卡：wifi-detect 首启默认生成 AP 接口，AP 一占住射频
#      STA 接口就永远起不来；而且 AP 启动（set key 组密钥）会把固件搞崩，
#      USB 随后掉线、只能物理重插。  => 默认关掉所有 AP 接口
#   2) country '00'/空：hostapd 报 Invalid country_code '00' 直接拒绝起 AP
#   3) band 被写成 '6g'：LuCI 判定 is_6ghz 后把加密列表砍到只剩 WPA3-SAE/OWE
#   4) 干净刷机没有 wwan 接口与 wan zone 归属，STA 拿到 IP 也出不了网
#
include $(TOPDIR)/rules.mk

PKG_NAME:=xg040g-wifi-defaults
PKG_VERSION:=1.1
PKG_RELEASE:=1
PKG_LICENSE:=MIT
PKG_MAINTAINER:=GinRyan

include $(INCLUDE_DIR)/package.mk

define Package/xg040g-wifi-defaults
  SECTION:=base
  CATEGORY:=Base system
  TITLE:=XG-040G-MD USB Wi-Fi defaults (STA bridge: AP off, band/country sane)
  DEPENDS:=+wpad-openssl
endef

define Package/xg040g-wifi-defaults/description
 First-boot uci-defaults for Nokia XG-040G-MD with a USB Wi-Fi adapter used as a
 STA bridge (wireless uplink -> wired LAN). It disables every AP interface (the
 MT7921AU is single-vif and the AP path crashes its firmware), fixes an invalid
 country code and a mis-detected 6 GHz band, and can optionally pre-configure the
 upstream SSID/encryption/key plus the wwan interface, the wan firewall zone and
 the lan->wan forwarding rule. No post-flash shell needed.
endef

define Build/Compile
endef

# INSTALL_BIN 会带上可执行位，所以不需要 git update-index --chmod=+x
define Package/xg040g-wifi-defaults/install
	$(INSTALL_DIR) $(1)/etc/uci-defaults
	$(INSTALL_BIN) ./files/99-xg040g-wifi $(1)/etc/uci-defaults/99-xg040g-wifi
endef

$(eval $(call BuildPackage,xg040g-wifi-defaults))
