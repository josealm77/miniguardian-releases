# MiniGuardian — beta releases and documentation

MiniGuardian is a host security daemon for Linux with an AI-containment layer,
built on the deterministic **Jayce kernel**. It is created by **Jose
Almendarez**, Jayce Automata Research.

- **AI containment.** `mg-cli` runs an AI coding tool (Claude Code, opencode,
  and others) inside a Landlock sandbox with default-deny networking. Every
  file write, command or network call that changes the system waits for a
  human to approve it, and the decision needs your password. Enforcement is
  deterministic: rules, proof-checked missions and counters, never a language
  model.
- **Host protection.** Process, file, memory, network and browser detectors
  feed a threat brain that rates the host Calm, Suspicious, Hostile or Paranoid,
  and can block, quarantine and restore.
- **Self-protection.** Sealed immutable binaries, a hash manifest, an
  independent antifreeze sentinel, shadow copies of critical files and a
  hash-chained audit log.
- **Attack immunity.** It acts during an attack and learns from it:
  - ransomware is paused mid-run and the files are restored;
  - an attack it stopped once is refused at its first step next time, after
    a restart and under a new name or path;
  - every attack it survives makes the machine harder to attack again.
- **Kernel judge.** An independent copy of the decision engines runs in its
  own small VM and must agree with every decision, so taking over root alone
  is not enough to change what MiniGuardian decides.
- **Supply-chain protection.** Dependencies of plugins and AI sessions are
  checked against their pinned checksums before anything runs (Rust, Python,
  Node, git). If a bad package gets in anyway, it has no network of its own
  and can't quietly send your secrets anywhere.
- **Controlled internet.** Plugin commands, and AI agents' commands when you
  opt in, reach the internet only through MiniGuardian's proxy:
  - one pop-up per new site;
  - every connection recorded;
  - programs that go around the proxy have no network.
- **Jayce kernel engines.** The daemon runs the kernel's engines in-process at
  1 kHz, including the Impossible State Detector, the Predictive Execution
  Horizon and the Self-Healing Causality Mesh, and the dashboard shows them live.

**Tested, not just claimed:**
- **126 documented security features.**
- **A 525-scenario red-team campaign:** 525 of 525 passed, all stable.
- **A decision benchmark:** 170 of 170 correct, before and after the attack
  traffic.
- **A live repeat drill:** ransomware, a rootkit, a malicious download and
  cron persistence were each refused at the first step after a restart.

