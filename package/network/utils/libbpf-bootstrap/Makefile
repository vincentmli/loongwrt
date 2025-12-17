include $(TOPDIR)/rules.mk

PKG_NAME:=libbpf-bootstrap
PKG_RELEASE:=1
PKG_VERSION:=1.0.0

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.xz
PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

PKG_MAINTAINER:=Vincent Li <vincent.mc.li@gmail.com>
PKG_BUILD_DEPENDS:=bpftool/host

# === CRITICAL ADDITIONS ===
PKG_INSTALL:=1
PKG_ARCH:=$(ARCH)
PKG_LICENSE:=GPL-2.0-only
PKG_LICENSE_FILES:=LICENSE
# ==========================

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/bpf.mk
include $(INCLUDE_DIR)/nls.mk

PKG_BUILD_PARALLEL:=1

define Package/libbpf-bootstrap
  SECTION:=net
  CATEGORY:=Network
  TITLE:=libbpf-bootstrap - BPF Example Applications
  DEPENDS:=+libelf +zlib +libbpf $(BPF_DEPENDS) @KERNEL_DEBUG_INFO_BTF
  # === ADD ARCHITECTURE ===
  ARCHITECTURE:=all
endef

define Package/libbpf-bootstrap/description
  Complete BPF example applications using OpenWrt's system libbpf and bpftool.
  Built using the original libbpf-bootstrap Makefile with proper skeleton generation.
  Note: bootstrap app excluded (requires argp library).
endef

TARGET_CFLAGS += -I$(STAGING_DIR)/usr/include
TARGET_LDFLAGS += $(INTL_LDFLAGS) -L$(STAGING_DIR)/usr/lib

# Architecture mapping
ifeq ($(ARCH),x86_64)
  LIBBPF_ARCH:=x86
  VMLINUX_ARCH:=x86_64
else ifeq ($(ARCH),aarch64)
  LIBBPF_ARCH:=arm64
  VMLINUX_ARCH:=aarch64
else ifeq ($(ARCH),loongarch64)
  LIBBPF_ARCH:=loongarch
  VMLINUX_ARCH:=loongarch64
else
  LIBBPF_ARCH:=$(ARCH)
  VMLINUX_ARCH:=$(ARCH)
endif

# Define original list of apps from libbpf-bootstrap
ORIGINAL_APPS := bootstrap minimal minimal_legacy minimal_ns uprobe kprobe fentry \
                 usdt sockfilter tc ksyscall task_iter lsm profile biotop \
                 netqtop tc_syncookie tcpstates wg_handshake

# Define which apps to EXCLUDE (troublesome ones - require argp)
EXCLUDE_APPS := bootstrap sockfilter tc ksyscall task_iter lsm profile biotop netqtop tc_syncookie

# Dynamically compute BUILD_APPS by filtering out EXCLUDE_APPS
BUILD_APPS := $(filter-out $(EXCLUDE_APPS),$(ORIGINAL_APPS))

define Build/Prepare
	$(call Build/Prepare/Default)
	
	# Simple vmlinux.h copy
	if [ -f "$(PKG_BUILD_DIR)/vmlinux.h/include/$(VMLINUX_ARCH)/vmlinux.h" ]; then \
		mkdir -p "$(PKG_BUILD_DIR)/examples/c/$(LIBBPF_ARCH)"; \
		cp -f "$(PKG_BUILD_DIR)/vmlinux.h/include/$(VMLINUX_ARCH)/vmlinux.h" \
			"$(PKG_BUILD_DIR)/examples/c/$(LIBBPF_ARCH)/vmlinux.h"; \
	elif [ -f "$(PKG_BUILD_DIR)/examples/c/$(LIBBPF_ARCH)/vmlinux.h" ]; then \
		true; \
	else \
		echo "ERROR: vmlinux.h not found"; exit 1; \
	fi
	
	# Patch trace_helpers.c
	if [ -f "$(PKG_BUILD_DIR)/examples/c/trace_helpers.c" ]; then \
		$(SED) 's/__isoc23_fscanf/__isoc99_fscanf/g' "$(PKG_BUILD_DIR)/examples/c/trace_helpers.c"; \
		$(SED) 's/__isoc23_sscanf/__isoc99_sscanf/g' "$(PKG_BUILD_DIR)/examples/c/trace_helpers.c"; \
		$(SED) 's/__isoc23_strtol/__isoc99_strtol/g' "$(PKG_BUILD_DIR)/examples/c/trace_helpers.c"; \
	fi
	
	# Patch uprobe_helpers.c
	if [ -f "$(PKG_BUILD_DIR)/examples/c/uprobe_helpers.c" ]; then \
		$(SED) 's/__isoc23_sscanf/__isoc99_sscanf/g' "$(PKG_BUILD_DIR)/examples/c/uprobe_helpers.c"; \
	fi
