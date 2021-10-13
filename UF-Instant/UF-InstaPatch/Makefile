-include config.mk

define check_var_msg
$1 ($2) not specified.
	Please specify all required variables either in the configuration file
	or on the command line
endef
check_var = $(if $($(1)),,$(error $(check_var_msg)))

$(call check_var,GPON_SN,GPON serial number)
$(call check_var,GPON_MAC,GPON MAC address)
DISABLE_WATCHDOG ?= 1

BUILD_DIR  := build
OLT_FW_DIR := $(BUILD_DIR)/olt
ONU_FW_DIR := $(BUILD_DIR)/onu

FAKEROOT_ENV_FILE := $(BUILD_DIR)/fakeroot.env

### calculated variables
# sanitize GPON SN
GPON_SN_SERIAL := $(shell echo "$(GPON_SN)" | tr -dc [:xdigit:] | rev | cut -c-8 | rev | tr [:upper:] [:lower:])
GPON_SN_MFGR   := $(shell \
	[ `echo "$(GPON_SN)" | wc -c` -lt 16 ] \
		&& echo "$(GPON_SN)" | xxd -p -l 4 \
		|| echo "$(GPON_SN)" | tr -dc [:xdigit:] | cut -c-8 | tr [:upper:] [:lower:])
GPON_SN_RAW    := $(GPON_SN_MFGR)$(GPON_SN_SERIAL)
GPON_SN        := $(shell echo "$(GPON_SN_MFGR)" | xxd -r -p)$(GPON_SN_SERIAL)

# sanitize GPON MAC
GPON_MAC_SAFE  := $(shell echo "$(GPON_MAC)" | tr -dc [:xdigit:] | tr [:upper:] [:lower:] | sed 's/../&:/g;s/:$$//')

### functions
bin_offset_of       = `grep -PUzabo "$(2)" $(1) | cut -zd: -f1 | tr '\0' '\n'`
bin_patch_at_offset = dd of=$(1) bs=1 seek=$(2) conv=notrunc 2>/dev/null

### recipes
all: fw_patched.img
.PHONY: all

fw_patched.img: $(ONU_FW_DIR)/fw_patched.img
	cp $< $@

### extract original OLT firmware
$(OLT_FW_DIR)/fw.tar: | $(OLT_FW_DIR)/
	$(call check_var,OLT_FW_URL,OLT firmware download URL)
	wget -O $@ $(OLT_FW_URL)

$(OLT_FW_DIR)/squashfs.tmp: $(OLT_FW_DIR)/fw.tar
	tar -C $(OLT_FW_DIR) -xf $<
	touch $@

$(OLT_FW_DIR)/squashfs-root: $(OLT_FW_DIR)/squashfs.tmp
	$(RM) -r $@
	fakeroot unsquashfs -d $@ $<
	touch $@

### extract original ONU firmware
$(ONU_FW_DIR)/fw.img: | $(ONU_FW_DIR)/
$(ONU_FW_DIR)/fw.img: $(OLT_FW_DIR)/squashfs-root
	cp $</usr/share/onufw/UF-ONU.UF-INSTANT.* $@

$(ONU_FW_DIR)/fw.tar: $(ONU_FW_DIR)/fw.img
	tail -c +57 $< >$@

$(ONU_FW_DIR)/archive: | $(ONU_FW_DIR)/archive/
$(ONU_FW_DIR)/archive: $(ONU_FW_DIR)/fw.tar
	tar -C $@ -xf $<

# NOTE: fakeroot needs squashfs-root to not move between extract and repack to
# keep its environment sane
$(ONU_FW_DIR)/squashfs-root: $(ONU_FW_DIR)/archive
	$(RM) -r $@
	fakeroot -s $(FAKEROOT_ENV_FILE) unsquashfs -d $@ $</rootfs
	touch $@

### patch ONU firmware
# NOTE: fails if already patched (not idempotent)
$(ONU_FW_DIR)/squashfs-root/lib/modules/ubnthal.ko: $(ONU_FW_DIR)/squashfs-root
	printf "$(GPON_SN)\0"		| $(call bin_patch_at_offset,$@,$(call bin_offset_of,$@,%c%c%c%c%02x%02x%02x%02x$$))
	printf "$(GPON_SN_RAW)\0"	| $(call bin_patch_at_offset,$@,$(call bin_offset_of,$@,%02x%02x%02x%02x%02x%02x%02x%02x$$))
	printf "$(GPON_MAC_SAFE)\n\0"	| $(call bin_patch_at_offset,$@,$(call bin_offset_of,$@,%02x:%02x:%02x:%02x:%02x:%02x\n$$))

