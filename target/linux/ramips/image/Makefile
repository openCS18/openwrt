#
# Copyright (C) 2008-2011 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

DEVICE_VARS += DTS IMAGE_SIZE NETGEAR_BOARD_ID NETGEAR_HW_ID
DEVICE_VARS += BUFFALO_TAG_PLATFORM BUFFALO_TAG_VERSION BUFFALO_TAG_MINOR
DEVICE_VARS += SEAMA_SIGNATURE SEAMA_MTDBLOCK
DEVICE_VARS += SERCOMM_HWID SERCOMM_HWVER SERCOMM_SWVER
DEVICE_VARS += JCG_MAXSIZE

loadaddr-y := 0x80000000
loadaddr-$(CONFIG_TARGET_ramips_rt288x) := 0x88000000
loadaddr-$(CONFIG_TARGET_ramips_mt7621) := 0x80001000

KERNEL_LOADADDR := $(loadaddr-y)

KERNEL_DTB = kernel-bin | append-dtb | lzma
define Device/Default
  PROFILES = Default
  KERNEL := $(KERNEL_DTB) | uImage lzma
  MTK_SOC :=
  DEVICE_DTS_DIR := ../dts
  DEVICE_DTS = $$(MTK_SOC)_$(1)
  IMAGES := sysupgrade.bin
  SUPPORTED_DEVICES := $(subst _,$(comma),$(1))
  sysupgrade_bin := append-kernel | append-rootfs | pad-rootfs
  IMAGE/sysupgrade.bin := append-kernel | append-rootfs | pad-rootfs | append-metadata | check-size $$$$(IMAGE_SIZE)
endef

define Device/seama
  SEAMA_MTDBLOCK := 2
  IMAGES += factory.bin

  # 64 bytes offset:
  # - 28 bytes seama_header
  # - 36 bytes of META data (4-bytes aligned)
  IMAGE/default := append-kernel | pad-offset $$$$(BLOCKSIZE) 64 | append-rootfs
  IMAGE/sysupgrade.bin := \
	$$(IMAGE/default) | seama | pad-rootfs | append-metadata | check-size $$$$(IMAGE_SIZE)
  IMAGE/factory.bin := \
	$$(IMAGE/default) | pad-rootfs -x 64 | seama | seama-seal | check-size $$$$(IMAGE_SIZE)
  SEAMA_SIGNATURE :=
endef

define Build/jcg-header
	$(STAGING_DIR_HOST)/bin/jcgimage -v $(1) $(if $(JCG_MAXSIZE), -m $(JCG_MAXSIZE),) -u $@ -o $@.new
	mv $@.new $@
endef

define Build/trx
	$(STAGING_DIR_HOST)/bin/trx $(1) \
		-o $@ \
		-m $$(($(subst k, * 1024,$(IMAGE_SIZE)))) \
		-f $(IMAGE_KERNEL) \
		-a 4 -f $(IMAGE_ROOTFS)
endef

define Build/loader-common
	rm -rf $@.src
	$(MAKE) -C lzma-loader \
		PKG_BUILD_DIR="$@.src" \
		TARGET_DIR="$(dir $@)" LOADER_NAME="$(notdir $@)" \
		BOARD="$(BOARDNAME)" PLATFORM="$(PLATFORM)" \
		LZMA_TEXT_START=0x82000000 LOADADDR=$(KERNEL_LOADADDR) \
		$(1) compile loader.$(LOADER_TYPE)
	mv "$@.$(LOADER_TYPE)" "$@"
	rm -rf $@.src
endef

define Build/loader-kernel
	$(call Build/loader-common,LOADER_DATA="$@")
endef

define Build/relocate-kernel
	rm -rf $@.relocate
	$(CP) ../../generic/image/relocate $@.relocate
	$(MAKE) -C $@.relocate KERNEL_ADDR=$(KERNEL_LOADADDR) CROSS_COMPILE=$(TARGET_CROSS)
	( \
		dd if=$@.relocate/loader.bin bs=32 conv=sync && \
		perl -e '@s = stat("$@"); print pack("V", @s[7])' && \
		cat $@ \
	) > $@.new
	mv $@.new $@
	rm -rf $@.relocate