endef

define Build/Configure
	@echo "=== Original apps: $(ORIGINAL_APPS) ==="
	@echo "=== Excluding: $(EXCLUDE_APPS) ==="
	@echo "=== Building apps: $(BUILD_APPS) ==="
	@echo "=== Architecture: $(ARCH) ==="
endef

define Build/Compile
	@echo "=== Starting Compilation ==="
	
	cd $(PKG_BUILD_DIR)/examples/c && \
	$(MAKE) all \
		ARCH=$(LIBBPF_ARCH) \
		CC="$(TARGET_CC)" \
		CLANG="$(CLANG)" \
		LLC="$(LLVM_LLC)" \
		BPFTOOL="$(STAGING_DIR_HOST)/usr/sbin/bpftool" \
		LIBBPF_SRC="" \
		BPFTOOL_SRC="" \
		LIBBPF_OBJ="$(STAGING_DIR)/usr/lib/libbpf.a" \
		VMLINUX="./$(LIBBPF_ARCH)/vmlinux.h" \
		OUTPUT="$(PKG_BUILD_DIR)/examples/c/.output" \
		APPS="$(BUILD_APPS)" \
		INCLUDES="-I$(PKG_BUILD_DIR)/examples/c/.output \
			-I$(PKG_BUILD_DIR)/examples/c/$(LIBBPF_ARCH) \
			-I$(STAGING_DIR)/usr/include \
			-I$(STAGING_DIR)/usr/include/bpf \
			-I$(PKG_BUILD_DIR)/examples/c" \
		CFLAGS="$(TARGET_CFLAGS) -D_GNU_SOURCE" \
		LDFLAGS="$(TARGET_LDFLAGS) -L$(STAGING_DIR)/usr/lib" \
		CLANG_BPF_SYS_INCLUDES=""
	
	@echo "=== Compilation Complete ==="
	@echo "Built binaries:"
	@for app in $(BUILD_APPS); do \
		if [ -f "$(PKG_BUILD_DIR)/examples/c/$$$$app" ]; then \
			ls -la "$(PKG_BUILD_DIR)/examples/c/$$$$app"; \
		else \
			echo "Missing: $$$$app"; \
		fi \
	done
endef

# === Build/Install SECTION ===
define Build/Install
	@echo "=== Build/Install Stage ==="

	$(INSTALL_DIR) $(PKG_INSTALL_DIR)/usr/bin

	# Copy all binaries
	$(foreach app,$(BUILD_APPS), \
		$(INSTALL_BIN) $(PKG_BUILD_DIR)/examples/c/$(app) $(PKG_INSTALL_DIR)/usr/bin/; \
	)

	@echo "DEBUG: Files in PKG_INSTALL_DIR:"
	ls -la $(PKG_INSTALL_DIR)/usr/bin/
endef

# === Package/install SECTION ===
define Package/libbpf-bootstrap/install
	@echo "=== Package/install Stage ==="

	$(INSTALL_DIR) $(1)/usr/bin

	# Check source exists
	@echo "DEBUG: Checking PKG_INSTALL_DIR..."
	@if [ ! -d "$(PKG_INSTALL_DIR)/usr/bin" ]; then \
		echo "ERROR: $(PKG_INSTALL_DIR)/usr/bin missing!"; \
		exit 1; \
	fi

	# Copy all files
	$(CP) $(PKG_INSTALL_DIR)/usr/bin/* $(1)/usr/bin/

	@echo "DEBUG: Files copied to .pkgdir:"
	ls -la $(1)/usr/bin/
endef

# === ADD POST-INSTALL SCRIPT (Required for APK) ===
define Package/libbpf-bootstrap/postinst
#!/bin/sh
echo "libbpf-bootstrap $(PKG_VERSION) installed successfully"
echo "Binaries available: $(BUILD_APPS)"
exit 0
endef

$(eval $(call BuildPackage,libbpf-bootstrap))
