CFLAGS    := -ffreestanding
LDFLAGS   := -m elf_i386 -z nodefaultlib
# EFLAGS  := ./libdrivers.a ./libmm.a ./libs5fs.a 
EFLAGS	  := ./libdrivers.a ./libs5fs.a 
# XXX should have --omagic?

include ../Global.mk

CFLAGS    += -D__KERNEL__ -DCS402TESTS=${CS402TESTS} -DGDBWAIT=${GDBWAIT}

###

HEAD      := $(wildcard include/*/*.h include/*/*/*.h)
#SRCDIR    := main boot util drivers/disk drivers/tty drivers mm proc fs/ramfs fs/s5fs fs vm api test test/kshell entry test/vfstest
# libdrivers.a: drivers/disk drivers/tty drivers 
#         drivers/memdevs.c:        NOT_YET_IMPLEMENTED("VM: zero_mmap");
# libmm.a: mm
#         mm/pframe.c:        NOT_YET_IMPLEMENTED("S5FS: pframe_get");
#         mm/pframe.c:        NOT_YET_IMPLEMENTED("S5FS: pframe_pin");
#         mm/pframe.c:        NOT_YET_IMPLEMENTED("S5FS: pframe_unpin");
# libs5fs.a: fs/s5fs
#         fs/s5fs/s5fs.c:        NOT_YET_IMPLEMENTED("VM: s5fs_mmap");
SRCDIR    := main boot util mm proc fs/ramfs fs vm api test test/kshell entry test/vfstest
#LIBDIR    := mm drivers/disk drivers/tty drivers fs/s5fs
SRC       := $(foreach dr, $(SRCDIR), $(wildcard $(dr)/*.[cS]))
OBJS      := $(addsuffix .o,$(basename $(SRC)))
#LIB_OBJS  := $(foreach dr, $(LIBDIR), $(wildcard $(dr)/*.o))
SCRIPTS   := $(foreach dr, $(SRCDIR), $(abspath $(wildcard $(dr)/*.gdb $(dr)/*.py)))

ifneq "$(shell which mkisofs 2>/dev/null)" ""
    MAKEISO := mkisofs
else
    MAKEISO := genisoimage
endif

SYMBOLS   := weenix.dbg
BSYMBOLS  := symbols.dbg
KERNEL    := kernel.bin
IMAGE     := weenix.img
ISO_IMAGE := weenix.iso
GDBCOMM   := gdb-commands

.PHONY: all cscope clean

all: $(BSYMBOLS) $(ISO_IMAGE) $(GDBCOMM)

$(SYMBOLS): $(OBJS) $(LIB_OBJS)
	@ echo "  Linking for \"kernel/$@\"..."
	@ $(LD) $(LDFLAGS) -T link.ld $(filter-out entry/entry.o,$^) -o $@ $(EFLAGS) # entry.o included from link.ld

$(BSYMBOLS): $(SYMBOLS)
	@ echo "  Generating kernel symbols list..."
	@ readelf -s $(SYMBOLS) | grep -Ev 'SECTION|UND|FILE|Num:|Symbol|^$$' | awk '{printf "0x%s %s\n", $$2, $$8}' > $@

$(KERNEL): $(SYMBOLS)
	@ echo "  Setting addresses for kernel stage 1 and stage 2..."
	@ objcopy -j .stage1 --set-section-flags .stage1=load --change-section-address .stage1=0 -O binary $< /tmp/temptemp
	@ cat /tmp/temptemp > kernel.bin
	@ objcopy -j .stage2 --set-section-flags .stage2=load --change-section-address .stage2=0 -O binary $< /tmp/temptemp
	@ cat /tmp/temptemp >> kernel.bin
	@ objcopy -O binary $< /tmp/temptemp
	@ cat /tmp/temptemp >> kernel.bin
	@ rm /tmp/temptemp # XXX find better solution than this

$(IMAGE): $(KERNEL)
	@ echo "  Creating floppy disk image from kernel binary..."
	@ dd if=/dev/zero of=$@ bs=1024 count=1440 2> /dev/null
	@ dd if=$< of=$@ conv=notrunc 2> /dev/null

$(ISO_IMAGE): $(IMAGE)
	@ echo "  Creating \"kernel/$@\" from floppy disk image..."
	@ $(MAKEISO) -quiet -input-charset utf-8 -o $@ -b $< $<

$(GDBCOMM): $(SCRIPTS)
	@ echo "  Creating gdb command list..."
	@ $(foreach script, $(SCRIPTS), echo $(script) >> $(dir $(script))$(shell basename $(dir $(script))).gdbcomm; )
	@ cat */*.gdbcomm > $@

%.o: %.c 
	@ echo "  Compiling \"kernel/$<\"..."
	@ $(CC) -c $(CFLAGS) $< -o $@

%.o: %.S
	@ echo "  Compiling \"kernel/$<\"..."
	@ $(CC) -c $(ASFLAGS) $(CFLAGS) $< -o $@

cscope: $(HEAD) $(SRC)
	@ echo "  Updating cscope symbol cross-reference..."
	@ echo $(HEAD) $(SRC) > cscope.files
	@ $(CSCOPE) -k -b -q -v > /dev/null

FILTER=`echo "DRIVERS $(DRIVERS)\nVFS $(VFS)\nS5FS $(S5FS)\nVM $(VM)" | grep 1 | cut -f1 -d" " | tr "\n" "|"`PROCS
nyi:
	@ echo "  Not yet implemented:"
	@ echo
	@ find . -name \*.c -printf "%P\n" \
| xargs grep -Hn "NOT_YET_IMPLEMENTED" \
| sed -e 's/^\(.*:.*\):.*\"\(.*\): \(.*\)\".*/\2 \1 \3/' \
| grep -E "^($(FILTER))" \
| awk '{printf("%25s %30s() %8s\n", $$2, $$3, $$1)}'

clean:
	@ rm -f $(OBJS) $(SYMBOLS) $(BSYMBOLS) $(KERNEL) $(IMAGE) $(ISO_IMAGE) $(GDBCOMM) */*.gdbcomm cscope*.out cscope.files
