# Copyright Contributors to the L3AF Project.
# SPDX-License-Identifier: (GPL-2.0 OR BSD-2-Clause)

LINUX_SRC_PATH ?= /usr/src/linux
BPF_SAMPLES_PATH := $(LINUX_SRC_PATH)/samples/bpf
TOOLS_PATH := $(BPF_SAMPLES_PATH)/../../tools
L3AF_SRC_PATH := $(CURDIR)

# List of programs to build
hostprogs-y += mirroring

# Libbpf dependencies
LIBBPF = $(TOOLS_PATH)/lib/bpf/libbpf.a

mirroring-objs := mirroring.o ../../bpf_load.o


# Tell kbuild to always build the programs
always := $(hostprogs-y)
always += mirroring_ingress_kern.o
always += mirroring_egress_kern.o

KBUILD_HOSTCFLAGS += -I$(objtree)/usr/include
KBUILD_HOSTCFLAGS += -I$(srctree)/tools/lib/
KBUILD_HOSTCFLAGS += -I$(srctree)/tools/testing/selftests/bpf/
KBUILD_HOSTCFLAGS += -I$(srctree)/tools/lib/ -I$(srctree)/tools/include -I$(srctree)/tools/include/uapi
KBUILD_HOSTCFLAGS += -I$(srctree)/tools/perf

HOSTCFLAGS_bpf_load.o += -I. -I$(objtree)/usr/include -Wno-unused-variable
HOSTCFLAGS_trace_helpers.o += -I$(srctree)/tools/lib/bpf/

HOSTCFLAGS_mirroring.o +=  -I. -I$(BPF_SAMPLES_PATH) -I$(srctree)/tools/lib/bpf/ -g -LTEST/libbpf.a

KBUILD_HOSTLDLIBS               += $(LIBBPF) -lelf
HOSTLDLIBS_test_overhead        += -lrt
HOSTLDLIBS_mirroring  += -pthread -lm -lz -lpcap -lcrypto -lcurl -g -lpthread

# Allows pointing LLC/CLANG to a LLVM backend with bpf support, redefine on cmdline:
LLC ?= llc
CLANG ?= clang
LLVM_OBJCOPY ?= llvm-objcopy
BTF_PAHOLE ?= pahole

# Detect that we're cross compiling and use the cross compiler
ifdef CROSS_COMPILE
HOSTCC = $(CROSS_COMPILE)gcc
CLANG_ARCH_ARGS = -target $(ARCH)
endif

# Trick to allow make to be run from this directory
all:build tar.zip
tar.zip:
	@rm -rf l3af_traffic_mirroring
	@rm -f l3af_traffic_mirroring.tar.gz
	@rm -f l3af_traffic_mirroring.tar
	@mkdir l3af_traffic_mirroring
	@cp $(L3AF_SRC_PATH)/mirroring l3af_traffic_mirroring/
	@cp $(L3AF_SRC_PATH)/mirroring_ingress_kern.o l3af_traffic_mirroring/
	@cp $(L3AF_SRC_PATH)/mirroring_egress_kern.o l3af_traffic_mirroring/
	@tar -cvf l3af_traffic_mirroring.tar ./l3af_traffic_mirroring
	@gzip l3af_traffic_mirroring.tar

build: $(LIBBPF)
	$(MAKE) -C $(LINUX_SRC_PATH)/ $(L3AF_SRC_PATH)/ BPF_SAMPLES_PATH=$(BPF_SAMPLES_PATH)

clean:
	$(MAKE) -C $(LINUX_SRC_PATH) M=$(L3AF_SRC_PATH)/ clean
	@rm -f ../*.o
	@rm -f *~
	@rm -f l3af_traffic_mirroring.tar.gz

$(LIBBPF): FORCE
# Fix up variables inherited from Kbuild that tools/ build system won't like
	$(MAKE) -C $(dir $@) RM='rm -rf' LDFLAGS= srctree=$(BPF_SAMPLES_PATH)/../../ O=

FORCE:
# Verify LLVM compiler tools are available and bpf target is supported by llc
.PHONY: verify_cmds verify_target_bpf $(CLANG) $(LLC)

#verify_cmds: $(CLANG) $(LLC)
	@for TOOL in $^ ; do \
                if ! (which -- "$${TOOL}" > /dev/null 2>&1); then \
                        echo "*** ERROR: Cannot find LLVM tool $${TOOL}" ;\
                        exit 1; \
                else true; fi; \
        done

#verify_target_bpf: verify_cmds
	@if ! (${LLC} -march=bpf -mattr=help > /dev/null 2>&1); then \
                echo "*** ERROR: LLVM (${LLC}) does not support 'bpf' target" ;\
                echo "   NOTICE: LLVM version >= 3.7.1 required" ;\
                exit 2; \
        else true; fi

$(BPF_SAMPLES_PATH)/*.c: $(LIBBPF)
$(src)/*.c: $(LIBBPF)

$(obj)/%.o: $(src)/%.c 
	@echo "  CLANG-bpf " $@
	$(Q)$(CLANG) $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) -I$(obj) \
		-I$(srctree)/tools/testing/selftests/bpf/ \
		-D__KERNEL__ -D__BPF_TRACING__ -Wno-unused-value -Wno-pointer-sign \
		-D__TARGET_ARCH_$(ARCH) -Wno-compare-distinct-pointer-types \
		-Wno-gnu-variable-sized-type-not-at-end \
		-Wno-address-of-packed-member -Wno-tautological-compare \
		-Wno-unknown-warning-option $(CLANG_ARCH_ARGS) \
		-I$(srctree)/samples/bpf/ -include asm_goto_workaround.h \
		-O2 -emit-llvm -c $< -o -| $(LLC) -march=bpf $(LLC_FLAGS) -filetype=obj -o $@
ifeq ($(DWARF2BTF),y)
	$(BTF_PAHOLE) -J $@
endif

