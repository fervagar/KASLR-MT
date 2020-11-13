
# KASLR-MT Linux kernel patch



## Description

This kernel patch is a proof of concept implementation of KASLR-MT for Linux v5.0.6. This feature allows the kernel memory layout to be deterministically produced from a key passed through a command line argument. It is intended to be beneficial in multi-tenant systems where several tenants are owners of one or more groups of virtual machines, and all of them share the resources of a single physical machine by the use of virtualization technologies. This option allows guest kernels to randomize the addresses of their memory regions consistently among multiple VMs belonging to the same tenant. Thus, it is compatible with memory deduplication and increases the benefits of memory sharing in the host machine while keeping the security provided by KASLR in the guest kernels.

For more details, please refer to this [research article](https://www.sciencedirect.com/science/article/abs/pii/S0743731519304095). It provides a comprehensive description and analysis of the problem and a detailed discussion of the proposed solution.

This implementation is a KASLR-MT proof of concept that enables the randomization of different Linux kernel memory regions using two different strategies:

* **Per-VM**: Every time a guest reboots, the kernel memory region will have a different base address and it will not be shared across virtual machines belonging to same or other tenants (same as with standard KASLR).
* **Per-Tenant**: The kernel memory region base address is deterministically derived from a key passed through the kernel cmdline. Although it is deterministic, this address is still random for an external attacker. This allows guest kernels belonging to a same tenant to have the same memory base address (random) for a particular kernel memory region.

The random key can be specified through the `kaslr_mt=` argument. The format of this argument in this implementation is as follows:

```
kaslr_mt=RandomKeyString-AddressBitmap
```
where:

* **RandomKeyString:** String of common printable ascii characters representing the key from which the memory addresses of the regions randomized *per-tenant* will be determined.

* **AddressBitmap:** Bitmap used for testing the feature. This bitmap can be used to specify how to randomize the base address of each kernel memory region, either *per-vm* or *per-tenant*. The position of each kernel region is as follows:

    ```
    XXXXXX
    ||||| \--> kernel physical address
    |||| \---> kernel virtual address
    ||| \----> direct physical mapping (physmap)
    || \-----> vmalloc
    | \------> vmemmap
     \-------> modules base
    ```

  Therefore, the corresponding kernel region will be randomized *per-tenant* if `1` is specified, and *per-vm* otherwise.
  For example:`kaslr_mt=0xf32v46a2k3y-110010` takes the key `0xf32v46a2k3y` and randomizes the kernel memory regions with this configuration: 
  
  | Kernel Base Addresses   | Randomization |
  | :---------------------- | :------------ |
  | Kernel Physical Address | Per-VM        |
  | Kernel Virtual Address  | Per-Tenant    |
  | Physical Direct Mapping | Per-VM        |
  | Vmalloc/ioremap         | Per-VM        |
  | Vmemmap                 | Per-Tenant    |
  | Modules                 | Per-Tenant    |
  
  
  Accordingly, if the bitmap `000000` is specified, all kernel memory regions will be randomized *per-vm*, just as with standard KASLR.



This feature can be enabled under:

```
-> Processor type and features
   -> Build a relocatable kernel
      -> Randomize the address of the kernel image (KASLR) 
         -> Kernel Address Space Layout Randomization Multi-Tenant
```



## How to compile

>  Compiler used: GCC 8.4.0

Download the Linux kernel source v5.0.6:

```
https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/snapshot/linux-5.0.6.tar.gz
```

Decompress the source:

```
$ tar xf linux-5.0.6.tar.gz
```

Apply the patch:

```
$ cd linux-5.0.6/
$ patch -p1 < ../kaslr-mt_testing.patch 
```

Prepare the kernel source:

```
$ make x86_64_defconfig ; make kvmconfig
```

Compile it:

```
make -j $(nproc)
```



## Reference

[KASLR-MT: Kernel Address Space Layout Randomization for Multi-Tenant Cloud Systems](https://www.sciencedirect.com/science/article/abs/pii/S0743731519304095)

**Authors:** Fernando Vano-Garcia && Hector Marco-Gisbert