Details are in the [developer manual](docs/OPEN-PLUGIN-DEVELOPER-MANUAL.md#10-how-secure-is-it-honestly).

**What it does not do:** MiniGuardian raises the cost of an attack and
constrains AI tools. It cannot stop an attacker who already has root on the
machine. The plan for enforcement that can is in
[`docs/ROADMAP_COHOST.md`](docs/ROADMAP_COHOST.md).

This repository holds the **signed beta packages** (under
[Releases](../../releases)) and the **public documentation**. The MiniGuardian
and Jayce kernel source code is private; reviewers and partners can ask for
access.

## Install the beta (0.9.1-beta-29)

Copy and paste the block for your distro. It downloads the package, the
checksums, the signature and the public key, verifies them, and installs only
if everything checks out.

**Ubuntu, Pop!_OS, Debian**
```bash
mkdir -p ~/miniguard-0.9.1-beta-29 && cd ~/miniguard-0.9.1-beta-29
for f in miniguard_0.9.1-beta-29_amd64.deb SHA256SUMS SHA256SUMS.asc JAYCE_RELEASE.asc; do
  wget -q https://github.com/josealm77/miniguardian-releases/releases/download/v0.9.1-beta-29/$f
done
gpg --import JAYCE_RELEASE.asc
gpg --verify SHA256SUMS.asc SHA256SUMS && sha256sum -c SHA256SUMS --ignore-missing \
  && sudo apt install ./miniguard_0.9.1-beta-29_amd64.deb
```

<!-- fedora-install-start -->
**Fedora, RHEL, AlmaLinux, Rocky**
```bash
mkdir -p ~/miniguard-0.9.1-beta-29 && cd ~/miniguard-0.9.1-beta-29
for f in miniguard-0.9.1-29.x86_64.rpm SHA256SUMS SHA256SUMS.asc JAYCE_RELEASE.asc; do
  curl -fsSLO https://github.com/josealm77/miniguardian-releases/releases/download/v0.9.1-beta-29/$f
done
gpg --import JAYCE_RELEASE.asc
gpg --verify SHA256SUMS.asc SHA256SUMS && sha256sum -c SHA256SUMS --ignore-missing \
  && sudo dnf install ./miniguard-0.9.1-29.x86_64.rpm
```
<!-- fedora-install-end -->

`gpg --verify` must print **Good signature from "Jayce Automata Research"**
using key `962FD600CF524AA8B610E06B7D7EE65D52E1182E`
(`962F D600 CF52 4AA8 B610  E06B 7D7E E65D 52E1 182E`). If it says BAD, or
shows a different key, stop and don't install. A warning that the key "is not
certified with a trusted signature" is normal the first time; comparing the
fingerprint above is what establishes trust.

Then tell MiniGuardian what the machine is for — `sudo mg-setup` (workstation,
server, drone or vehicle; it picks the protection numbers and the folders that
get ransomware pre-write copies). Until you do, the dashboard says "profile not
chosen".

Open the dashboard with `sudo miniguard-status` and run an AI tool
governed with `sudo mg-cli claude` (or `opencode`, and others). Start with the
[TUI User Guide](docs/TUI_USER_GUIDE.md) and the
[AI Containment Manual](docs/AI_CONTAINMENT_MANUAL.md).

## Build with MiniGuardian

- **Plugins in any language.** `mg-plugin new hello --lang python` (or node,
  rust, apodeixis) creates a plugin that runs inside MiniGuardian's sandbox.
  It asks for everything it does, and you approve what it may touch at
  install. AI agents can use installed plugins as tools.
  - **First plugin:** [Plugin Quickstart](docs/PLUGIN-QUICKSTART.md), about
    15 minutes.
  - **Full guide:** [Open Plugin Developer Manual](docs/OPEN-PLUGIN-DEVELOPER-MANUAL.md).
    It covers what MiniGuardian and the Jayce kernel are, building plugins,
    adding MiniGuardian to existing projects, OS images and servers,
    commands, how approvals work, and an honest security summary.
- **`mg-why -- <command>`** explains why any program failed, in any language.
  It names the file and line, the cause and the processes involved, with
  secrets masked. See [Explain a failure](docs/EXPLAIN-A-FAILURE.md).

### Fedora: blank screen after reboot (beta-5 to beta-10)

On Fedora Workstation, beta-5 to beta-10 could stop at a blank screen with a
cursor after the first reboot. Fixed in beta-11. If it happened to you, boot
a Fedora live USB, open a terminal, and remove the line the installer added:

```bash
lsblk -f                                         # find the large btrfs partition, e.g. nvme0n1p3
sudo mount -o subvol=root /dev/nvme0n1p3 /mnt    # use the partition you found
sudo sed -i '/^# MiniGuardian sealed deployment — hidepid=2$/{N;/\nproc \/proc proc defaults,hidepid=2 0 0$/d}' /mnt/etc/fstab
sudo umount /mnt
```

Reboot, then install the latest beta above (upgrading also removes that line).

## Uninstall

**Ubuntu, Pop!_OS, Debian**
```bash
sudo apt purge miniguard     # removes everything, including daemon state and keys
# or: sudo apt remove miniguard   (keeps state in /var/lib/jayce/miniguard for a later reinstall)
```

**Fedora, RHEL, AlmaLinux, Rocky**
```bash
sudo dnf remove miniguard
sudo rm -rf /var/lib/jayce/miniguard   # RPM has no purge: delete the state by hand
```

Removal lifts the seal first: it stops the services, unlocks the immutable
files and directories, and removes the read-only mount on `/usr/local/bin`.
It then undoes the host changes MiniGuardian made: the `hidepid` line it
added to `/etc/fstab` (a `hidepid` line you added yourself is kept) and the
browser integration files. **Kernel lockdown** stays at `integrity` until
the next reboot, because Linux cannot lower it on a running system;
MiniGuardian never changes your kernel command line, so a reboot restores
the default.

These steps apply to 0.9.1-beta-7 and later. Removing beta-6 or earlier needs
the seal lifted by hand first, because those versions' removal scripts were
incomplete:

```bash
sudo systemctl stop miniguard-sentinel miniguard
sudo mg-seal --unseal
while findmnt /usr/local/bin >/dev/null; do sudo umount -l /usr/local/bin; done
sudo chattr -i /etc/miniguard /usr/local/bin /usr/share/miniguard /opt/jayce 2>/dev/null
for f in $(dpkg -L miniguard) /etc/miniguard/*; do [ -e "$f" ] && sudo chattr -i "$f"; done
sudo apt purge -y miniguard
sudo sed -i '/^# MiniGuardian sealed deployment — hidepid=2$/{N;/\nproc \/proc proc defaults,hidepid=2 0 0$/d}' /etc/fstab
grep -q hidepid /etc/fstab || sudo mount -o remount,hidepid=0 /proc
sudo rm -rf /var/lib/jayce/miniguard
sudo systemctl daemon-reload
```

## Documentation

- [Overview](docs/overview.md): what is implemented, partial and planned
- [TUI User Guide](docs/TUI_USER_GUIDE.md), [Error and Recovery Guide](docs/MINIGUARDIAN_ERROR_GUIDE.md)
- [AI Containment Manual](docs/AI_CONTAINMENT_MANUAL.md), [Governed Tools Reference](docs/GOVERNED_TOOLS_REFERENCE.md)
- [Security Features Catalog](docs/SECURITY_FEATURES_CATALOG.md)
- [Open Plugin Developer Manual](docs/OPEN-PLUGIN-DEVELOPER-MANUAL.md), [Plugin Quickstart](docs/PLUGIN-QUICKSTART.md)
- [Explain a failure (`mg-why`)](docs/EXPLAIN-A-FAILURE.md), [Write a language file](docs/WRITE-A-LANGUAGE-FILE.md)
- [Apodeixis language guide](docs/APODEIXIS_LANGUAGE_GUIDE.md), [Jayce 1 kHz tick whitepaper](docs/JAYCE_1000HZ_TICK_WHITEPAPER.md)
- [Cohost roadmap](docs/ROADMAP_COHOST.md)
- Papers: [`docs/papers/`](docs/papers/)

## Use policy

MiniGuardian and the Jayce kernel are for **civilian protection**: people,
homes, businesses, and critical infrastructure such as water, power, health
care and industrial control. They will never be licensed for weapons or
military use. The license makes this non-waivable, and the kernel's Shim ABI
refuses military and weapons use classes by design.

## License

Proprietary, under the Jayce Automata Research Source License v1.1 (see
[`LICENSE`](LICENSE)). In short:
- **Run it** for personal use and evaluation.
- **Ship the official signed packages, unmodified, with your program** or as a
  dependency of it. This is free for individuals, small organisations and
  non-commercial programs; larger companies need a commercial licence.
- **Write plugins:** the open plugin system is open to everyone, and the
  plugins you write are yours to license and sell.
- **Changing MiniGuardian itself, or copying its code,** needs written
  permission.
- **Weapons use is never permitted.**

Contact: JayceAutomataResearch@proton.me · [www.jayceautomata.com](https://www.jayceautomata.com)
