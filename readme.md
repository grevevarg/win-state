# win-state

Preflight and state configuration for a Windows 10 laptop (cloud engineering
course, using VMware Workstation for the lab VMs).

## 1. Install packages with winget

From an elevated PowerShell on Windows:

    winget configure winget-config.yml --accept-configuration-agreements

`winget configure` is the DSC runner that ships with winget; it installs the
`WinGet` DSC resources it needs on first run and prompts only when a package
requires it. The config's `isWindows10` assertion stops it from running on an
unsupported OS.

VMware Workstation is **not** installed here by winget - it is installed from
the course mirror by the `software.yml` playbook (step 5) because the course
pins a specific build (26H1) that winget does not provide.

## 2. Install Ansible (inside WSL)

Ansible is a Linux control node, so there is **no winget package** for it.
Install it inside your WSL distro:

    # inside WSL (use a venv to avoid "externally managed environment" errors)
    python3 -m venv ~/.ansible
    ~/.ansible/bin/pip install ansible pywinrm
    ~/.ansible/bin/ansible-galaxy collection install ansible.windows community.windows

`pywinrm` is what Ansible uses for the WinRM connection. The `ansible.windows`
collection provides most of the `win_*` modules used by the playbooks; the
`community.windows` collection is also required for two of them
(`win_unzip`, `win_lineinfile`).

## 3. One-time Windows prep

From an elevated PowerShell on Windows:

    Enable-PSRemoting -SkipNetworkProfileCheck -Force

This starts the WinRM service that the playbooks connect to. Playbooks escalate
with `runas`, which relies on the Secondary Logon service (`seclogon`) - it is
enabled by default on Windows 10/11.

WSL2 runs on its own virtual network, so `localhost` inside WSL is *not* the
Windows host by default. Either:

* enable mirrored networking by creating `%USERPROFILE%\.wslconfig`:

      [wsl2]
      networkingMode=mirrored

  then run `wsl --shutdown` and reopen the terminal, or

* pass the Windows host's LAN IP on every run:

      -e ansible_host=192.168.x.x

## 4. Run the playbooks

All playbooks target the local Windows host over WinRM; **no inventory file is
needed**:

    ansible-playbook -i localhost, registry.yml -u <windows-user> --ask-pass
    ansible-playbook -i localhost, software.yml  -u <windows-user> --ask-pass
    ansible-playbook -i localhost, starship.yml -u <windows-user> --ask-pass
    ansible-playbook -i localhost, fonts.yml     -u <windows-user> --ask-pass

Or all four in one run:

    ansible-playbook -i localhost, registry.yml software.yml starship.yml fonts.yml -u <windows-user> --ask-pass

Notes:

* `<windows-user>` must be an admin: HKLM registry changes and service/font
  installs escalate via `runas` (become). HKCU settings are written to that
  user's hive.
* Add `--ask-become-pass` only if the elevation password differs from the
  login password.
* `registry.yml` hardens privacy/telemetry settings, `software.yml` installs
  the course lab tools (next section), `starship.yml` configures the Starship
  prompt, and `fonts.yml` installs Fira Code, Cascadia Code, JetBrainsMono
  Nerd, and Hack Nerd fonts.

## 5. Course lab setup (VMware + VMs)

`software.yml` installs VMware Workstation and stages the course files for the
lab VMs, all from the course mirror (`files.cloudinfra.se`):

* installs **VMware Workstation 26H1** silently (the build the course uses)
* creates `C:\VMs\yh2026\` and downloads the **Windows 10 Business 22H2 ISO**
  (~6.4 GB) and the **GNS3 VM for VMware Workstation 2.2.61** (~866 MB) there

The big downloads are skipped if the files already exist, so the playbook can
be re-run safely. Then, in VMware Workstation:

1. Create a new VM from the Windows 10 ISO in `C:\VMs\yh2026\`.
2. Import the extracted GNS3 appliance from `C:\VMs\yh2026\GNS3VM\`
   (Open -> `GNS3 VM.ovf`), and point GNS3 at that VM.