# Springboard

VERSION: 3.0.1/EARLY/UNRELEASED  
LICENSE: [Apache-2.0](https://github.com/azyklus/springboard/blob/trunk/LICENSE)  
README: [Where would you rather be?](https://xkcd.com/650/)  
INFO:

The upstream bootloader, and lib(s) source tree can be found here.

An experimental x86_64 bootloader that works on both BIOS and UEFI systems. Written in Rust and some inline assembly, buildable on all platforms without additional build-time dependencies (just some `rustup` components).

Trident is an operating system originally developed as part of a series on [Medium](https://medium.com/@zaiqi) and [MBP2](https://mbp2.blog/@az)
before being co-opted into a personal research project.

The `springboard` bootloader project is a fork of [rust-osdev/bootloader](https://github.com/rust-osdev/bootloader)
fine-tuned to the needs of the Trident 3 project.

As of 2023.11, you will need:
- The Rust nightly as of 2023.11.12, which you may obtain through the official website: https://rust-lang.org/learn/get-started
- Docker to run the build environment: https://www.docker.com/get-started/

### Extensions

You will eventually be able to develop your own extensions to Trident 3 through a proposed Extensions API.

### Contributing

If you'd like to contribute to this project, please [fork](https://github.com/azyklus/sys3/fork) it and
submit pull requests with your desired features.

1. [Fork it.](https://github.com/azyklus/sys3/fork)
2. ????? (I forgot what went here)
3. Submit pull request with your feature. ("[FEATURE] describe your feature").
4. Profit?

### Useful Links

- [MBP2 Page](https://mbp2.blog/src/@trident)
- [Matrix Room](https://matrix.to/#/%23two-worlds:mozilla.org)
- [Discord Server](https://discord.gg/B9agTdVH4U)

### Requirements

You need a nightly [Rust](https://www.rust-lang.org) compiler with the `llvm-tools-preview` component, which can be installed through `rustup component add llvm-tools-preview`.

### Usage

To use this crate, you need to adjust your kernel to be bootable first. Then you can create a bootable disk image from your compiled kernel. These steps are explained in detail below.

If you're already using an older version of the `springboard` crate or the `rust-osdev/bootloader` crate, follow our [migration guides](docs/migration).

### Kernel

To make your kernel compatible with `springboard`:

- Add a dependency on the `springboard_api` crate in your kernel's `Cargo.toml`.
- Your kernel binary should be `#![no_std]` and `#![no_main]`.
- Define an entry point function with the signature `fn kernel_main(boot_info: &'static mut springboard_api::BootInfo) -> !`. The function name can be arbitrary.
  - The `boot_info` argument provides information about available memory, the framebuffer, and more. See the API docs for `springboard_api` crate for details.
- Use the `entry_point` macro to register the entry point function: `springboard_api::entry_point!(kernel_main);`
  - The macro checks the signature of your entry point function and generates a `_start` entry point symbol for it. (If you use a linker script, make sure that you don't change the entry point name to something else.)
  - To use non-standard configuration, you can pass a second argument of type `&'static springboard_api::BootloaderConfig` to the `entry_point` macro. For example, you can require a specific stack size for your kernel:
    ```rust
    const CONFIG: springboard_api::BootloaderConfig = {
        let mut config = springboard_api::BootloaderConfig::new_default();
        config.kernel_stack_size = 100 * 1024; // 100 KiB
        config
    };
    springboard_api::entry_point!(kernel_main, config = &CONFIG);
    ```
- Compile your kernel to an ELF executable by running **`cargo build --target x86_64-unknown-none`**. You might need to run `rustup target add x86_64-unknown-none` before to download precompiled versions of the `core` and `alloc` crates.
- Thanks to the `entry_point` macro, the compiled executable contains a special section with metadata and the serialized config, which will enable the `springboard` crate to load it.

### Booting

To combine your kernel with a bootloader and create a bootable disk image, follow these steps:

- Move your full kernel code into a `kernel` subdirectory.
- Create a new `os` crate at the top level that defines a [workspace](https://doc.rust-lang.org/cargo/reference/workspaces.html).
- Add a `build-dependencies` on the `bootloader` crate.
- Create a [`build.rs`](https://doc.rust-lang.org/cargo/reference/build-scripts.html) build script.
- Set up an [artifact dependency](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#artifact-dependencies) to add your `kernel` crate as a `build-dependency`:
  ```toml
  # in Cargo.toml
  [build-dependencies]
  kernel = { path = "kernel", artifact = "bin", target = "x86_64-unknown-none" }
  ```
  ```toml
  # .cargo/config.toml

  [unstable]
  # enable the unstable artifact-dependencies feature, see
  # https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#artifact-dependencies
  bindeps = true
  ```
  Alternatively, you can use [`std::process::Command`](https://doc.rust-lang.org/stable/std/process/struct.Command.html) to invoke the build command of your kernel in the `build.rs` script.
- Obtain the path to the kernel executable. When using an artifact dependency, you can retrieve this path using `std::env::var_os("CARGO_BIN_FILE_MY_KERNEL_my-kernel")`
- Use `springboard::UefiBoot` and/or `springboard::BiosBoot` to create a bootable disk image with your kernel.
- Do something with the bootable disk images in your `main.rs` function. For example, run them with QEMU.

See our [disk image creation template](docs/create-disk-image.md) for a more detailed example.

## Architecture

This project is split into three separate entities:

- A [`springboard_api`](./api) library with the entry point, configuration, and boot info definitions.
  - Kernels should include this library as a normal cargo dependency.
  - The provided `entry_point` macro will encode the configuration settings into a separate ELF section of the compiled kernel executable.
- [BIOS](./bios) and [UEFI](./uefi) binaries that contain the actual bootloader implementation.
  - The implementations share a higher-level [common library](./common).
  - Both implementations load the kernel at runtime from a FAT partition. This FAT partition is created
  - The configuration is read from a special section of the kernel's ELF file, which is created by the `entry_point` macro of the `bootloader_api` library.
- A `springboard` library to create bootable disk images that run a given kernel. This library is the top-level crate in this project.
  - The library builds the BIOS and UEFI implementations in the [`build.rs`](./build.rs).
  - It provides functions to create FAT-formatted bootable disk images, based on the compiled BIOS and UEFI bootloaders.

## License

Licensed under Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE) or http://www.apache.org/licenses/LICENSE-2.0).

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be licensed as above, without any additional terms or conditions.
