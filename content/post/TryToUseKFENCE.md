---

date: 2022-04-21T19:03:37+08:00
draft: false

title: "使用 KUnit 体验 KFENCE"
description: " "
tags: ["Linux", "Kernel"]
categories: ["技术"]

---

> 2022 上半年，在很多 Kernel 技术社区都听到了 KFENCE 特性。Google 出的相关文章大多是介绍原理，没有看到实践案例。在翻代码的过程中，发现 KFENCE 使用了 KUnit 写了测试用例。所以希望通过运行测试用例的方式体验下 KFENCE。

## 概念

### KFENCE

[KFENCE](https://docs.kernel.org/dev-tools/kfence.html#) ，全称 Kernel Electric-Fence。在 [Linux 5.12 引入 Upstream](https://lwn.net/Articles/835542/)，是一种基于低开销采样的内存安全错误检测器。 KFENCE 检测可以检测 heap out-of-bounds access、use-after-free 和 invalid-free 等错误。

而功能与其相似的 [KASAN](https://www.kernel.org/doc/html/v4.14/dev-tools/kasan.html) ，通过编译时插桩，并检查每个内存访问，准确性更好，但会产生性能影响，并带来额外的内存开销。可以在非生产环境排查问题使用。而 KFENCE 基于采样实现，其性能开销几乎为零。设计初衷是希望在生产环境启用，两者可为互补。

### KUnit

[KUnit](https://www.kernel.org/doc/html/latest/dev-tools/kunit/index.html)，全称 Kernel unit testing framework，在 [Linux 5.5 引入 Upstream](https://lwn.net/Articles/780985/)。KUnit 为 Linux 内核中的单元测试提供了一个通用框架。

KUnit 测试是内核的一部分，使用 C 语言编写。KUnit 可以测试任何内核组件，效率也很高。它使用 Linux 内核源码脚本 `tools/testing/kunit/kunit.py`，可以在 QEMU 或 UML 下运行 KUnit 测试，并提供可读性高的结果。 

## 过程

开发环境：Arch Linux，Source Code：Linux 5.17.3，GCC：gcc version 11.2.0

### 初始化 KUnit

KUnit 与 Linux 内核具有相同的依赖关系。所以只要你能构建内核，你就可以运行 KUnit。

在内核代码目录，可以通过 `./tools/testing/kunit/kunit.py config` 生成 KUnit 的默认配置。

```
# ./tools/testing/kunit/kunit.py config

[13:43:28] Configuring KUnit Kernel ...
Generating .config ...
Populating config with:
$ make ARCH=um olddefconfig O=.kunit
[13:43:32] Elapsed time: 4.157s
```

配置成功后，KUnit 会在代码目录下生成`.kunit` 的新目录，单独存放与 KUnit 相关的文件。

在不修改默认配置的前提下，所有与 KUnit 相关的配置文件、过程文件（例如 KUnit 编译的 *.o）和目标文件，都会存在在该目录下。**并不会影响内核的配置和编译文件**。

下面展示了一个已经**使用 KUnit 完成测试后**，`.kunit` 下的全部内容。

```
# ls -a .kunit

.                      linux                      test.log
..                     Makefile                   .tmp_System.map
arch                   .missing-syscalls.d        .tmp_vmlinux.kallsyms1
block                  mm                         .tmp_vmlinux.kallsyms1.o
certs                  modules.builtin            .tmp_vmlinux.kallsyms1.S
.config                modules.builtin.modinfo    .tmp_vmlinux.kallsyms2
.config.old            modules-only.symvers       .tmp_vmlinux.kallsyms2.o
crypto                 .modules-only.symvers.cmd  .tmp_vmlinux.kallsyms2.S
drivers                modules.order              tools
fs                     .modules.order.cmd         usr
.gitignore             Module.symvers             .version
include                .Module.symvers.cmd        virt
init                   net                        vmlinux
ipc                    scripts                    .vmlinux.cmd
kernel                 security                   vmlinux.o
.kunitconfig           sound                      vmlinux.symvers
last_used_kunitconfig  source
lib                    System.map
```

### KUnit 配置文件

在 `.kunit` 下，存在两个我们需要关注的文件：`.kunit/.config` 和 `.kunit/.kunitconfig`。

`.kunit/.config` 与内核源码下的 `.config` 功能相同，展示所有已经选中的配置。在 KUnit中，需要使用 `.kunit/.kunitconfig` 间接生成`.kunit/.config`。

默认的 `.kunit/.kunitconfig` 拷贝自 `tools/testing/kunit/configs/default.config`，内容非常简单：

```
CONFIG_KUNIT=y
CONFIG_KUNIT_EXAMPLE_TEST=y
CONFIG_KUNIT_ALL_TESTS=y
```

### 运行 KUnit

完成配置后，就可以执行 `./tools/testing/kunit/kunit.py run` 运行 KUnit 测试用例了。

需要等待一点编译的时间。测试结果如下所示：

```
# ./tools/testing/kunit/kunit.py run

[14:09:26] Configuring KUnit Kernel ...
[14:09:26] Building KUnit Kernel ...
Populating config with:
$ make ARCH=um olddefconfig O=.kunit
Building with:
$ make ARCH=um --jobs=4 O=.kunit
[14:11:00] Starting KUnit Kernel (1/1)...
[14:11:00] ============================================================
[14:11:02] =============== time_test_cases (1 subtest) ================
[14:11:02] [PASSED] time64_to_tm_test_date_range
[14:11:02] ================= [PASSED] time_test_cases =================
[14:11:02] ================ sysctl_test (10 subtests) =================
[14:11:02] [PASSED] sysctl_test_api_dointvec_null_tbl_data
[14:11:02] [PASSED] sysctl_test_api_dointvec_table_maxlen_unset
[14:11:02] [PASSED] sysctl_test_api_dointvec_table_len_is_zero
[14:11:02] [PASSED] sysctl_test_api_dointvec_table_read_but_position_set
[14:11:02] [PASSED] sysctl_test_dointvec_read_happy_single_positive
[14:11:02] [PASSED] sysctl_test_dointvec_read_happy_single_negative
[14:11:02] [PASSED] sysctl_test_dointvec_write_happy_single_positive
[14:11:02] [PASSED] sysctl_test_dointvec_write_happy_single_negative
[14:11:02] [PASSED] sysctl_test_api_dointvec_write_single_less_int_min
[14:11:02] [PASSED] sysctl_test_api_dointvec_write_single_greater_int_max
[14:11:02] =================== [PASSED] sysctl_test ===================
[14:11:02] ==================== hash (2 subtests) =====================
[14:11:02] [PASSED] test_string_or
[14:11:02] [PASSED] test_hash_or
[14:11:02] ====================== [PASSED] hash =======================
[14:11:02] ================== list_sort (1 subtest) ===================
[14:11:02] [PASSED] list_sort_test
[14:11:02] ==================== [PASSED] list_sort ====================
[14:11:02] =================== lib_sort (1 subtest) ===================
[14:11:02] [PASSED] test_sort
[14:11:02] ==================== [PASSED] lib_sort =====================
[14:11:02] ============= kunit_executor_test (5 subtests) =============
[14:11:02] [PASSED] parse_filter_test
[14:11:02] [PASSED] filter_subsuite_test
[14:11:02] [PASSED] filter_subsuite_test_glob_test
[14:11:02] [PASSED] filter_subsuite_to_empty_test
[14:11:02] [PASSED] filter_suites_test
[14:11:02] =============== [PASSED] kunit_executor_test ===============
[14:11:02] ============ kunit-try-catch-test (2 subtests) =============
[14:11:02] [PASSED] kunit_test_try_catch_successful_try_no_catch
[14:11:02] [PASSED] kunit_test_try_catch_unsuccessful_try_does_catch
[14:11:02] ============== [PASSED] kunit-try-catch-test ===============
[14:11:02] ============= kunit-resource-test (7 subtests) =============
[14:11:02] [PASSED] kunit_resource_test_init_resources
[14:11:02] [PASSED] kunit_resource_test_alloc_resource
[14:11:02] [PASSED] kunit_resource_test_destroy_resource
[14:11:02] [PASSED] kunit_resource_test_cleanup_resources
[14:11:02] [PASSED] kunit_resource_test_proper_free_ordering
[14:11:02] [PASSED] kunit_resource_test_static
[14:11:02] [PASSED] kunit_resource_test_named
[14:11:02] =============== [PASSED] kunit-resource-test ===============
[14:11:02] ================ kunit-log-test (1 subtest) ================
[14:11:02] [PASSED] kunit_log_test
[14:11:02] ================= [PASSED] kunit-log-test ==================
[14:11:02] ================ kunit_status (2 subtests) =================
[14:11:02] [PASSED] kunit_status_set_failure_test
[14:11:02] [PASSED] kunit_status_mark_skipped_test
[14:11:02] ================== [PASSED] kunit_status ===================
[14:11:02] ============= string-stream-test (3 subtests) ==============
[14:11:02] [PASSED] string_stream_test_empty_on_creation
[14:11:02] [PASSED] string_stream_test_not_empty_after_add
[14:11:02] [PASSED] string_stream_test_get_string
[14:11:02] =============== [PASSED] string-stream-test ================
[14:11:02] =================== example (3 subtests) ===================
[14:11:02] [PASSED] example_simple_test
[14:11:02] [SKIPPED] example_skip_test
[14:11:02] [SKIPPED] example_mark_skipped_test
[14:11:02] ===================== [PASSED] example =====================
[14:11:02] ============== list-kunit-test (36 subtests) ===============
[14:11:02] [PASSED] list_test_list_init
[14:11:02] [PASSED] list_test_list_add
[14:11:02] [PASSED] list_test_list_add_tail
[14:11:02] [PASSED] list_test_list_del
[14:11:02] [PASSED] list_test_list_replace
[14:11:02] [PASSED] list_test_list_replace_init
[14:11:02] [PASSED] list_test_list_swap
[14:11:02] [PASSED] list_test_list_del_init
[14:11:02] [PASSED] list_test_list_move
[14:11:02] [PASSED] list_test_list_move_tail
[14:11:02] [PASSED] list_test_list_bulk_move_tail
[14:11:02] [PASSED] list_test_list_is_first
[14:11:02] [PASSED] list_test_list_is_last
[14:11:02] [PASSED] list_test_list_empty
[14:11:02] [PASSED] list_test_list_empty_careful
[14:11:02] [PASSED] list_test_list_rotate_left
[14:11:02] [PASSED] list_test_list_rotate_to_front
[14:11:02] [PASSED] list_test_list_is_singular
[14:11:02] [PASSED] list_test_list_cut_position
[14:11:02] [PASSED] list_test_list_cut_before
[14:11:02] [PASSED] list_test_list_splice
[14:11:02] [PASSED] list_test_list_splice_tail
[14:11:02] [PASSED] list_test_list_splice_init
[14:11:02] [PASSED] list_test_list_splice_tail_init
[14:11:02] [PASSED] list_test_list_entry
[14:11:02] [PASSED] list_test_list_first_entry
[14:11:02] [PASSED] list_test_list_last_entry
[14:11:02] [PASSED] list_test_list_first_entry_or_null
[14:11:02] [PASSED] list_test_list_next_entry
[14:11:02] [PASSED] list_test_list_prev_entry
[14:11:02] [PASSED] list_test_list_for_each
[14:11:02] [PASSED] list_test_list_for_each_prev
[14:11:02] [PASSED] list_test_list_for_each_safe
[14:11:02] [PASSED] list_test_list_for_each_prev_safe
[14:11:02] [PASSED] list_test_list_for_each_entry
[14:11:02] [PASSED] list_test_list_for_each_entry_reverse
[14:11:02] ================= [PASSED] list-kunit-test =================
[14:11:02] ================== slub_test (5 subtests) ==================
[14:11:02] [PASSED] test_clobber_zone
[14:11:02] [PASSED] test_next_pointer
[14:11:02] [PASSED] test_first_word
[14:11:02] [PASSED] test_clobber_50th_byte
[14:11:02] [PASSED] test_clobber_redzone_free
[14:11:02] ==================== [PASSED] slub_test ====================
[14:11:02] =================== memcpy (3 subtests) ====================
[14:11:02] [PASSED] memset_test
[14:11:02] [PASSED] memcpy_test
[14:11:02] [PASSED] memmove_test
[14:11:02] ===================== [PASSED] memcpy ======================
[14:11:02] =============== qos-kunit-test (3 subtests) ================
[14:11:02] [PASSED] freq_qos_test_min
[14:11:02] [PASSED] freq_qos_test_maxdef
[14:11:02] [PASSED] freq_qos_test_readd
[14:11:02] ================= [PASSED] qos-kunit-test ==================
[14:11:02] =============== property-entry (7 subtests) ================
[14:11:02] [PASSED] pe_test_uints
[14:11:02] [PASSED] pe_test_uint_arrays
[14:11:02] [PASSED] pe_test_strings
[14:11:02] [PASSED] pe_test_bool
[14:11:02] [PASSED] pe_test_move_inline_u8
[14:11:02] [PASSED] pe_test_move_inline_str
[14:11:02] [PASSED] pe_test_reference
[14:11:02] ================= [PASSED] property-entry ==================
[14:11:02] ============================================================
[14:11:02] Testing complete. Passed: 90, Failed: 0, Crashed: 0, Skipped: 2, Errors: 0
[14:11:02] Elapsed time: 96.501s total, 0.001s configuring, 94.280s building, 2.219s running
```

可以看到所有的测试用例都通过了。

### 添加 KFENCE 到 KUnit 配置文件中

虽然我们在配置文件中选择了`CONFIG_KUNIT_ALL_TESTS=y`，但在上述测试结果中并未发现 KFENCE 的身影。

通过查看 `.kunit/.config `文件的内容，我们并没有找到 `CONFIG_KFENCE` 的配置项，说明默认配置没有打开 KFENCE，所以 KFENCE 的测试用例也没有被执行。

在内核源码的 Documents 中可以找到开启 KFENCE 需要的内核选项。因为我当前使用的 Arch Linux，内核版本也是 5.17.3，默认打开了 KFENCE，直接通过当前的配置文件，看看 KFENCE 的配置：

```
# zcat /proc/config.gz | grep -i kfence

CONFIG_HAVE_ARCH_KFENCE=y
CONFIG_KFENCE=y
CONFIG_KFENCE_SAMPLE_INTERVAL=100
CONFIG_KFENCE_NUM_OBJECTS=255
CONFIG_KFENCE_STRESS_TEST_FAULTS=0
```

就把这些内容和 `CONFIG_KFENCE_KUNIT_TEST` 一起添加到 `.kunit/.kunitconfig`中。为了便于查看，把 `CONFIG_KUNIT_ALL_TESTS` 删除，只保留 KUnit 测试用例和示例：

```
# cat .kunit/.kunitconfig

# kfence
CONFIG_HAVE_ARCH_KFENCE=y
CONFIG_KFENCE=y
CONFIG_KFENCE_SAMPLE_INTERVAL=100
CONFIG_KFENCE_NUM_OBJECTS=255
CONFIG_KFENCE_STRESS_TEST_FAULTS=0

# kunit
CONFIG_KUNIT=y
CONFIG_KUNIT_EXAMPLE_TEST=y
CONFIG_KFENCE_KUNIT_TEST=y
```

再使用 `./tools/testing/kunit/kunit.py config` 生成 KUnit 的配置，发现出错了：

```
# ./tools/testing/kunit/kunit.py config

[14:49:37] Configuring KUnit Kernel ...
Regenerating .config ...
Populating config with:
$ make ARCH=um olddefconfig O=.kunit
ERROR:root:Not all Kconfig options selected in kunitconfig were in the generated .config.
This is probably due to unsatisfied dependencies.
Missing: CONFIG_KFENCE=y, CONFIG_HAVE_ARCH_KFENCE=y, CONFIG_KFENCE_KUNIT_TEST=y, CONFIG_KFENCE_SAMPLE_INTERVAL=100, CONFIG_KFENCE_NUM_OBJECTS=255, CONFIG_KFENCE_STRESS_TEST_FAULTS=0
Note: many Kconfig options aren't available on UML. You can try running on a different architecture with something like "--arch=x86_64".
[14:49:38] Elapsed time: 1.069s
```

错误信息描述的比较清楚：

1. 添加 KFENCE 相关配置缺少依赖，
2. 添加的配置不支持 UML，需要指定体系结构。

添加 `--arch=x86_64` 再尝试下：

```
# ./tools/testing/kunit/kunit.py config --arch=x86_64

[15:47:42] Configuring KUnit Kernel ...
Regenerating .config ...
Populating config with:
$ make ARCH=x86_64 olddefconfig O=.kunit
ERROR:root:Not all Kconfig options selected in kunitconfig were in the generated .config.
This is probably due to unsatisfied dependencies.
Missing: CONFIG_KFENCE_KUNIT_TEST=y
[15:47:43] Elapsed time: 1.191s
```

### 排查依赖

通过 `make menuconfig O=.kunit` 发现 `CONFIG_KFENCE_KUNIT_TEST` 的依赖 `CONFIG_TRACEPOINTS` 没有被选择，补齐依赖后配置就正常了。

完整的`.kunit/.kunitconfig` 如下：

```
# kfence
CONFIG_HAVE_ARCH_KFENCE=y
CONFIG_KFENCE=y
CONFIG_KFENCE_SAMPLE_INTERVAL=100
CONFIG_KFENCE_NUM_OBJECTS=255
CONFIG_KFENCE_STRESS_TEST_FAULTS=0
CONFIG_KFENCE_KUNIT_TEST=y

# tracepoints
CONFIG_FTRACE=y
CONFIG_TRACEPOINTS=y

# kunit
CONFIG_KUNIT=y
CONFIG_KUNIT_EXAMPLE_TEST=y
```

运行 KUnit 测试用例也需要添加 `--arch=x86_64` ：

```
# ./tools/testing/kunit/kunit.py run --arch=x86_64

[17:42:19] Configuring KUnit Kernel ...
[17:42:19] Building KUnit Kernel ...
Populating config with:
$ make ARCH=x86_64 olddefconfig O=.kunit
Building with:
$ make ARCH=x86_64 --jobs=4 O=.kunit
[17:45:28] Starting KUnit Kernel (1/1)...
[17:45:28] ============================================================
Running tests with:
$ qemu-system-x86_64 -nodefaults -m 1024 -kernel .kunit/arch/x86/boot/bzImage -append 'mem=1G console=tty kunit_shutdown=halt console=ttyS0 kunit_shutdown=reboot' -no-reboot -nographic -serial stdio
[17:46:03] =================== example (3 subtests) ===================
[17:46:03] [PASSED] example_simple_test
[17:46:03] [SKIPPED] example_skip_test
[17:46:03] [SKIPPED] example_mark_skipped_test
[17:46:03] [ERROR] Test example: Expected test number 1 but found 2
[17:46:03] ===================== [PASSED] example =====================
[17:46:03] ============================================================
[17:46:03] Testing complete. Passed: 1, Failed: 0, Crashed: 0, Skipped: 2, Errors: 1
[17:46:03] Elapsed time: 224.249s total, 0.002s configuring, 189.405s building, 34.825s running
```

执行成功，但我们希望的 KFENCE 测试用例也没有出现在结果中。

### KFENCE 测试用例没有执行？

通过上述结果，可以看到在指定 `--arch=x86_64` 后，变为使用 QEMU 进行测试了。

日志输出中有完整的 QEMU 命令。直接执行命令，通过虚拟机日志看看问题：

```
# qemu-system-x86_64 -nodefaults -m 1024 -kernel .kunit/arch/x86/boot/bzImage -append 'mem=1G console=tty kunit_shutdown=halt console=ttyS0 kunit_shutdown=reboot' -no-reboot -nographic -serial stdio

# 这里省略了无关 Log
==================================================================
BUG: KFENCE: out-of-bounds read in test_out_of_bounds_read+0x85/0x1b1

Out-of-bounds read at 0x(____ptrval____) (1B left of kfence-#3):
 test_out_of_bounds_read+0x85/0x1b1
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

kfence-#3: 0x(____ptrval____)-0x(____ptrval____), size=32, cache=kmalloc-32

allocated by task 20 on cpu 0 at 0.381983s:
 test_alloc+0xf3/0x36e
 test_out_of_bounds_read+0x76/0x1b1
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

CPU: 0 PID: 20 Comm: kunit_try_catch Not tainted 5.17.3-dirty #3
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS ArchLinux 1.16.0-1 04/01/2014
==================================================================

# 这里省略了部分 Log

==================================================================
    ok 24 - test_krealloc
    # test_memcache_alloc_bulk: setup_test_cache: size=32, ctor=0x0
    ok 25 - test_memcache_alloc_bulk
# kfence: pass:23 fail:0 skip:2 total:25
# Totals: pass:23 fail:0 skip:2 total:25
ok 1 - kfence
```

通过日志，看到 KFENCE 已经打印信息了。上述最后3行，也说明 KFENCE 的测试用例执行成功且通过。

所以 KFENCE 测试用例已经被执行，而且符合预期。

## 内核错误解读

###  invalid free

看下这段内核报错日志：

```
==================================================================
BUG: KFENCE: invalid free in test_double_free+0xa4/0x118

Invalid free of 0x(____ptrval____) (in kfence-#18):
 test_double_free+0xa4/0x118
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

kfence-#18: 0x(____ptrval____)-0x(____ptrval____), size=32, cache=kmalloc-32

allocated by task 27 on cpu 0 at 1.969091s:
 test_alloc+0xf3/0x36e
 test_double_free+0x60/0x118
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

freed by task 27 on cpu 0 at 1.969159s:
 test_double_free+0x82/0x118
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

CPU: 0 PID: 27 Comm: kunit_try_catch Tainted: G    B             5.17.3-dirty #3
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS ArchLinux 1.16.0-1 04/01/2014
==================================================================
    ok 7 - test_double_free
    # test_double_free-memcache: setup_test_cache: size=32, ctor=0x0
    # test_double_free-memcache: test_alloc: size=32, gfp=cc0, policy=any, cache=1
```

第一行日志就清晰说明了问题的原因：

```
BUG: KFENCE: invalid free in test_double_free+0xa4/0x118
```

既然日志中指明了出现问题的函数名和地址，我们就尝试找到这段代码。

首先使用 `nm` 命令找到 `test_double_free` 函数所在的地址：

```
# nm .kunit/vmlinux | grep test_double_free

ffffffff812d3f27 t test_double_free
```

看到问题函数 `test_double_free` 位于代码区，地址是 `ffffffff812d3f27`。

`ffffffff812d3f27` + `0xa4  `= `ffffffff812d3fcb`，使用 `addr2line` 查看对应的代码行：

```
# addr2line -e .kunit/vmlinux ffffffff812d3fcb

/home/leesheen/Workdir/linux-kernel/.kunit/../mm/kfence/kfence_test.c:397
```

定位到问题代码出现在 `mm/kfence/kfence_test.c` 第 397 的**上一行**。

```
 385   │ static void test_double_free(struct kunit *test)
 386   │ {
 387   │     const size_t size = 32;
 388   │     struct expect_report expect = {
 389   │         .type = KFENCE_ERROR_INVALID_FREE,
 390   │         .fn = test_double_free,
 391   │     };
 392   │
 393   │     setup_test_cache(test, size, 0, NULL);
 394   │     expect.addr = test_alloc(test, size, GFP_KERNEL, ALLOCATE_ANY);
 395   │     test_free(expect.addr);
 396   │     test_free(expect.addr); /* Double-free. */
 397   │     KUNIT_EXPECT_TRUE(test, report_matches(&expect));
 398   │ }
```

定位到的就是 KFENCE 的测试用例，结果符合预期。

上述就是 double free 的 KUnit 代码，逻辑非常简单，就是对一个地址做两次 `free` 操作。

下面也按照问题类型，摘录了 `out-of-bounds`、 `use-after-free `和  `memory corruption` 错误日志的例子。解读的方法与上述相似，就不展开了。

### out-of-bounds

```
==================================================================
BUG: KFENCE: out-of-bounds read in test_out_of_bounds_read+0x11e/0x1b1

Out-of-bounds read at 0x(____ptrval____) (32B right of kfence-#6):
 test_out_of_bounds_read+0x11e/0x1b1
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

kfence-#6: 0x(____ptrval____)-0x(____ptrval____), size=32, cache=kmalloc-32

allocated by task 20 on cpu 0 at 0.697802s:
 test_alloc+0xf3/0x36e
 test_out_of_bounds_read+0x110/0x1b1
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

CPU: 0 PID: 20 Comm: kunit_try_catch Tainted: G    B             5.17.3-dirty #3
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS ArchLinux 1.16.0-1 04/01/2014
==================================================================
    ok 1 - test_out_of_bounds_read
    # test_out_of_bounds_read-memcache: setup_test_cache: size=32, ctor=0x0
    # test_out_of_bounds_read-memcache: test_alloc: size=32, gfp=cc0, policy=left, cache=1
```

### use-after-free

```
==================================================================
BUG: KFENCE: use-after-free read in test_use_after_free_read+0x8a/0xfc

Use-after-free read at 0x(____ptrval____) (in kfence-#16):
 test_use_after_free_read+0x8a/0xfc
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

kfence-#16: 0x(____ptrval____)-0x(____ptrval____), size=32, cache=kmalloc-32

allocated by task 25 on cpu 0 at 1.753356s:
 test_alloc+0xf3/0x36e
 test_use_after_free_read+0x60/0xfc
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

freed by task 25 on cpu 0 at 1.753440s:
 test_use_after_free_read+0x82/0xfc
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

CPU: 0 PID: 25 Comm: kunit_try_catch Tainted: G    B             5.17.3-dirty #3
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS ArchLinux 1.16.0-1 04/01/2014
==================================================================
    ok 5 - test_use_after_free_read
    # test_use_after_free_read-memcache: setup_test_cache: size=32, ctor=0x0
    # test_use_after_free_read-memcache: test_alloc: size=32, gfp=cc0, policy=any, cache=1
```

### memory corruption

```
==================================================================
BUG: KFENCE: memory corruption in test_corruption+0x126/0x19a

Corrupted memory at 0x(____ptrval____) [ ! ] (in kfence-#23):
 test_corruption+0x126/0x19a
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

kfence-#23: 0x(____ptrval____)-0x(____ptrval____), size=32, cache=kmalloc-32

allocated by task 31 on cpu 0 at 2.508569s:
 test_alloc+0xf3/0x36e
 test_corruption+0xfc/0x19a
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

freed by task 31 on cpu 0 at 2.508631s:
 test_corruption+0x126/0x19a
 kunit_try_run_case+0x4b/0x70
 kunit_generic_run_threadfn_adapter+0x11/0x20
 kthread+0xad/0xe0
 ret_from_fork+0x22/0x30

CPU: 0 PID: 31 Comm: kunit_try_catch Tainted: G    B             5.17.3-dirty #3
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS ArchLinux 1.16.0-1 04/01/2014
==================================================================
    ok 11 - test_corruption
    # test_corruption-memcache: setup_test_cache: size=32, ctor=0x0
    # test_corruption-memcache: test_alloc: size=32, gfp=cc0, policy=left, cache=1
```

## 总结

经过上述分析，可以看到当打开 KFENCE 后，可以更直接地发现内存相关问题。

是否直接在线上环境使用？会不会导致性能和稳定性问题？可能还需要一些时间检验。
