# tdx-measure

## Scope
Command-line tool and Rust library to calculate expected measurement of an Intel TDX guest VM for confidential computing.

The `tdx-measure` tool takes a set of image binaries and platform config files as an input and outputs the corresponding TDX measurements. This makes it possible to exhaustively publish all images and config files that uniquely identify a TDX workload on a machine, making TD environments transparent and auditable.

The tool specifically targets the boot chains from the Canonical TDX [repo](https://github.com/canonical/tdx).

### Acknowledgment
This project is a fork of dstack-mr from the [Dstack-TEE/dstack](https://github.com/Dstack-TEE/dstack) repository.

## Usage

```tdx-measure [OPTIONS] <METADATA>```

### Arguments:
`<METADATA>` Path to metadata json file (see format lower down)

### Options
```
      --two-pass-add-pages         Enable two-pass add pages
      --direct-boot <DIRECT_BOOT>  Enable direct boot (overrides JSON configuration) [possible values: true, false]
      --json                       Output JSON
      --json-file <JSON_FILE>      Output JSON to file
      --platform-only              Compute MRTD and RTMR0 only
      --runtime-only               Compute RTMR1 and RTMR2 only
      --transcript <TRANSCRIPT>    Generate a human-readable transcript of all metadata files and write to the specified file
  -h, --help                       Print help
  -V, --version                    Print version
```

WARNING: when running with `--runtime-only`, the tool will assume a VM memory size higher that 2.75GB.

### Direct Boot

#### Metadata

Create `metadata.json` file with the below metadata:

```
{
  "boot_config": {
    "cpus": 16,
    "memory": "2G",
    "bios": "[path to OVMF.fd]",
    "acpi_tables": "[path to acpi_tables.bin]",
    "rsdp": "[path to rsdp.bin]",
    "table_loader": "[path to table_loader.bin]",
    "boot_order": "[path to BootOrder.bin]",
    "path_boot_xxxx": "[path to directory containing Bootxxxx.bin files]"
  },
  "direct": {
    "kernel": "[path to vmlinuz]",
    "initrd": "[path to initrd]",
    "cmdline": "root=/dev/sda1 console=ttyS0"
  }
}
```

#### Metadata Field Descriptions

- `boot_config`: Platform configuration used to compute MRTD and RTMR[0]
  - `cpus`: Number of virtual CPUs allocated to the TD.
  - `memory`: The amount of memory allocated to the TD (e.g., "2G" for 2 gigabytes).
  - `bios`: Path to file (e.g., `OVMF.fd`) containing a virtual BIOS, which is used to boot the TD image.
    The file can be obtained by setting up the host OS following [these instructions](https://github.com/canonical/tdx/tree/main?tab=readme-ov-file#4-setup-host-os) and retrieving it from `/usr/share/ovmf/OVMF.fd`.
  - `acpi_tables`: Path to file containing ACPI tables, which describe the hardware configuration and device tree that the TD uses to discover and configure hardware.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `rsdp`: Path to file containing a Root System Description Pointer (RSDP), which is an ACPI data structure that provides the address of the RSDT/XSDT table.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `table_loader`: Path to file containing a ACPI table loader, which contains QEMU-specific commands for loading and patching ACPI tables.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `boot_order`: Path to file containing a UEFI BootOrder variable, which specifies the order in which the firmware attempts to boot from different boot options.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `path_boot_xxxx`: Path to directory containing files (e.g., `Boot0000.bin`, `Boot0001.bin`, `Boot0002.bin`) for each Boot#### UEFI variables.
    Each variable defines a specific boot option with its device path and description.
    These files can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.

- `direct`: Direct boot specific configuration used to compute RTMR[1] and RTMR[2]
  - `kernel`: Path to file (e.g., `vmlinuz`) of kernel image, which will be directly loaded and executed by OVMF.
    The file can be obtain by following [these instructions](https://github.com/canonical/tdx/tree/main/guest-tools/direct-boot#prerequisites).
  - `initrd`: Path to file (e.g., `initrd.img`) of initial RAM disk, which is a temporary root filesystem loaded into memory during boot, containing drivers and tools needed to mount the actual root filesystem.
    The file can be obtain by following the [these instructions](https://github.com/canonical/tdx/tree/main/guest-tools/direct-boot#prerequisites).
  - `cmdline`: Kernel command line parameters.
    These parameters specify the kernel command line arguments that are passed to QEMU using the `-append` option.

### Indirect Boot

#### Metadata

Create `metadata.json` file with the below metadata:

```
{
  "boot_config": {
    "cpus": 32,
    "memory": "10G",
    "bios": "[path to OVMF.fd]",
    "acpi_tables": "[path to acpi_tables.bin]",
    "rsdp": "[path to rsdp.bin]",
    "table_loader": "[path to table_loader.bin]",
    "boot_order": "[path to BootOrder.bin]",
    "path_boot_xxxx": "[path to directory containing Bootxxxx.bin files]"
  },
  "indirect": {
    "qcow2": "[path to tdx-guest-ubuntu-25.04-generic.qcow2]",
    "cmdline": "console=ttyS0 root=/dev/vda1",
    "mok_list": "[path to MokList.bin]",
    "mok_list_trusted": "[path to MokListTrusted.bin]",
    "mok_list_x": "[path to MokListX.bin]",
    "sbat_level": "[path to SbatLevel.bin]"
  }
}
```

#### Metadata Field Descriptions

- `boot_config`: Platform configuration used to compute MRTD and RTMR[0]
  - `cpus`: Number of virtual CPUs allocated to the TD.
  - `memory`: The amount of memory allocated to the TD (e.g., "2G" for 2 gigabytes).
  - `bios`: Path to file (e.g., `OVMF.fd`) containing a virtual BIOS, which is used to boot the TD image.
    The file can be obtained by setting up the host OS following [these instructions](https://github.com/canonical/tdx/tree/main?tab=readme-ov-file#4-setup-host-os) and retrieving it from `/usr/share/ovmf/OVMF.fd`.
  - `acpi_tables`: Path to file containing ACPI tables, which describe the hardware configuration and device tree that the TD uses to discover and configure hardware.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `rsdp`: Path to file containing a Root System Description Pointer (RSDP), which is an ACPI data structure that provides the address of the RSDT/XSDT table.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `table_loader`: Path to file containing a ACPI table loader, which contains QEMU-specific commands for loading and patching ACPI tables.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `boot_order`: Path to file containing a UEFI BootOrder variable, which specifies the order in which the firmware attempts to boot from different boot options.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `path_boot_xxxx`: Path to directory containing files (e.g., `Boot0000.bin`, `Boot0001.bin`, `Boot0002.bin`) for each Boot#### UEFI variables.
    Each variable defines a specific boot option with its device path and description.
    These files can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.

- `indirect`: Indirect boot specific configuration used to compute RTMR[1] and RTMR[2]
  - `qcow2`: Path to guest OS disk image in QCOW2 format.
    The image contains the guest filesystem with the bootloader chain (e.g., SHIM and Grub) and a kernel.
    The image can be created by following [these instructions](https://github.com/canonical/tdx/tree/main?tab=readme-ov-file#5-create-td-image).
  - `cmdline`: Kernel command line parameters.
    These parameters can be obtained by executing `cat /proc/cmdline` inside a TD, which is configured identically to the target configuration.
  - `mok_list`: Path to file containing a Machine Owner Key (MOK) list for Secure Boot.
    Contains user-enrolled keys that supplement the vendor-provided Secure Boot keys.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `mok_list_trusted`: Path to file containing a MOK trusted list.
    Contains additional trusted keys for Secure Boot verification.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `mok_list_x`: Path to file containing a MOK blacklist (forbidden keys).
    Contains keys that have been explicitly revoked and should not be trusted.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.
  - `sbat_level`: Path to file containing a SBAT (Secure Boot Advanced Targeting) revocation list.
    Used to revoke specific bootloader versions without revoking their signing keys.
    The file can be extracted by running the [`extract_config_files.py`](extract_config_files.py) script inside a TD, which is configured identically to the target configuration.

### Transcript

The transcript flag makes it possible to generate a human-readable transcript from the different binary configuration files. The command line tool `iasl` needs to be installed in order to disassembled the ACPI tables and include its representation in the transcript.

## Prerequisite

### Install Rust

[If not already done] Install Rust dependency, install Rust, activate it in the current shell, and test the installation:
```
sudo apt update
sudo apt install build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"
cargo --version
```
See [Rust's installation documentation](https://www.rust-lang.org/tools/install) for more detailed information.

## Build the project

```
cargo build --release
```

## Install the CLI tool

```
cargo install --path cli
```



## Info

### Boot Methods
Canonical repo offer two boot options:

1) Direct Boot:
With this method, `OVMF` (the TDVF or virtual firmware) directly boots the kernel image. In this mode, the `kernel`, `initrd` and the kernel `cmdline` are directly supplied to `qemu`.

2) Indirect Boot:
With this method, `tdvirsh` is used to run TDs, the boot chain is more complex and involves `OVMF`, a `SHIM`, `Grub`, and finally the `kernel`+`initrd` image.

### What goes in the measurements

TDX attestation reports expose 4 measurement registers (MR).

The first one, `MRTD`, represent the measurements for the TD virtual firmware binary (TDVF, specifically OVMF.fd in our case).

Three other runtime measurement registers (`RTMR`) correspond to different boot stages and vary depending on the boot chain.

`RTMR[0]` contains firmware configuration and platform specific measurements. This includes hashes of:
- The TD HOB which mostly contains a description of the memory accessible to the TD.
- TDX configuration values.
- Various Secure Boot variables.
- ACPI tables that describe the device tree.
- Boot variables (BootOrder and others).
- [for indirect boot only] [SbatLevel](https://github.com/rhboot/shim/blob/main/SbatLevel_Variable.txt) variable.

`RTMR[1]` contains measurements of the `kernel` for direct boot. For indirect boot, it contains measurement for the bootchain a.k.a. `gpt` (GUID Partition Table), `shim`, and `grub`.

`RTMR[2]` contains measurements of the kernel `cmdline` and `initrd` for direct boot. For indirect boot, it also contains the measurements of machine owner key [(MOK) variables](https://github.com/rhboot/shim/blob/main/MokVars.txt).
