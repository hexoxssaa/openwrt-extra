#
# Copyright (C) 2008-2016 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
# Based on the initial porting done by el1n

include $(TOPDIR)/rules.mk

PKG_NAME:=softethervpn
PKG_RELEASE:=1

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/SoftEtherVPN/SoftEtherVPN.git
PKG_SOURCE_DATE:=2018-12-03
PKG_SOURCE_VERSION:=863f87fac1b36bbd5c30f7827aa56f74ceaa56c1
PKG_MIRROR_HASH:=43d450d190636e08148a61f69b6b9b9487e3ec004c7de1b528609d522bdfaeb9

PKG_LICENSE:=GPL-2.0
PKG_LICENSE_FILES:=COPYING

PKG_BUILD_DEPENDS:=softethervpn-server/host perl/host

HAMCORE_SE2:=$(STAGING_DIR_HOST)/share/softethervpn/hamcore.se2

include $(INCLUDE_DIR)/nls.mk
include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/host-build.mk


# Override CC to add fake libreadline to linker search path
HOSTCC += -L./src/readline

# Add defines to turn add_history() and readline() calls into no-ops
HOSTCC += -D'add_history(x)' -D'readline(x)=\"\"'

# Execute in host build directory
HOST_MAKE_FLAGS += -C $(HOST_BUILD_DIR)

# Select 32 or 64 bit Makefile for host build depending on host architecture
HOST_MAKE_FLAGS += -f src/makefiles/linux_$(if $(shell uname -m | grep 64),64,32)bit.mak

# Prevent calling upstream configure
define Host/Configure
endef

define Host/Compile
	# Prepare fake readline headers and library
	mkdir -p $(HOST_BUILD_DIR)/src/readline
	touch $(HOST_BUILD_DIR)/src/readline/readline.h
	touch $(HOST_BUILD_DIR)/src/readline/history.h
	ar rcs $(HOST_BUILD_DIR)/src/readline/libreadline.a

	# Build hamcorebuilder using host compiler and let it generate
	# the hamcore.se2 archive file
	CC="$(HOSTCC)" $(MAKE) $(HOST_MAKE_FLAGS) \
		src/bin/BuiltHamcoreFiles/unix/hamcore.se2
endef

define Host/Install
	$(INSTALL_DIR) $(STAGING_DIR_HOST)/share/softethervpn
	$(INSTALL_DATA) $(HOST_BUILD_DIR)/src/bin/BuiltHamcoreFiles/unix/hamcore.se2 $(HAMCORE_SE2)
endef

define Build/Prepare
	$(Build/Prepare/Default)
	# remove EUCJP defaults
	$(SED) 's/"eucJP"/"utf-8"/gI' $(PKG_BUILD_DIR)/src/Mayaqua/Internat.c
endef	

# Tune CFLAGS for target
TARGET_CFLAGS += \
	$(if $(CONFIG_OPENSSL_WITH_SSL3),,-DSSL_OP_NO_SSLv3) \
	$(if $(filter mips mipsel,$(ARCH)),-minterlink-mips16)

# Select 32 or 64 bit Makefile for target build depending on 64bit config symbol
MAKE_FLAGS += \
	-f src/makefiles/linux_$(if $(CONFIG_ARCH_64BIT),64,32)bit.mak

# Map nonstandard CCFLAGS variable to standard TARGET_CFLAGS
MAKE_FLAGS += \
	CC="$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) $(TARGET_LDFLAGS)"

define Build/Configure
	# Fetch prebuilt hamcore.se2 from staging dir
	$(INSTALL_DIR) $(PKG_BUILD_DIR)/src/bin/BuiltHamcoreFiles/unix
	$(CP) $(HAMCORE_SE2) $(PKG_BUILD_DIR)/src/bin/BuiltHamcoreFiles/unix/hamcore.se2

	# Portably set hamcore.se2 modtime to one day in the future
	# to prevent rebuilding it
	perl -e 'utime(time() + 86400, time() + 86400, $$$$ARGV[0])' \
		$(PKG_BUILD_DIR)/src/bin/BuiltHamcoreFiles/unix/hamcore.se2
endef


define Package/softethervpn-server
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=VPN
  DEPENDS:=+libpthread +librt +libreadline +libopenssl +libncurses +kmod-tun +zlib $(ICONV_DEPENDS)
  TITLE:=Free Cross-platform Multi-protocol VPN server
  URL:=http://www.softether.org/
  MAINTAINER:=Federico Di Marco <fededim@gmail.com>
endef

# needs SSLv3_method for < 1.1
define Package/softethervpn-server/config
  select OPENSSL_WITH_SSL3
endef

define Package/softethervpn-server/description
SoftEther VPN ("SoftEther" means "Software Ethernet") is one of the world's most powerful and easy-to-use multi-protocol VPN software developed as an academic
project from University of Tsukuba, Japan. SoftEther VPN has strong compatibility to today's most popular VPN products among the world. It has the interoperability
with OpenVPN, L2TP, IPsec, EtherIP, L2TPv3, Cisco VPN Routers and MS-SSTP VPN Clients. SoftEther VPN is the world's only VPN software which supports SSL-VPN,
OpenVPN, L2TP, EtherIP, L2TPv3 and IPsec as a single VPN software. SoftEther VPN is not only an alternative VPN server to existing VPN products (OpenVPN,
IPsec and MS-SSTP), but has also original strong SSL-VPN protocol to penetrate any kinds of firewalls. Ultra-optimized SSL-VPN Protocol of SoftEther VPN
has very fast throughput, low latency and firewall resistance.
endef


define Package/softethervpn-server/conffiles
	/usr/libexec/softethervpn/vpn_server.config
	/usr/libexec/softethervpn/lang.config
endef

define Package/softethervpn-server/install
	$(INSTALL_DIR) $(1)/usr/libexec/softethervpn
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/bin/vpnserver/vpnserver $(1)/usr/libexec/softethervpn
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/bin/vpnserver/hamcore.se2 $(1)/usr/libexec/softethervpn
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/bin/vpncmd/vpncmd $(1)/usr/libexec/softethervpn
	$(INSTALL_BIN) files/launcher.sh $(1)/usr/libexec/softethervpn

	$(INSTALL_DATA) files/dummy $(1)/usr/libexec/softethervpn/vpn_server.config
	$(INSTALL_DATA) files/dummy $(1)/usr/libexec/softethervpn/lang.config

	$(INSTALL_DIR) $(1)/usr/bin

	$(LN) ../../usr/libexec/softethervpn/launcher.sh $(1)/usr/bin/vpncmd

	$(INSTALL_DIR) $(1)/etc
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) files/vpnserver.init $(1)/etc/init.d/softethervpnserver
endef

$(eval $(call HostBuild))
$(eval $(call BuildPackage,softethervpn-server))

