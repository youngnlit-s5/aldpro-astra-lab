# Runbook — deploy ald-dc / ald-cl1 and prove the domain works

Host: libvirt/KVM, user in the `kvm` + `libvirt` groups (no root shell needed except
for the few `sudo` lines called out below). Everything else in this runbook is
`virsh`/`virt-install` against `qemu:///system`.

This is the sequence that actually worked, after fixing the bugs described in the
"Журнал развёртывания" section of the main [README](../README.md). If you just want
the finished, debugged commands, they're all below.

## 0. Network

One isolated NAT network, `aldlab` (192.168.150.0/24), gateway/DNS-during-install at
`192.168.150.1` (the host), with DHCP host reservations matching the two VMs' MACs so
the "static" addresses are also what an initial DHCP handshake hands out:

```bash
virsh net-define net/aldlab-net.xml
virsh net-autostart aldlab
virsh net-start aldlab
```

| Host | MAC | IP | Role |
|---|---|---|---|
| ald-dc | 52:54:00:15:00:10 | 192.168.150.10 | Domain controller, **4 vCPU** / 8GB RAM |
| ald-cl1 | 52:54:00:15:00:20 | 192.168.150.20 | Domain client, 2 vCPU / 4GB RAM, full Fly desktop |

**4 vCPU on the DC is not optional** — with 2 vCPU, `aldpro-server-install` triggered
real Python segfaults under parallel salt/celery load (see journal, §2).

## 1. Unattended OS install (preseed)

Astra's installer is a stock debian-installer (isolinux, `install.amd/{vmlinuz,initrd.gz}`,
an `auto=true priority=critical` boot entry already on the ISO). Serve the preseed
files over HTTP so `preseed/url=` can fetch them from inside the VM:

```bash
cd preseed && python3 -m http.server 8080 --bind 192.168.150.1 &
```

Boot straight off the ISO **file** (not a mounted directory — otherwise virt-install
won't attach it as a virtual CD-ROM, which the installer needs even for a network
install):

```bash
virt-install \
  --connect qemu:///system --name ald-dc \
  --memory 8192 --vcpus 4 \
  --disk pool=default,size=32,format=qcow2,bus=virtio \
  --network network=aldlab,mac=52:54:00:15:00:10,model=virtio \
  --graphics none --console pty,target_type=serial \
  --serial file,path=ald-dc-console.log \
  --os-variant debian12 \
  --location installation-1.7.7.9-24.04.25_19.28.iso,kernel=install.amd/vmlinuz,initrd=install.amd/initrd.gz \
  --extra-args "auto=true priority=critical locale=en_US.UTF-8 astra_install=1 \
    astra-license/license=true cdrom-detect/load_media=true \
    netcfg/disable_autoconfig=false net.ifnames=0 \
    preseed/url=http://192.168.150.1:8080/preseed-dc.cfg \
    console=ttyS0,115200n8 quiet" \
  --noautoconsole --wait -1
```