endef

define Build/elx-header
	$(eval hw_id=$(word 1,$(1)))
	$(eval xor_pattern=$(word 2,$(1)))
	( \
		echo -ne "\x00\x00\x00\x00\x00\x00\x00\x03" | \
			dd bs=42 count=1 conv=sync; \
		hw_id="$(hw_id)"; \
		echo -ne "\x$${hw_id:0:2}\x$${hw_id:2:2}\x$${hw_id:4:2}\x$${hw_id:6:2}" | \
			dd bs=20 count=1 conv=sync; \
		echo -ne "$$(printf '%08x' $$(stat -c%s $@) | fold -s2 | xargs -I {} echo \\x{} | tr -d '\n')" | \
			dd bs=8 count=1 conv=sync; \
		echo -ne "$$($(STAGING_DIR_HOST)/bin/mkhash md5 $@ | fold -s2 | xargs -I {} echo \\x{} | tr -d '\n')" | \
			dd bs=58 count=1 conv=sync; \
	) > $(KDIR)/tmp/$(DEVICE_NAME).header
	$(call Build/xor-image,-p $(xor_pattern) -x)
	cat $(KDIR)/tmp/$(DEVICE_NAME).header $@ > $@.new
	mv $@.new $@
	rm -rf $(KDIR)/tmp/$(DEVICE_NAME).header
endef

define Build/umedia-header
	fix-u-media-header -T 0x46 -B $(1) -i $@ -o $@.new && mv $@.new $@
endef

define Build/edimax-header
	$(STAGING_DIR_HOST)/bin/mkedimaximg -i $@ -o $@.new $(1)
	@mv $@.new $@
endef

define Build/poray-header
	$(STAGING_DIR_HOST)/bin/mkporayfw $(1) -f $@ -o $@.new
	mv $@.new $@
endef

define Build/wrg-header
	mkwrgimg -i $@ -d "/dev/mtdblock/2" -s $(1) -o $@.new
	mv $@.new $@
endef

# combine kernel and rootfs into one image
# mkdlinkfw <type> <optional extra arguments to mkdlinkfw binary>

define Build/mkdlinkfw
	-$(STAGING_DIR_HOST)/bin/mkdlinkfw \
		-k $(IMAGE_KERNEL) \
		-r $(IMAGE_ROOTFS) \
		-o $@ \
		$(if $(DLINK_IMAGE_OFFSET), -O $(DLINK_IMAGE_OFFSET)) \
		-s $(DLINK_FIRMWARE_SIZE)
endef

define Build/mkdlinkfw-factory
	-$(STAGING_DIR_HOST)/bin/mkdlinkfw \
		-m $(DLINK_ROM_ID) -f $(DLINK_FAMILY_MEMBER) \
		-F $@ \
		-o $@.new \
		$(if $(DLINK_IMAGE_OFFSET), -O $(DLINK_IMAGE_OFFSET)) \
		-s $(DLINK_FIRMWARE_SIZE)
	mv $@.new $@
endef

define Build/zyimage
	$(STAGING_DIR_HOST)/bin/zyimage $(1) $@
endef

define Build/sercom-seal
	$(STAGING_DIR_HOST)/bin/mksercommfw \
		-i $@ \
		-b $(SERCOMM_HWID) \
		-r $(SERCOMM_HWVER) \
		-v $(SERCOMM_SWVER) \
		$(1)
endef

define Build/sercom-footer
	$(call Build/sercom-seal,-f)
endef

define Build/fonfxcimage
	uimage_padhdr -i $@ -o $@.new
	mv $@.new $@
endef

ifeq ($(SUBTARGET),rt288x)
include rt288x.mk
endif

ifeq ($(SUBTARGET),rt305x)
include rt305x.mk
endif

ifeq ($(SUBTARGET),rt3883)
include rt3883.mk
endif

ifeq ($(SUBTARGET),mt7620)
include mt7620.mk
endif

ifeq ($(SUBTARGET),mt7621)
include mt7621.mk
endif

ifeq ($(SUBTARGET),mt76x8)
include mt76x8.mk
endif

$(eval $(call BuildImage))
