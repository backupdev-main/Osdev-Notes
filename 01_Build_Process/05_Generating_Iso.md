# Generating A Bootable Iso

Generating a bootable iso is specific to the bootloader of choice, both grub and limine are outlined below!
The generated iso can then be used as a cdrom in an emulator, or burnt to a real disk or usb thumb drive as we would do for installing any other operating system.

## Xorriso

Xorriso is a tool used to create iso disk images. Iso is actually quite a complex format, as it aims to be compatible with a lot of different formats and ways of booting.

A walkthrough of xorriso is outside the scope of this chapter, but just know it's the standard tool for working with the iso format.

## Grub (Multiboot 2)

Grub provides a tool called `grub-mkrescue`. Originally it was intended for making rescue disks (hence the name), but it works just fine for our purposes.

The tool is very straight forward, just create a root folder and populate that with what we want to appear on the final iso. In our case we'll need `grub.cfg` and our kernel (called `kernel.elf` for now), so we'll create the following directory layout:

```
disk/
  \-boot/
      |-grub/
      |   \-grub.cfg
      \-kernel.elf
```

We can now run `grub-mkrescue -o my_iso.iso disk`, and it will generate an iso called `my_iso.iso` with the contents of `disk/` as the root directory.

### Grub.cfg

This file contains some global config options for grub (setting boot timeouts, etc...) as well as a list of boot options for this disk. In our case we only have the one option of our kernel.

Global options can be changed using the `set option=value` syntax, and a list is available from the grub documentation.

For each boot option, a menu entry needs to be created within `grub.cfg`. A menu entry consists of a header, and then a body containing a series of grub commands to boot the operating system. This allows grub to provide a flexible interface for more complex setups.

However in our case, we just want to boot our kernel using the standard multiboot2 protocol. Fortunately grub has a built in command (`multiboot2`) for doing just that:

```
menuentry "My Operating System" {
    multiboot2 /boot/kernel.elf
    boot
}
```

Note the last command `boot`, this tells grub to complete the boot process and actually run our kernel.

## Limine (Limine Protocol, Multiboot 2)
The process for limine is a little more manual: we must build the iso ourselves, and then use the provided tool to install limine onto the iso we created.

To get started we'll want to create a working directory to use as the root of our iso. In this case we'll use `disk/`. Next we'll need to clone the latest binary branch of limine that at time of writing is at version `8.x` (using `git clone https://github.com/limine-bootloader/limine.git --branch=v8.x-binary --depth=1`) and copy 'limine.sys', 'limine-cd.bin', and 'limine-cd-efi.bin' into `disk/limine/`, creating that directory if it doesn't exist.

Now we can copy our `limine.conf` and kernel into the working directory. The `limine.conf` file needs to be in one of a few locations in order to be found, the best place is the root directory.

Now we're ready to run the following xorriso command:

```sh
xorriso -as mkisofs -b limine-cd.bin -no-emul-boot \
    -boot-load-size 4 -boot-info-table --efi-boot \
    limine-cd-efi.bin -efi-boot-part --efi-boot-image \
    --protective-msdos-label disk -o my_iso.iso
```

That's a lot of flags! Because of how flexible the iso format is, we need to be quite specific when building one. If curious about how crazy things can get, take a look at the flags grub-mkrescue generates behind the scenes.

Now we have an iso with our kernel, config and bootloader files on it that can boot if using uefi. If we want to make it bootable also using the `bios` an extra step is required, we need to install limine into the boot partitions, it can be done using the `limine` command like so:

```sh
./limine/limine bios-install my_iso.iso
```

That took a little more work than grub, but this can (and should) be automated as part of the build system. If can't find `limine` inside the cloned limine directory, we may need to run `make -C limine` from there for it to be build.

### Limine.conf
Similar to grub, limine also uses a config file. This config file has its own documentation, which is available in the limine repository.

The `limine.conf` file lists each boot entry as a title, followed by a series of key-value pairs. To boot our example from above using stivale 2, our config might look like the following:

```
# The entry name that will be displayed in the boot menu.
/My Operating System
    # We use the Limine boot protocol.
    protocol: limine

    # Path to the kernel to boot. boot():/ represents the partition on which limine.conf is located.
    kernel_path: boot():/boot/kernel.elf
```

One thing to note is how the kernel's path is specified. It uses the URI-like format, which begins with a protocol and then a protocol-specific path. In our case we're using the boot disk (`boot():`), and then an absolute path (`/boot/kernel.elf`)