Same command for `ald-cl1` with `mac=52:54:00:15:00:20`, `--memory 4096 --vcpus 2` and
`preseed-cl1.cfg`. Watch progress by replaying the serial log through a real terminal
emulator (it's raw ANSI/VT100, not plain text):

```bash
tmux new-session -d -s astra -x 80 -y 24 "cat ald-dc-console.log; sleep 3600"
tmux capture-pane -t astra -p
```

## 2. Promote ald-dc to a domain controller

Over SSH as `labadmin` (created by the preseed), following *Руководство администратора
ALD Pro, часть 1, раздел 2.1* exactly:

```bash
# security level must be "Смоленск" (max) before installing ALD Pro
sudo astra-modeswitch getname
sudo astra-modeswitch set 2 && sudo reboot

# repositories (frozen branch -- required for ALD Pro compatibility, NOT stable)
sudo tee /etc/apt/sources.list <<'EOF'
deb http://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-main 1.7_x86-64 main non-free contrib
deb http://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-update 1.7_x86-64 main contrib non-free
EOF
sudo tee /etc/apt/sources.list.d/aldpro.list <<'EOF'
deb https://dl.astralinux.ru/aldpro/frozen/01/3.0.0/ 1.7_x86-64 main base
EOF
sudo apt update && sudo apt dist-upgrade -y -o Dpkg::Options::=--force-confold

sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -q \
  aldpro-mp aldpro-gc aldpro-syncer aldpro-syncer-pwdsync
```

**Snapshot here**, before running `aldpro-server-install` — it took us five attempts
to get through it cleanly (see journal, §2), and every retry is safe from this point:

```bash
sudo shutdown -h now   # on the guest
virsh snapshot-create-as ald-dc pre-server-install --disk-only --atomic
virsh start ald-dc
```

Promote the domain controller:

```bash
sudo aldpro-server-install -d lab.local -n ald-dc --ip 192.168.150.10 \
  --setup_gc --setup_syncer --no-reboot -p 'ПАРОЛЬ_АДМИНА_ДОМЕНА'
```

If it fails with `Превышено максимальное количество попыток` — that's a known salt-call
race in this product version, not a config error. Clean up and retry the exact same
command:

```bash
# if it hung on grains.delkey (100% CPU, no progress for minutes): kill and clear
sudo pkill -9 -f aldpro-salt-call
sudo sed -i '/^is_first_dc:/d' /srv/aldpro-salt-master/config/grains
# then just re-run aldpro-server-install -- it resumes further each time
```

Or, faster: revert to the `pre-server-install` snapshot and try again from a
guaranteed-clean state:

```bash
virsh destroy ald-dc
rm /var/lib/libvirt/images/ald-dc.pre-server-install
qemu-img create -f qcow2 -b ald-dc.qcow2 -F qcow2 ald-dc.pre-server-install
virsh start ald-dc
```

After it succeeds and reboots, fix the FQDN hostname if `aldproctl` crashes with
`IndexError` (preseed only sets the short name):

```bash
sudo hostnamectl set-hostname ald-dc.lab.local   # needs a running system, not chroot
sudo aldproctl status
```
📸 [`docs/screenshots/aldpro-portal-dashboard.png`](screenshots/aldpro-portal-dashboard.png) ·
output saved to [`docs/logs/aldproctl-status.txt`](logs/aldproctl-status.txt)

DNS hardening from *2.1.4* (disable DNSSEC validation, allow recursion/cache from the
lab subnet):

```bash
sudo tee /etc/bind/ipa-options-ext.conf <<'EOF'
allow-recursion { any; };
allow-query-cache { any; };
dnssec-validation no;
EOF
sudo named-checkconf /etc/bind/named.conf
sudo systemctl restart bind9-pkcs11.service
```

## 3. Provision groups, users, sudo and HBAC rules

Via `ipa` CLI (ALD Pro is FreeIPA underneath, so the FreeIPA CLI works exactly as
documented in *часть 2*):

```bash
kinit admin
ipa group-add developers --desc='App developers'
ipa group-add helpdesk --desc='Helpdesk / first-line support'
ipa user-add i.sredoevich --first=Ilya --last=Sredoevich --password
ipa user-add m.ivanova   --first=Maria --last=Ivanova   --password
ipa group-add-member developers --users=i.sredoevich
ipa group-add-member helpdesk   --users=m.ivanova

# disable the default allow-everyone rule, or your own HBAC rule below does nothing
ipa hbacrule-disable allow_all

ipa hbacrule-add allow_developers --desc='Login access for developers group'
ipa hbacrule-add-user allow_developers --groups=developers
ipa hbacrule-mod allow_developers --hostcat=all
# HBAC gates sudo as its own PAM service -- without these, sudo fails with
# "PAM account management error" even with a valid sudo rule (see journal, §5)
ipa hbacrule-add-service allow_developers --hbacsvcs=sshd
ipa hbacrule-add-service allow_developers --hbacsvcs=login
ipa hbacrule-add-service allow_developers --hbacsvcs=sudo
ipa hbacrule-add-service allow_developers --hbacsvcs=sudo-i

ipa sudorule-add allow-developers-full --desc='Full sudo for developers group'
ipa sudorule-mod allow-developers-full --hostcat=all
ipa sudorule-mod allow-developers-full --cmdcat=all
ipa sudorule-add-user allow-developers-full --groups=developers
```
📸 [`docs/screenshots/aldpro-portal-users.png`](screenshots/aldpro-portal-users.png),
[`aldpro-portal-groups.png`](screenshots/aldpro-portal-groups.png),
[`aldpro-portal-sudo-rules.png`](screenshots/aldpro-portal-sudo-rules.png)

## 4. Join ald-cl1 to the domain

Point the client at the DC for DNS (*2.2.1*):

```bash
sudo tee /etc/resolv.conf <<'EOF'
domain lab.local
search lab.local
nameserver 192.168.150.10
EOF
ping -c2 dl.astralinux.ru && ping -c2 ald-dc.lab.local

sudo apt-get install -y -q aldpro-client
```

**The documented GUI joiner (`aldpro-client-installer`) doesn't work headless** — it's
a PyQt5 app that needs a real display and, even then, doesn't accept `--domain`/
`--account`/`--password` as pre-filled form values in this build (see journal, §3).
Use the underlying FreeIPA client directly instead — same tool the docs use for
*un*enrolling (`-U`), just without that flag:

```bash
sudo astra-freeipa-client -d lab.local -u admin -p 'ПАРОЛЬ_АДМИНА_ДОМЕНА' -y
sudo reboot
```
📸 [`docs/screenshots/aldpro-portal-computers.png`](screenshots/aldpro-portal-computers.png)
(both `ald-dc` and `ald-cl1` listed after join)

## 5. Prove it actually works

```bash
# domain identity, not /etc/passwd
ssh i.sredoevich@ald-cl1.lab.local
id
klist          # real krbtgt + ldap service ticket, obtained at login
sudo -l         # "(root) ALL" -- the sudo rule from the portal, live
sudo whoami     # root
```
📸 [`docs/screenshots/domain-login-id-klist.png`](screenshots/domain-login-id-klist.png),
[`sudo-rule-works.png`](screenshots/sudo-rule-works.png)

```bash
# a user outside the HBAC rule: connection is closed at PAM/account phase,
# before a shell ever exists -- not just "sudo denied"
ssh m.ivanova@ald-cl1.lab.local
# Connection closed by 192.168.150.20 port 22
```
📸 [`docs/screenshots/login-denied-helpdesk.png`](screenshots/login-denied-helpdesk.png) ·
confirmed server-side in [`auth.log`](logs/login-deny-authlog.txt):
`pam_sss(sshd:account): Access denied for user m.ivanova: 6 (Permission denied)`

Web portal (works from any browser once you cap TLS at 1.2 — the server is
`SSLProtocol TLSv1.2` only and modern browsers' GREASE extensions in the ClientHello
trip up this mod_ssl build; a plain `curl` negotiates it fine, Firefox/Chromium need
`--ssl-version-max=tls1.2` or the equivalent `security.tls.version.max` pref):

```
https://ald-dc.lab.local
```
📸 [`docs/screenshots/aldpro-portal-login.png`](screenshots/aldpro-portal-login.png)

## Teardown

```bash
virsh destroy ald-cl1 ; virsh undefine ald-cl1 --remove-all-storage
virsh destroy ald-dc  ; virsh undefine ald-dc  --remove-all-storage
virsh net-destroy aldlab ; virsh net-undefine aldlab
```
