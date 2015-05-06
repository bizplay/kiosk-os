REPO := git://github.com/bizplay/kiosk-os-chroot.git
BRANCH := master
OUTPUT := /srv/www/build.webconverger.org/output

# Check if live-build is new enough
LB_VERSION:=$(shell lb --version)
MIN_LB_VERSION:=4.0.0
TOONEW_LB_VERSION:=5.0~
ifneq ($(shell dpkg --compare-versions "$(LB_VERSION)" ge "$(MIN_LB_VERSION)"; echo $$?),0)
$(error live-build is too old, needs at least version $(MIN_LB_VERSION) $(CMP))
endif
ifneq ($(shell dpkg --compare-versions "$(LB_VERSION)" lt "$(TOONEW_LB_VERSION)"; echo $$?),0)
$(error live-build is too new, needs at less than version $(TOONEW_LB_VERSION) $(CMP))
endif

VERSION=$(shell cd chroot; git describe --always)

build: TYPE=iso-hybrid
build: binary

build-hdd: TYPE=hdd
build-hdd: binary

ifneq ($(shell id -u),0)
SUDO=sudo
endif

VDI=binary.vdi
build-vbox-hdd: build-hdd
	# If there's already a vdi file, try to keep the UUID the same,
	# so you don't have to re-add the vdi file in Virtualbox.
	if [ -f "$(VDI)" ]; then \
		UUID="--uuid $$(VBoxManage showhdinfo $(VDI) | awk '$$1 == "UUID:" {print $$2}')"; \
		rm -f $(VDI); \
	fi; \
	VBoxManage convertfromraw live-image-i386 $(VDI) $$UUID

binary: clean
	# Check that the chroot has no changes
	[ -z "$$(cd chroot &&  git status --porcelain)" ] || \
	(echo "You have uncommitted changes in the chroot, bailing out now!" && false)

	test -d chroot || git clone --depth 1 -b $(BRANCH) $(REPO) chroot
	# Do a shallow clone to minimize image size. This only works
	# when using a file:// url (instead of plain path), so we use
	# ${CURDIR} to construct an absolute path). If chroot is already
	# a shallow clone, then just copy the .git directory
	if test -f chroot/.git/shallow; then \
		$(SUDO) mkdir -p binary/live; \
		$(SUDO) cp -r chroot/.git binary/live/filesystem.git; \
	else \
		$(SUDO) git clone --no-local --depth 1 --bare file://$(CURDIR)/chroot/.git binary/live/filesystem.git; \
	fi
	$(SUDO) git --git-dir=binary/live/filesystem.git remote set-url origin "$(REPO)"
	$(SUDO) git --git-dir=binary/live/filesystem.git tag initial-revision
	# Make the repository look like a non-bare one. It will still
	# not have a worktree on disk, but in a running system, the
	# filesystem.git directory will be bindmounted as /.git, making
	# the root filesystem its worktree
	$(SUDO) git --git-dir=binary/live/filesystem.git config --replace-all core.bare false
	$(SUDO) lb config -b $(TYPE) --build-with-chroot false # --chroot-filesystem none
	$(SUDO) lb binary_linux-image
	$(SUDO) lb binary_syslinux
	# Copy the template bootloader config, so the config can be
	# regenerated on upgrades (but only on an hdd image, since an
	# iso-hybrid image is not writable anyway).
	if [ "$(TYPE)" = hdd ]; then \
		$(SUDO) cp config/bootloaders/syslinux/live.cfg.in binary/boot; \
	fi
	# Build either a .iso or .img file, depending on the configured
	# image type
	$(SUDO) lb binary_iso
	$(SUDO) lb binary_hdd
	# Create an empty small partition in the img file after the one that the build
	# just created. This empty small partition can be extended when booting from
	# the image to create room for persisting user data is needed
	#
	# Create an ext filesystem on the newly created empty partition and label it
	# The ext3 filesystem is used because it will painlessly extend when the partition
	# is resized later to fit the device. The label can be used to identify this
	# partition when it is needed.
	echo "Creating extra partition to optionally persist user data"
	$(SUDO) truncate --size=+2M live-image-i386
	first_partition_end=`$(SUDO) parted --script live-image-i386 unit B print | awk '$$7 ~ /boot/ {gsub(/.$$/,"",$$3); print $$3}'`; \
	second_partition_start=$$((first_partition_end + 1)); \
	$(SUDO) parted --script live-image-i386 mkpart primary $${second_partition_start}B 100%; \
	$(SUDO) losetup -o $${second_partition_start} /dev/loop0 live-image-i386
	$(SUDO) mke2fs -L user_data -t ext2 -T default /dev/loop0
	$(SUDO) mkdir chroot/tmp/tmp_mount
	$(SUDO) mount /dev/loop0 chroot/tmp/tmp_mount
	$(SUDO) chown -R 1000 chroot/tmp/tmp_mount
	$(SUDO) umount chroot/tmp/tmp_mount
	$(SUDO) rmdir chroot/tmp/tmp_mount
	$(SUDO) losetup --detach /dev/loop0

clean:
	$(SUDO) lb clean --binary
	rm -f webc*.iso webc*.img webc*.txt

deploy: build
	mv binary.hybrid.iso webc-$(VERSION).iso
	./chroot.sh "dpkg-query -l" | grep ^ii > webc-$(VERSION).txt
	for i in webc*.iso; do mv $$i $(OUTPUT); echo "Redirect /latest.iso /$$i " > $(OUTPUT)/.htaccess ; done
	for i in webc*.txt; do mv $$i $(OUTPUT); echo "Redirect /latest.txt /$$i " >> $(OUTPUT)/.htaccess ; done
