# aldpro-enterprise-project

A real **ALD Pro 3.0.0** domain — directory service, Kerberos KDC, integrated DNS and
the web management portal — deployed on **Astra Linux Special Edition 1.7.7**, with a
client machine joined to the domain, domain users, groups, sudo and HBAC rules all
provisioned and *proven working*, not just installed. That two-VM proof of concept is
documented below and is what's actually deployed and screenshotted today.

The project's second phase — the [org structure for a ~200-user, ~150-machine
company](#scaling-up-a-200-user-enterprise) running the same stack — is further down
this document. It builds on the exact same domain and the same lessons learned in the
[deployment log](#deployment-log), scaled up to the size where group policy, tiered
support, and departmental access rules actually have to hold up under real
organizational politics.

This is the production counterpart to [`freeipa-lab`](https://github.com/youngnlit-s5/freeipa-lab):
that repo is the open-source twin built on FreeIPA; this one is the actual commercial
product — same underlying FreeIPA/389-ds/MIT-Kerberos core, RusBITech-Astra's own
management portal, packaging and Astra-specific security-level (Parsec MAC) integration
on top. Everything here maps directly onto the domain migrations I run day to day.

![topology](docs/topology.svg)

## What it builds

| Layer | Component |
|---|---|
| **OS** | Astra Linux Special Edition 1.7.7, security level "Смоленск" (max, Parsec MAC) on the DC |
| **Directory** | FreeIPA 389-ds, wrapped by ALD Pro's `aldpro-mp` |
| **Authentication** | MIT Kerberos KDC, single sign-on via sssd on the client |
| **DNS** | Integrated BIND9 (bind9-pkcs11) for the `lab.local` zone |
| **Management** | ALD Pro web portal ("Портал управления"), `https://ald-dc.lab.local` |
| **Access control** | Groups, sudo rules, HBAC rules (FreeIPA `sudorule`/`hbacrule` under the hood) |

## ALD Pro ↔ Microsoft AD mapping

| ALD Pro (this lab) | Microsoft AD equivalent | Notes |
|---|---|---|
| `aldpro-server-install` | `dcpromo` / Install-ADDSForest | promotes a plain OS to a domain controller |
| Портал управления (web portal) | ADUC / ADAC / DNS Manager | one web UI instead of several MMC snap-ins |
| FreeIPA DS (389-ds) | NTDS.dit | LDAP directory back-end |
| MIT KDC | AD KDC | same Kerberos protocol, cross-realm trust is possible |
| BIND9 (bind9-pkcs11), AD-integrated zone | AD-integrated DNS | DC doubles as the zone's DNS server |
| Groups / sudo rules / HBAC rules | Groups / GPO / Restricted Groups | ALD Pro sudo+HBAC rules replace GPO-based privilege GPOs |
| `aldpro-client-installer` / `astra-freeipa-client` | `Add-Computer -DomainName` | client enrollment |
| `aldpro-gpupdate --pm/--gp` | `gpupdate /force` | policy refresh on the client |
| `aldpro-syncer` + `aldpro-syncer-pwdsync` | AD Connect / ADMT | hybrid sync and password sync with a real MS AD forest |
| Global Catalog (`aldpro-gc`) | Global Catalog | cross-domain object lookup for trusted MS AD forests |

The `aldpro-syncer` row is not cosmetic: ALD Pro is built to run *next to* an existing
MS AD forest during a migration, syncing users/groups/passwords both ways so nothing
breaks mid-cutover — this is the same mechanism behind the 200-workstation AD-to-ALD
Pro migration I ran, just exercised here end to end on a disposable pair of VMs instead
of production hardware.

## Layout

```
aldpro-enterprise-project/
├── preseed/                    # Astra unattended-install answer files (not the ALD
│                                # Pro distro itself -- see Getting the software below).
│                                # Passwords are redacted to CHANGE_ME_* placeholders.
│   ├── preseed-dc.cfg
│   └── preseed-cl1.cfg
├── net/
│   └── aldlab-net.xml          # libvirt NAT network, 192.168.150.0/24
└── docs/
    ├── topology.svg
    ├── RUN.md                  # full walkthrough with the real, working commands
    ├── logs/                   # captured output: id, klist, sudo -l, auth.log denial, aldproctl status
    └── screenshots/            # real screenshots -- portal (login/dashboard/users/groups/
                                 # computers/sudo/HBAC) + real terminal sessions (id+klist,
                                 # sudo working, login denied)
```

## Getting the software

ALD Pro is RusBITech-Astra's commercial product. It is **not** bundled in this repo.
The install pulls the base OS from the official Astra Linux ISO and the ALD Pro
packages from RusBITech-Astra's own apt repositories:

```
deb http://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-main 1.7_x86-64 main non-free contrib
deb http://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-update 1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/aldpro/frozen/01/3.0.0/ 1.7_x86-64 main base
```

Get the Astra Linux ISO and the ALD Pro administrator documentation from
[astralinux.ru](https://astralinux.ru) / [rubitech.ru](https://www.rubitech.ru); this
lab was built strictly against *Руководство администратора ALD Pro 3.0.0*.

## Quick start

See [`docs/RUN.md`](docs/RUN.md) for the full walkthrough (VM provisioning, preseed
automation, domain controller promotion, client join, and the exact commands used to
prove access control actually works).

## Proof, not just claims

| Check | Evidence |
|---|---|
| Domain controller up (LDAP/KDC/DNS/portal) | [`docs/logs/aldproctl-status.txt`](docs/logs/aldproctl-status.txt) |
| Client joined, domain identity resolves | `id i.sredoevich` → real domain UID/GID |
| Kerberos SSO works | `klist` shows a real `krbtgt`+`ldap` ticket |
| Sudo rule fires for an allowed user | `sudo -l` → `(root) ALL` |
| Disallowed user is denied, not just un-sudo'd | SSH connection is closed at the PAM/HBAC layer before a shell exists, [`auth.log`](docs/logs/login-deny-authlog.txt) |

All screenshots below are real captures off the running VMs — the portal ones via
headless Chromium driven over the DevTools protocol (real browser, real TLS handshake,
real login), the terminal ones off an actual SSH session into `ald-cl1`. None of it is
staged. (The portal UI is Russian-only in this ALD Pro 3.0.0 build — see the
["can the portal run in English?"](#can-the-portal-run-in-english) note below.)

### Web portal

![Login page](docs/screenshots/aldpro-portal-login.png)
*Login page — `https://ald-dc.lab.local`, username/password or Kerberos SSO.*

![Dashboard after login](docs/screenshots/aldpro-portal-dashboard.png)
*Main dashboard right after a successful login ("Авторизация прошла успешно").*

![Users list](docs/screenshots/aldpro-portal-users.png)
*Users — `admin`, `i.sredoevich` (developers), `m.ivanova` (helpdesk), all Active.*

![Groups list](docs/screenshots/aldpro-portal-groups.png)
*Groups — `developers` and `helpdesk` alongside the built-in FreeIPA/ALD Pro groups.*

![Computers list](docs/screenshots/aldpro-portal-computers.png)
*Computers — both `ald-dc` and `ald-cl1` registered with their real lab IPs, proof the
client join actually reached the directory.*

![Sudo rules](docs/screenshots/aldpro-portal-sudo-rules.png)
*Sudo rule `allow-developers-full`, enabled.*

![HBAC rules](docs/screenshots/aldpro-portal-hbac-rules.png)
*HBAC access rules — the `allow_all` default is disabled so `allow_developers` actually
governs who gets in.*

### Terminal — proof it works end to end

![id and klist as a domain user](docs/screenshots/domain-login-id-klist.png)
*Real SSH login as `i.sredoevich`: forced password change on first login (FreeIPA
default), `id` resolving domain UID/GID/group via sssd, `klist` showing a real
Kerberos ticket obtained transparently at login.*

![sudo rule firing](docs/screenshots/sudo-rule-works.png)
*`sudo -l` shows the actual rule from the portal (`(root) ALL`), `sudo whoami` returns
`root`.*

![login denied for a user outside HBAC](docs/screenshots/login-denied-helpdesk.png)
*`m.ivanova` (helpdesk, not covered by the HBAC rule) enters the correct password and
the connection is simply closed — denied at the PAM/account phase, before a shell ever
exists.*

### Can the portal run in English?

Short answer: not in this build, at least not through anything user-facing. English
translation files genuinely exist on disk for every module
(`/opt/rbta/aldpro/mp/ui/app/*/locales/en/*.json`), and the frontend uses `i18next`
with the standard `localStorage.i18nextLng` key — but on every real page load the app
resets that key back to `"ru"` regardless of what's stored beforehand, a `?lng=en`
query string, or the browser's own `navigator.language` (which was already `en-US` in
the headless Chromium used for these screenshots). There's no language item in the
header icons either — the four icons are a DC picker, "your user" (opens your own
profile card), light/dark theme, and help. Nothing in the user's LDAP record
(`ipa user-show admin --all`) carries a locale attribute either. So the English
strings are bundled in the product but the switch isn't wired up anywhere I could
find — the portal is effectively Russian-only in practice.

## Why this matters

Standing up a directory is easy to claim and hard to fake. This repo shows the whole
loop on the real product: two VMs installed unattended from a stock Astra ISO, a
domain controller promoted by the book, a client joined and *proven* — login as an
allowed user, denial of a disallowed one, `klist` showing a real Kerberos ticket, a
sudo rule actually firing — with the command output and screenshots to back it up, plus
an honest log of what broke along the way and how it got fixed (below).

---

# Deployment log

Detailed notes on standing this up on real VMs: preseed broke on dialogs that don't
exist in a standard Debian preseed several times over, `aldpro-server-install` itself
failed five times in a row at different steps because of an internal race condition,
and the graphical `aldpro-client-installer` turned out to be unusable without a real
screen. None of this was visible from just reading the documentation — only from
actually running it.

## 1. Preseed-based unattended install of Astra Linux SE 1.7.7 — eight bugs in a row

Astra's installer is a classic debian-installer (isolinux + `install.amd/{vmlinuz,
initrd.gz}`, an `Automated install` menu entry with `auto=true priority=critical`).
The preseed example in the ALD Pro documentation (Part 2, "Roles and site services →
Automation → Boot profiles → Preseed") works as a base, but it's built for a PXE
scenario with a fully network-based debootstrap. Here I had a static IP with no DHCP
at first boot, plus an attached ISO. Here are all eight bugs, each one only surfacing
once the previous one was fixed:

1. **`astra-license/license`** — the Astra Linux end-user license agreement. There's
   no key for this in a normal Debian preseed — it's Astra-specific. Found it by
   unpacking the .udeb package (`ar x astra-license_*.udeb`, the `debian/templates`
   file).
2. **`cdrom-detect/load_media`** — "No common CD-ROM drive was detected." This
   happened because `--location` in virt-install pointed at a mounted directory
   instead of the .iso file itself — in that case virt-install doesn't attach the ISO
   as a virtual CD at all. Fix: `--location <path>.iso` instead of a mounted
   directory.
3. **The language-selection dialog** — pops up before the network preseed.cfg is even
   fetched (the network isn't up yet at that point). Fix: `locale=en_US.UTF-8` right
   on the kernel cmdline (`--extra-args`), not inside the preseed file.
4. **`debootstrap` over the network fails with exit 1** against `repository-base`.
   Switched to the officially documented offline path instead: since the ISO is
   already physically attached, the base system installs straight from the CD
   (`apt-setup/cdrom/set-first boolean true`), and the ALD Pro network repositories
   get added after the OS boots, over SSH.
5. **`apt-setup/cdrom/set-double`** (defaults to `true`, and I hadn't set it
   explicitly) — after a successful CD-based debootstrap, the installer asks "Scan
   another CD or DVD?" and hangs there. Found the key by unpacking
   `apt-cdrom-setup_*.udeb`.
6. **GRUB password** — forgot `grub-installer/password` in the first version of the
   preseed.
7. **`late_command` can't use `wget`** — on a base system installed only from CD
   (no network debootstrap), `wget` isn't part of the default package set. The
   installer just showed a generic "Failed to run preseeded command" with no
   explanation. Fix: rewrite every postinstall step (SSH access, `/etc/hosts`,
   disabling NetworkManager) as `in-target bash -c '...'` with no external
   dependencies.
8. **A one-off virtual CD read glitch** ("Debootstrap warning: ... was corrupt").
   Checking the file's md5sum against the ISO's own `md5sum.txt` confirmed the file
   itself wasn't corrupt — this was a transient IDE CD-ROM emulation hiccup in QEMU,
   not a problem with the distro. The serial console was configured write-only (no
   interactive pty), so there was nothing to click "Continue" with — just reran the
   install from scratch, and the glitch didn't come back.

Separately: `late_command` tried to call `in-target hostnamectl set-hostname
ald-dc.lab.local` — that failed, because `in-target` is just a chroot, and
`hostnamectl` talks to `systemd-hostnamed` over D-Bus, which doesn't exist in a
chroot. Fix: write `/etc/hostname` directly (`printf ... > /etc/hostname`).

Telling "quietly working" apart from "actually hung" on the installer's raw serial
text output (ANSI/VT100 escape sequences dumped into a log file) — the way that
actually works is to replay the log through a real terminal emulator:

```bash
tmux new-session -d -s astra -x 80 -y 24 "cat ald-dc-console.log; sleep 3600"
tmux capture-pane -t astra -p
```

## 2. A race condition inside `aldpro-server-install` — a systemic product issue

After a clean OS install, raising the security level to "Смоленск" (max), and wiring
up the repositories, the command `aldpro-server-install -d lab.local -n ald-dc --ip
192.168.150.10 --setup_gc --setup_syncer --no-reboot` **failed five times in a row**,
each time at a different step, with the same error:

```
Exception: Произошла ошибка. Превышено максимальное количество попыток.
Пожалуйста, попробуйте выполнить команду повторно.
```
("An error occurred. Maximum number of attempts exceeded. Please retry the command.")

The failures landed on `saltutil.sync_all`, `grains.set is_first_dc`, a hang on
`grains.delkey is_first_dc` (35+ minutes at 100% CPU — a genuine hang, not just slow:
the same command runs in 1-2 seconds by hand), and `state.apply utils.dc_passwords`.
And **every** failed step, rerun by hand right after the failure, completed in
seconds with no error at all. Conclusion: it's not any one step — `aldpro-server-install`
fires `aldpro-salt-call` invocations back to back with no pause, and the standalone
salt-minion doesn't always release its local lock in time between calls. Even the
script's own error handler can hang: when it tries to log the failure to LDAP via
`send_log_to_ldap`, it reaches for SSSD/D-Bus, which doesn't physically exist yet at
this stage of the install.

The sixth attempt ended in an actual Python 3.7 interpreter **segfault**, hitting
several celery workers at once (`dmesg`: `segfault at ... in python3.7`) — not OOM
(there was plenty of free memory), but not enough cores under parallel load on 2
vCPUs. **Bumping the DC to 4 vCPUs fixed it completely** — the seventh attempt ran
start to finish with zero failures (~10 minutes: FreeIPA/Kerberos/DNS, the ALD Pro
portal, Global Catalog).

Side finding: `aldproctl status` crashed with an `IndexError`, because
`socket.gethostname()` (the raw `hostname`, not `hostname -f`) only returned the
short name. Preseed via `netcfg/get_hostname` + `netcfg/get_domain` only writes the
short name into `/etc/hostname` — but the ALD Pro documentation (2.1.1.3) explicitly
requires `hostnamectl set-hostname <FQDN>`. Fix: write the full FQDN into
`/etc/hostname` directly.

**The tactic that actually saved the day:** take an offline disk snapshot right after
the OS install, wiring up the repositories, and installing the ALD Pro packages
(before `aldpro-server-install`), and just roll back to it on failure instead of
manually cleaning up state on the fly:

```bash
virsh snapshot-create-as ald-dc pre-server-install --disk-only --atomic
# on failure -- roll back and retry:
virsh destroy ald-dc
rm /var/lib/libvirt/images/ald-dc.pre-server-install
qemu-img create -f qcow2 -b ald-dc.qcow2 -F qcow2 ald-dc.pre-server-install
virsh start ald-dc
```

## 3. The graphical `aldpro-client-installer` doesn't work without a real screen

The documentation presents it as a "command-line method," but the binary is a
PyQt5/PyInstaller app that *always* needs a Qt platform:

- Without `DISPLAY` and without `--gui` it just crashes: `qt.qpa.plugin: Could not
  load the Qt platform plugin "xcb"`.
- With `QT_QPA_PLATFORM=offscreen` it doesn't crash, but it hangs dead: the process's
  only open file descriptor is an `eventfd` (the Qt event loop) — it never opens a
  single network connection to the domain controller.
- With a real `Xvfb` — the screen just shows an empty form: the "Domain name",
  "Account", and "Password" fields never get filled in from
  `--domain`/`--account`/`--password`. Command-line values simply aren't wired into
  GUI mode in this 3.0.0 build.

The fix is to skip the GUI wrapper entirely and use the underlying CLI directly,
`astra-freeipa-client` (the same tool the documentation uses for un-enrolling a
computer, via `-U`):

```bash
sudo astra-freeipa-client -d lab.local -u admin -p 'PASSWORD' -y
```

Worked on the first try: got the CA certificate, registered the host's SSH keys, and
configured SSSD/Kerberos/PAM — `The ipa-client-install command was successful`.

## 4. `ipa hbactest` is broken in ALD Pro 3.0.0 for plain local users

After creating an HBAC rule and disabling the default `allow_all` (otherwise the rule
has no effect at all), `ipa hbactest --user=i.sredoevich --host=... --service=sshd`
came back with `ipa: ERROR: an internal error has occurred`. The full traceback showed
up in `/var/log/apache2/error.log`:

```
File ".../ipaserver/plugins/hbactest.py", line 394, in execute
    is_valid_sid = ipaserver.dcerpc.is_sid_valid(options['user'])
File ".../ipaserver/dcerpc.py", line 89, in is_sid_valid
    security.dom_sid(sid)
ValueError: Unable to parse string: 'i.sredoevich'
```

`hbactest` in this build unconditionally tries to parse the given user as a Windows
SID (code meant for MS AD trust scenarios) and crashes instead of first checking
whether the string even looks like a SID. The workaround is to not rely on the
simulation test at all, and instead prove allow/deny with real login attempts (which
was already required anyway).

## 5. HBAC allows login but not `sudo` — they're two separate PAM services

After setting up the sudo rule, `sudo -l` for an allowed user failed with `sudo: PAM
account management error: Permission denied` — not a denial from the sudo rule
itself, but an HBAC block at the PAM layer: the `allow_developers` rule allowed the
`sshd`/`login` services, but not `sudo`. SSSD's HBAC gates *every* PAM service
separately, `sudo` included. Fix:

```bash
ipa hbacrule-add-service allow_developers --hbacsvcs=sudo
ipa hbacrule-add-service allow_developers --hbacsvcs=sudo-i
```

After that, `sudo -l` showed the real rule (`(root) ALL`), and `sudo whoami` returned
`root`.

## Bottom line

The full list of what's actually proven live is in the ["Proof, not just claims"](#proof-not-just-claims)
section above. Every line there is a real command run on a real pair of VMs, not text
copied from the documentation.

---

# Scaling up: a 200-user enterprise

This section documents the organizational and access-control structure for phase
two: the same ALD Pro domain, scaled to a size that actually exercises what a
directory is for — ~200 people, ~150 machines, multiple departments, multiple sites,
a tiered support desk, and access rules that hold up under real organizational
politics (directors want broad visibility, HR needs privacy, tech support needs
escalation tiers, retail floors need lockdown).

This covers departments, headcounts, sites, the OU tree, the group/permission
matrix, and naming conventions. The build (VMs, domain promotion, client rollout,
GPOs, DNS/HTTPS/file services) is tracked and proven the same honest way as the
two-VM domain above — real screenshots, real command output, an honest log of what
broke.

## The company

> **Note:** company name and domain are changed for confidentiality — "ТехноЛайн" /
> `technoline.local` stand in for the real organization's name throughout this
> document and the infrastructure built for it.

**ООО «ТехноЛайн»** ("TechnoLine") — a consumer electronics and appliance retailer:
one head office, one central warehouse, one service/repair center, and a chain of
retail stores across several cities.

| Metric | Value |
|---|---|
| Domain | `technoline.local` (NetBIOS `TECHNOLINE`) |
| Employees | ~200 |
| Workstations/servers | ~150 |
| Sites | HQ + Warehouse + Service Center + 7 retail stores (10 sites total) |
| Email convention | `f.lastname@technoline.local` (first initial + dot + last name) |

**Why 150 machines for 200 people, not 1:1:** retail floor staff and warehouse
pickers work in shifts off shared POS terminals and handheld/kiosk stations rather
than personal desktops. Every office role (HQ, warehouse office, service center
back-office) gets its own machine; retail stores get 4-6 shared workstations each
regardless of headcount. That's also a realistic reason to have host-based access
control (HBAC) do real work: the same physical machine is used by different people
on different shifts, so *the directory*, not the machine, is what decides what a
given login can do.

## Departments and headcount

| Department (RU) | Department (EN) | Site | Headcount |
|---|---|---|---|
| Дирекция | Executive Management | HQ | 5 |
| ИТ-отдел | IT Department | HQ | 10 |
| Отдел продаж (корп.) | Corporate/B2B Sales | HQ | 10 |
| Маркетинг | Marketing | HQ | 6 |
| Отдел закупок | Procurement | HQ | 8 |
| Бухгалтерия и финансы | Accounting & Finance | HQ | 12 |
| Отдел кадров | HR | HQ | 6 |
| Юридический отдел | Legal | HQ | 4 |
| Служба безопасности | Security | HQ | 6 |
| АХО | Facilities/Admin-Economic | HQ | 8 |
| Склад и логистика | Warehouse & Logistics | Warehouse | 25 |
| Сервисный центр | Service Center | Service Center | 15 |
| Розничная сеть (7 магазинов) | Retail (7 stores) | Stores 1-7 | ~84 (≈12/store) |
| **Total** | | | **~199** |

### IT department breakdown (this is the group that gets modeled in the most detail)

| Role (RU) | Role (EN) | Count | Reports to |
|---|---|---|---|
| ИТ-директор | IT Director | 1 | Генеральный директор |
| Системный администратор | Systems Administrator (domain admin) | 2 | IT Director |
| Инженер 3-й линии | Support Engineer — Line 3 | 2 | IT Director |
| Специалист 2-й линии | Support Specialist — Line 2 | 2 | Line 3 lead |
| Специалист 1-й линии | Support Specialist — Line 1 (helpdesk) | 3 | Line 2 lead |

Escalation model: **L1 → L2 → L3 → Sysadmin**. Each tier can do everything the tier
below it can, plus more — modeled as nested ALD Pro/FreeIPA groups
(`support-l3` ⊃ `support-l2` ⊃ `support-l1`), not copy-pasted rule sets.

## Sites and network plan

| Site | Subnet (planned) | Role |
|---|---|---|
| Core / servers | `10.10.0.0/24` | DC1 (primary), DC2 (backup/replica), DNS, file server, web/portal |
| HQ | `10.10.1.0/24` | ~65 workstations (management, IT, sales, marketing, procurement, finance, HR, legal, security, facilities) |
| Warehouse | `10.10.2.0/24` | ~15 workstations + handheld/WMS terminals |
| Service Center | `10.10.3.0/24` | ~10 workstations (repair intake, diagnostics, parts) |
| Retail Store 1-7 | `10.10.10.0/24` – `10.10.16.0/24` | ~5-6 shared POS/back-office workstations per store |

Each site VPNs/routes back to the core over per-site VLAN-style subnets, same
network model as the `aldlab` network in the two-VM domain above, just scaled out.
Two domain controllers instead of one, both because 200 users genuinely warrants it
and because it lets group policy / replication get exercised for real.

## Directory (OU) tree

```
dc=technoline,dc=local
└── ou=TechnoLine
    ├── ou=HQ
    │   ├── ou=Management            (Дирекция)
    │   ├── ou=IT
    │   │   ├── ou=SysAdmins
    │   │   └── ou=Support
    │   │       ├── ou=Line1
    │   │       ├── ou=Line2
    │   │       └── ou=Line3
    │   ├── ou=Sales
    │   ├── ou=Marketing
    │   ├── ou=Procurement
    │   ├── ou=Finance
    │   ├── ou=HR
    │   ├── ou=Legal
    │   ├── ou=Security
    │   └── ou=Facilities
    ├── ou=Warehouse
    ├── ou=ServiceCenter
    ├── ou=Retail
    │   ├── ou=Store-01 ... ou=Store-07
    ├── ou=Computers
    │   ├── ou=HQ
    │   ├── ou=Warehouse
    │   ├── ou=ServiceCenter
    │   └── ou=Retail
    └── ou=ServiceAccounts
```

Users live under their department OU; computers live under a parallel
`ou=Computers` tree by site, not mixed in with user objects — that's what makes
per-site computer-group HBAC/sudo/GPO scoping clean later (e.g. "everyone in
`ou=Retail` gets the kiosk-lockdown GPO" without touching user objects at all).

## Groups and access matrix

Every row below is a real ALD Pro/FreeIPA group with actual HBAC + sudo rules behind
it, not just a label — this table *is* the spec for the rules that get created in
the build phase.

| Group | Members | Login rights | Sudo / admin rights | Data access |
|---|---|---|---|---|
| `domain-admins` | 2 senior sysadmins only | Everywhere | Full `sudo` on all hosts, ALD Pro portal full control | Everything (used for admin tasks only — no daily-driver use) |
| `it-directors` | IT Director | HQ hosts + read-only portal access everywhere | No `sudo` by default (separation of duties — approves access, doesn't grant it themself) | Read access to IT audit logs, no HR/finance data |
| `support-l3` | Line-3 engineers | HQ + all sites (remote) | `sudo` on servers (DNS, DHCP, file/web services) *except* domain-admin actions; full local admin on any workstation | IT ticketing system, network device configs |
| `support-l2` | Line-2 specialists (⊃ inherits L1) | HQ + all sites (remote) | Local admin on workstations (domain join/leave, software installs, printers); no server `sudo` | IT ticketing system |
| `support-l1` | Line-1 helpdesk | HQ only | Delegated password reset + account unlock only (via FreeIPA self-service/delegation ACI, not full admin); no local admin | IT ticketing system, read-only user directory |
| `executives` | Директор, замы | Own workstation + read access to department shares relevant to oversight (finance dir → accounting share, commercial dir → sales/CRM) | Local admin on **own** workstation only | Cross-department reporting views |
| `hr-staff` | HR department | HQ | None | Personnel records (own department only — cannot see finance/accounting) |
| `hr-managers` | HR lead(s), subset of `hr-staff` | HQ | None | Salary bands, hiring/termination records |
| `finance-staff` | Accounting & Finance | HQ | None | 1C:Бухгалтерия, financial shares |
| `sales-staff` | Corporate sales + retail sales consultants | HQ / assigned store | None | CRM / 1C:Розница, no financial or HR data |
| `warehouse-staff` | Warehouse & Logistics | Warehouse | None | WMS, shipment tracking |
| `service-staff` | Service Center technicians | Service Center | None | Repair ticketing, parts inventory |
| `retail-staff` | Store sales consultants, cashiers | Assigned store only | None | POS system only; GPO-locked desktop (no Control Panel, no USB storage, kiosk shell) |
| `store-managers` | One per store, subset of `retail-staff` | Assigned store | Local admin on that store's back-office workstation only | Store sales reports, local staff schedule |
| `marketing-staff` | Marketing | HQ | None | Shared marketing drive, website CMS |
| `procurement-staff` | Procurement | HQ | None | Supplier contracts, purchase orders |
| `legal-staff` | Legal | HQ | None | Contract repository |
| `security-staff` | Security | HQ | Read access to authentication/audit logs | Access-control and audit systems (mostly outside ALD Pro's scope) |
| `facilities-staff` | АХО | HQ | None | Facilities shares only |

Design rules this table encodes (worth stating explicitly, since they're what a
reviewer would actually check for):

- **No standing local-admin for end users**, except two narrow, deliberate
  exceptions: executives on their own single machine, and store managers on their
  own store's back-office machine — both scoped to exactly one host, not "admin
  everywhere."
- **Support tiers are additive groups, not duplicated rule sets** — `support-l3`
  nests `support-l2` nests `support-l1`, so a Line-3 engineer is never missing a
  Line-1 permission by accident.
- **HR and Finance are mutually exclusive by default** — HR can't read salary
  postings in accounting shares, Finance can't read personnel files. Any legitimate
  cross-access (e.g. payroll) goes through a dedicated `payroll-access` group with
  named members from both sides, not a blanket department merge.
- **`it-directors` is deliberately not in `domain-admins`** — oversight and
  execution are different rights. This is the same separation-of-duties point that
  came up for real in the two-VM lab's HBAC-vs-sudo bug above (allowing login is not
  the same grant as allowing `sudo`); here it's applied on purpose at the org level
  instead of being discovered as a bug.
- **Retail is the most locked-down tier** — shared machines, high staff turnover,
  and public-facing floor space make it the highest-risk tier despite having the
  lowest privileges, which is exactly why GPO desktop lockdown matters most there.

## Naming conventions

- **Users:** `f.lastname` login (e.g. `i.ivanov`), email `f.lastname@technoline.local`.
- **Computers:** `<site>-<role>-<seq>`, e.g. `hq-it-03`, `wh-wms-01`, `st07-pos-02`.
- **Groups:** lowercase, hyphenated, role-first (`support-l1`, `hr-managers`), so
  `id`/`groups` output reads naturally.
- **OUs:** PascalCase, matching the department table above.

## Rollout plan

The build phase — tracked separately as it happens — covers:

- Two domain controllers (`dc1`, `dc2`) + DNS + file/web services, promoted and
  proven the same way as the two-VM lab above (real commands, real screenshots).
- Client VMs created, joined to the right OU, and rotated through the build
  environment to populate ~150 computer objects and ~200 user accounts without
  needing 150 VMs running at once.
- Group Policy per OU (desktop lockdown for retail, software deployment per
  department, password policy, NTP).
- The full HBAC/sudo rule set implementing the access matrix above.
- Verification: prove login/deny per role, prove L1 can't do what L3 can, prove HR
  can't read Finance's share, prove a retail kiosk actually stays locked down.
