---
layout: default
title: Introduction to the classroom environment
nav_order: 0
---

# Introduction to the classroom environment

In this exercise, you access and use the lab environment and browse the
available resources.

**Outcomes**

You will be familiar with the servers in the lab environment, and should
be able to log in and browse the available resources.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start ansible-prep

1.  Verify that you can reach all SAP servers. For each server, explore
    the memory size, disks, and number of CPUs. The SAP servers to
    review are `hana1`, `hana2`, and `nodea-noded`.

    An example follows for `hana1`:

    1.  Log in to `hana1`:

            [student@workstation ~]$ ssh hana1
            ...output omitted...

    2.  Display the memory and swap space:

            [student@localhost ~]$ free -m
                          total        used        free      shared  buff/cache   available
            Mem:          48087         314       47563          16         208       47306
            Swap:             0           0           0

    3.  Display the attached disks:

            [student@localhost ~]$ lsblk
            NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
            vda    252:0    0   200G  0 disk
            ├─vda1 252:1    0     1M  0 part
            ├─vda2 252:2    0   100M  0 part /boot/efi
            └─vda3 252:3    0 199,9G  0 part /
            vdb    252:16   0    50G  0 disk

    4.  Display the configured CPUs:

            [student@localhost ~]$ lscpu
            Architecture:        x86_64
            CPU op-mode(s):      32-bit, 64-bit
            Byte Order:          Little Endian
            CPU(s):              6
            On-line CPU(s) list: 0-5
            Thread(s) per core:  1
            Core(s) per socket:  1
            Socket(s):           6
            NUMA node(s):        1
            Vendor ID:           GenuineIntel
            CPU family:          6
            Model:               85
            Model name:          Intel(R) Xeon(R) Gold 6248 CPU @ 2.50GHz
            Stepping:            7
            CPU MHz:             2494.056
            BogoMIPS:            4988.11
            Virtualization:      VT-x
            Hypervisor vendor:   KVM
            Virtualization type: full
            L1d cache:           32K
            L1i cache:           32K
            L2 cache:            4096K
            L3 cache:            16384K
            NUMA node0 CPU(s):   0-5
            Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon rep_good nopl xtopology cpuid tsc_known_freq pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves arat umip pku ospke avx512_vnni md_clear arch_capabilities

2.  Repeat steps 1.1 to 1.4 for `hana2` and `nodea`.

**Finish**

On the `workstation` machine, use the `lab` command to complete this
exercise. This step is important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ~]$ lab finish ansible-prep
