# WinCrypt SSH Agent

> [!NOTE]
> This project builds on [boypt/WinCryptSSHAgent](https://github.com/boypt/WinCryptSSHAgent), a fork of [buptczq/WinCryptSSHAgent](https://github.com/buptczq/WinCryptSSHAgent), and includes selected changes from [rfdonnelly/WinCryptSSHAgent](https://github.com/rfdonnelly/WinCryptSSHAgent), another fork of the original.
> 
> Changes from [boypt/WinCryptSSHAgent](https://github.com/boypt/WinCryptSSHAgent):
> 
> * Self-service key import — auto-load at startup plus tray import, no `ssh-add` needed.
> * Signing confirmation — Manual/Auto mode per signing request, persisted across restarts.
> * Source-tagged notifications — toasts show the requesting transport with categorized icons.
> * Windows ARM64 builds — `make all` builds amd64 and ARM64 binaries; release builds are versioned from Git tags.
> * Broader protocol support — XShell Xagent compatibility, Hyper-V vsock, graceful shutdown.
> 
> Change from [rfdonnelly/WinCryptSSHAgent](https://github.com/rfdonnelly/WinCryptSSHAgent):
> 
> * Added `--smart-card-logon-only` option.
> 
>   This option filters Windows Certificate Store certificates to include only those that have both of the following Extended/Enhanced Key Usage OIDs:
> 
>   * Client Authentication (1.3.6.1.5.5.7.3.2)
>   * Smart Card Logon (1.3.6.1.4.1.311.20.2.2)

## Introduction

WinCrypt SSH Agent is an SSH agent based on the Windows CryptoAPI.

It allows other programs to use SSH keys stored in the Windows Certificate Store for authentication. Windows user certificates and PIV-compatible smart cards, such as YubiKeys, are accessed through the Windows Certificate Store. Some cards or algorithms, such as YubiKey ECC certificates, may require the card manufacturer's smart-card minidriver.

OpenSSH private keys can also be loaded into an in-memory keyring. Imported keys and passphrases are kept in memory and are not written to disk.

## Overview

![Overview](overview.svg)

## Feature

* Use PIV-compatible smart cards through the Windows Certificate Store (some cards or algorithms may require a smart-card minidriver)
* Use OpenSSH certificates with smart-card keys
* Automatically load OpenSSH private keys at startup or import them from the tray menu
* Choose Auto Confirm or Manual Confirm for signing requests
* Show the requesting client in authentication notifications
* Select Smart Card Logon certificates with `--smart-card-logon-only`
* Build for Windows amd64 and ARM64

## Compatibility

There are several different SSH agent protocols used by Windows applications. WinCrypt SSH Agent supports:

* Cygwin UNIX Socket
* Windows UNIX Socket (Windows 10 version 1803 or later)
* Windows OpenSSH named pipe
* Pageant SSH Agent Protocol
* XShell Xagent Protocol
* Hyper-V vsock for WSL2 and Linux guests

With these protocols, one running agent can serve many SSH clients, including:

* Git for Windows
* Windows Subsystem for Linux
* Windows OpenSSH
* PuTTY
* JetBrains
* SecureCRT
* XShell
* Cygwin
* MSYS2 / MinGW
* ...

Some clients that use the Pageant protocol connect automatically and do not require a separate setting in the tray menu.

## Installing

### Manually Install

Download a build from the [Releases page](https://github.com/martinj9320/WinCryptSSHAgent/releases). Create a shortcut in the Windows Startup folder if you want the agent to start when you sign in.

## Usage

### Basic Usage

1. Start WinCryptSSHAgent.
2. Right-click its notification-area icon.
3. Select a menu item to view or copy the settings for an SSH client.

Use **Show Public Keys** to view or copy the available public keys. Choose **Import Key…** to add a private key to the in-memory keyring.

See the [YubiKey with WSL tutorial](doc/wsl_tutorial.md) for an example of using a PIV smart card with SSH from WSL.

### Work with Xshell

1. Install and run WinCryptSSHAgent.
2. Open the Properties dialog box of your session.
3. From Category, select 'SSH' and select 'Use Xagent (SSH agent)' for passphrase handling.
4. From Category, select 'Authentication' and select 'Public Key' as the authentication method.

### Work with WSL

Select **Show WSL Settings** from the tray menu and use the displayed command to set `SSH_AUTH_SOCK` in WSL. On supported Windows versions, the agent uses a UNIX socket. If that is unavailable, it provides a local TCP endpoint that can be bridged from WSL with `socat`.

### Work with Hyper-V / WSL2

The agent can accept connections over a Hyper-V vsock service (ID `0x22223333`). For WSL2 or Linux guests, select **Show WSL2 / Linux On Hyper-V Settings** from the tray menu and use the provided `socat` command to connect a guest-side UNIX socket to the host agent. This does not require port forwarding. `socat` 1.7.4 or later also supports the `VSOCK-CONNECT` address form.

When WinCryptSSHAgent runs inside a Windows Hyper-V guest, it detects the host agent and forwards signing requests to it. Run `WinCryptSSHAgent.exe -i` in the guest to register the communication service; this requires administrator privileges and a reboot.

### Smart Card Logon Only

Add `--smart-card-logon-only` to the application shortcut to select certificates for Smart Card Logon. The option applies to certificates in the Windows Certificate Store, not private keys loaded into the in-memory keyring.

The filter uses these Extended/Enhanced Key Usage OIDs:

* Client Authentication (1.3.6.1.5.5.7.3.2)
* Smart Card Logon (1.3.6.1.4.1.311.20.2.2)

### OpenSSH Private Keys

At startup, the agent looks for private keys matching `~/.ssh/id_*`, excluding public-key and certificate files. Set `WCSA_KEYS` to add other key paths, separated by `;` on Windows. Paths containing spaces are supported. Use **Import Key…** in the tray menu to select a key manually.

For encrypted keys, startup loading tries `WCSA_KEY_PASSPHRASE`, then the optional `WCSA_ASKPASS` helper, and then a passphrase dialog. A passphrase entered successfully can be reused for other keys during the current run. The startup variables are cleared after auto-loading; manual imports use the built-in dialog.

If a key cannot be read, parsed or unlocked, it is skipped and loading continues. PuTTY `.ppk` files are not supported; convert them to OpenSSH format with PuTTYgen.

### OpenSSH Certificates

OpenSSH certificates contain a public key and identity information signed with an SSH key. They use a format different from X.509, so an X.509 certificate cannot be converted into an OpenSSH certificate.

To use an OpenSSH certificate with a smart-card key, place the certificate in your user profile folder and name it `<Your Certificate Common Name>-cert.pub` or `<Your Certificate Serial Number>-cert.pub`.

### Signing Confirmation

By default, signing requests are confirmed automatically. Select **Manual Confirm** from the tray menu to approve each signing request in a Yes/No dialog. The selected mode is saved in the current user's registry and restored when the agent starts. In Auto Confirm mode, authentication notifications show the key and requesting client.

Start with `-confirm` or set `WCSA_CONFIRM=1` to enable Manual Confirm.

### Debug log

1. Run `setx WCSA_DEBUG 1`.
2. Close and restart WinCryptSSHAgent to take effect.
3. Reproduce the problem.
4. Find the log at `%USERPROFILE%\WCSA_DEBUG.log`.

## Advanced User Manual

### Environment variables

| Variable | Effect |
| --- | --- |
| `WCSA_DEBUG=1` | Write diagnostic output to `%USERPROFILE%\WCSA_DEBUG.log`. |
| `WCSA_CONFIRM=1` | Force Manual Confirm mode. |
| `WCSA_KEYS` | Additional startup key paths, separated by `;` on Windows. Cleared after auto-loading. |
| `WCSA_KEY_PASSPHRASE` | Passphrase tried for encrypted keys during startup loading; cleared after loading. |
| `WCSA_ASKPASS` | Helper invoked with the passphrase prompt as its argument during startup loading only; it inherits the agent's environment and is cleared after loading. |
| `WCSA_CHECKSVR=1` | Before CAPI signing, warn if the Smart Card service is stopped and offer to start it. |
| `SSH_AUTH_SOCK` | Client-side socket setting; use the value shown by the relevant tray-menu item. |

### Command-line flags

Run `WinCryptSSHAgent.exe -h` for the full list.

| Flag | Effect |
| --- | --- |
| `-i` | Install the Hyper-V guest communication service (requires elevation). |
| `-confirm` | Require user approval before signing. |
| `-disable-capi` | Use only the in-memory keyring instead of the Windows Certificate Store. |
| `-disable-pin-cache` | Clear the smart-card PIN cache after each operation. |
| `--smart-card-logon-only` | Filter certificates for Smart Card Logon use. |

### Registry

| Key | Purpose |
| --- | --- |
| `HKCU\Software\WinCryptSSHAgent` → DWORD `ConfirmRequired` | Saved signing mode (`0` = Auto, `1` = Manual; absent means Auto). |
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Virtualization\GuestCommunicationServices\<service-GUID>` | Hyper-V guest service registration created by `-i` (requires administrator privileges). |

### Contribute

**Please use issues for everything**

- For a small change, just send a PR.
- For bigger changes, open an issue for discussion before sending a PR.
- You can also contribute by:
  - Reporting issues
  - Suggesting new features or enhancements
  - Improving/fixing documentation