$(ONU_FW_DIR)/squashfs-root/etc/init.d/rcS: $(ONU_FW_DIR)/squashfs-root
ifneq ($(DISABLE_WATCHDOG),0)
	sed -Ei 's%^#?(/bin/ubnt-watchdog &)$$%#\1%' $@
endif

$(ONU_FW_DIR)/squashfs-root/etc/modinfo.txt: $(ONU_FW_DIR)/squashfs-root
	date "+Firmware patched by UF-InstaPatch @ %Y-%m-%d %H:%M:%S.%N (%Z)" >$@
.PHONY: $(ONU_FW_DIR)/squashfs-root/etc/modinfo.txt

### build patched ONU firmware
$(ONU_FW_DIR)/archive_patched: | $(ONU_FW_DIR)/archive_patched/
$(ONU_FW_DIR)/archive_patched: $(ONU_FW_DIR)/archive
	cp -au $</* $@/

$(ONU_FW_DIR)/archive_patched/rootfs: $(ONU_FW_DIR)/archive_patched
$(ONU_FW_DIR)/archive_patched/rootfs: $(ONU_FW_DIR)/squashfs-root/lib/modules/ubnthal.ko
$(ONU_FW_DIR)/archive_patched/rootfs: $(ONU_FW_DIR)/squashfs-root/etc/init.d/rcS
$(ONU_FW_DIR)/archive_patched/rootfs: | $(ONU_FW_DIR)/squashfs-root/etc/modinfo.txt
$(ONU_FW_DIR)/archive_patched/rootfs: $(ONU_FW_DIR)/squashfs-root
	$(RM) $@
	fakeroot -i $(FAKEROOT_ENV_FILE) mksquashfs $< $@ -comp lzma -always-use-fragments
	chmod 0700 $@

$(ONU_FW_DIR)/archive_patched/md5.txt: $(ONU_FW_DIR)/archive_patched/rootfs
$(ONU_FW_DIR)/archive_patched/md5.txt: $(ONU_FW_DIR)/archive
	md5sum $$(cut -d' ' -f3 $</md5.txt | sed 's#^#$(dir $@)#') | sed 's# $(dir $@)# #' >$@

$(ONU_FW_DIR)/fw_patched.tar: $(ONU_FW_DIR)/archive_patched/md5.txt
$(ONU_FW_DIR)/fw_patched.tar: $(ONU_FW_DIR)/archive_patched
	tar -cf $@ --owner=builder:1000 --group=builder:1000 $</* --xform='s#^$</##'

$(ONU_FW_DIR)/fw_patched.img: $(ONU_FW_DIR)/fw.img
$(ONU_FW_DIR)/fw_patched.img: $(ONU_FW_DIR)/fw_patched.tar
	dd if=$(word 2,$^) of=$@ bs=56 count=1 2>/dev/null
	cat $< >>$@
	printf '%08x%08x\n' `stat -c '%s' $@` 0 | xxd -r -p | $(call bin_patch_at_offset,$@,40)
	crc32 $@                                | xxd -r -p | $(call bin_patch_at_offset,$@,44)

### maintenance recipes
re-patch:
	$(RM) -r $(ONU_FW_DIR)/archive_patched $(ONU_FW_DIR)/squashfs-root
	$(MAKE) re-pack
.PHONY: re-patch

re-pack:
	$(RM) fw_patched.img
	$(RM) $(ONU_FW_DIR)/fw_patched.img $(ONU_FW_DIR)/fw_patched.tar
	$(RM) $(ONU_FW_DIR)/archive_patched/rootfs
	$(MAKE)
.PHONY: re-pack

%/:
	mkdir -p $@

clean:
	$(RM) -r $(BUILD_DIR) fw_patched.img
.PHONY: clean

# keep all intermediate files, except if they fail to generate
# NOTE: workaround to prevent Bash autocompletion breaking on prerequisiteless
# `.SECONDARY:`, see: https://github.com/scop/bash-completion/issues/53
ifeq (,$(filter npq%,$(firstword $(MAKEFLAGS))))
# not invoked for completion
.SECONDARY:
endif
.DELETE_ON_ERROR:
