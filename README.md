# Secure Boot Custom Key Management Toolkit

## Introduction

This is a set of bash scripts designed to help Linux users **create and manage custom Secure Boot keys**.

**Main Purpose**:

- Generate a complete set of custom Secure Boot keys (PK, KEK, db, and MOK)
- Enable a fully independent Secure Boot environment without relying on Microsoft keys
- Safely sign your own bootloader (shim/grub) and kernel modules

## Pre-requisites

**1. System Requirements**

- UEFI firmware with Secure Boot support
- The system must be in **Setup Mode** (see check command below)
- Most of the scripts and commands in this document need to be executed under root access.

**2. Required Software Packages**

**Debian / Ubuntu / Linux Mint:**

```bash
root@localhost:~# apt update
root@localhost:~# apt install mokutil sbsigntool efitools openssl uuid-runtime
```

**Fedora / RHEL / Rocky Linux / AlmaLinux:**

```bash
root@localhost:~# dnf install mokutil sbsigntool efitools openssl util-linux
```

**Arch Linux / Manjaro:**

```bash
root@localhost:~# pacman -S mokutil sbsigntool efitools openssl
```

**3. Check In Setup Mode**

Before running any script, verify Secure Boot status:

```bash
root@localhost:~# mokutil --sb-state
```

**Expected output** :

```
SecureBoot disabled
Platform is in Setup Mode
```

If not in **Setup Mode**, go into BIOS/UEFI and **clear Secure Boot keys**

---

## Generate Keys

```bash
root@localhost:~# ./1.gen_keys
```

**What this script does**:

- Generates PK, KEK, db, and MOK keys and certificates
- Creates EFI Signature Lists (.esl) and signed .auth files
- Creates a `secureboot/` directory with integrity checking

You will be prompted to enter your name (used in certificate CN). Press Enter to use the default.

**Important directories after generation**:

- `secureboot/cert/` — Private keys (`.key`) and certificates (`.crt`)
- `secureboot/cert/` — efi sign list (`.esl`)
- `secureboot/keystore/` — Signed `.auth` files for UEFI import

> **Tip**: Back up the entire `secureboot/` folder, especially the private key files.

---

## Signing Files

After generating keys, you need to sign the following critical files:

### 1. Sign **shim** with the **db** key 

```bash
root@localhost:~# sbsign --key secureboot/cert/db.key \
            --cert secureboot/cert/db.crt \
            --output outfile \
            infile
```

### 2. Sign the following with the **MOK** key:

- **grub**
- **Linux kernel** (`vmlinuz`)
- **Kernel modules** (If you have your own compiled kernel module)

```bash
root@localhost:~# sbsign --key secureboot/cert/MOK.key \
            --cert secureboot/cert/MOK.crt \
            --output outfile \
            infile
```

---

## Import MOK

```bash
root@localhost:~# ./2.import_mok
```

### Why import MOK first?

MOK (**Machine Owner Key**) is an additional key database trusted by **shim**.  
Even after importing the db key into UEFI, shim still needs MOK to trust your signed grub and kernel. Therefore, MOK must be enrolled before your custom-signed bootloader and kernel can boot successfully.

### Import Process

1. Run the script and **set a MOK password** (remember it).
2. The script will import `MOK.der` using `mokutil`.
3. **Reboot immediately**.
4. After reboot, you will enter the blue **MOK Management** screen:
   - Select **Enroll MOK**
   - Select **Continue**
   - Enter the password you set
   - Confirm **Yes** to enroll
   - Reboot again

Once completed, MOK will be permanently trusted by the firmware.

---

## Import UEFI Certificates

```bash
root@localhost:~# ./3.import_uefi_cert
```

This step writes your custom **PK/KEK/db** keys into the UEFI firmware.

After running the script, verify the keys are correctly loaded with:

```bash
root@localhost:~# mokutil -la
```

You should see your custom PK, KEK, and db certificates listed.

reboot again and enter the following command.

```shell
root@localhost:~# mokutil --sb-state
```

**Expected output**

```
SecureBoot enabled
```

## **Good luck with your custom Secure Boot setup!** 🔐

