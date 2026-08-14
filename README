# Setup Oracle Database 26ai RAC — 2 Node di KVM
Dokumentasi replikasi. Contoh di sini pakai environment `dsi-oracle26ai-rac` — ganti hostname/IP/nama sesuai server target.

## Status Sampai Titik Ini

| Tahap | Status |
|---|---|
| 1. Jaringan (public + private interconnect) | ✅ Selesai |
| 2. Shared Storage ASM | ✅ Selesai |
| 3. OS Preparation | ✅ Selesai |
| 4. Grid Infrastructure install | ✅ Selesai, cluster sehat, terverifikasi |
| 5. ASM Diskgroup DATA & RECO | ✅ Selesai, 3 diskgroup MOUNTED(2 of 2) |
| 6. Install Database software | ✅ Selesai |
| 7. Buat database (DBCA) | ✅ Selesai, instance jalan di kedua node, terverifikasi |

**Setup RAC 2-node selesai penuh.** Database `dsirac.ds-inovasi.com` (SID prefix `dsirac`, instance `dsirac1`/`dsirac2`), terverifikasi via `srvctl status database` dan `srvctl status scan_listener` — semua running normal.

---

## Prasyarat & Asumsi Environment

- 2 VM di **satu KVM host yang sama**, RHEL 9.x, minimal 2 vCPU / 16GB RAM per node
- Hypervisor punya bridge network untuk akses public (contoh: `bridgedsi`)
- Software Grid Infrastructure + Database sudah didownload dari edelivery.oracle.com (Linux x86-64)
- Cek dulu free RAM/CPU host sebelum provisioning — instalasi Grid (khususnya `root.sh`/DBCA) adalah fase paling boros memory

**Konfigurasi contoh yang dipakai di dokumen ini** (ganti sesuai target):

| Item | Node 1 | Node 2 |
|---|---|---|
| Domain name (virsh) | ORACLE26AI_RAC_1 | ORACLE26AI_RAC_2 |
| Hostname | oracle26ai-01.ds-inovasi.com | oracle26ai-02.ds-inovasi.com |
| Public IP | 192.168.90.216 | 192.168.90.230 |
| VIP | 192.168.90.217 | 192.168.90.98 |
| Private IP (interconnect) | 192.168.56.101 | 192.168.56.102 |
| SCAN | dsi-scan.ds-inovasi.com — 3 IP round-robin via DNS |
| ORACLE_HOME (Grid) | `/oracle/app/grid/26ai/grid_1` |
| ORACLE_HOME (DB) | `/oracle/app/oracle/product/26ai/db_1` |
| Database name (GDB / SID prefix) | `dsirac.ds-inovasi.com` / `dsirac` (instance `dsirac1`, `dsirac2`) — **catatan: hyphen tidak diizinkan di GDB name**, rencana awal `dsi-oracle26ai-rac` ditolak installer |

**Disk layout (kedua node identik):**

| Device | Ukuran | Fungsi |
|---|---|---|
| vda | 50 GB | OS |
| vdb | 100 GB | Data lokal non-shared (opsional, tidak wajib untuk RAC) |
| vdc | 5 GB | ASM diskgroup **OCRVOTE** (OCR + Voting) |
| vdd | 40 GB | ASM diskgroup **DATA** |
| vde | 20 GB | ASM diskgroup **RECO** |

Semua diskgroup ASM: redundancy **External** (karena 1 disk per diskgroup — ganti ke Normal/High kalau punya lebih dari 1 disk per diskgroup dan mau mirroring).

---

## 1. Jaringan

Private interconnect: isolated virtual network, tidak butuh NIC fisik tambahan kalau kedua VM di host yang sama.

```bash
# Di KVM host
virsh net-define /dev/stdin <<EOF
<network>
  <name>rac-priv</name>
  <bridge name='virbr-priv' stp='on' delay='0'/>
</network>
EOF
virsh net-start rac-priv
virsh net-autostart rac-priv

virsh attach-interface <domain-node1> network rac-priv --model virtio --live --config
virsh attach-interface <domain-node2> network rac-priv --model virtio --live --config
```

Di dalam guest, cek nama interface baru (`ip a`), set IP statis:
```bash
nmcli con add type ethernet ifname <ifname> con-name rac-priv ip4 <private-ip>/24
nmcli con up rac-priv
```

`/etc/hosts` di **kedua** node, identik (Public, VIP, Private — **SCAN tidak dimasukkan di sini**, resolve lewat DNS):
```
<public-ip-1>    <hostname-1> <short-1>
<public-ip-2>    <hostname-2> <short-2>
<vip-1>          <hostname-1>-vip
<vip-2>          <hostname-2>-vip
<priv-ip-1>      <hostname-1>-priv
<priv-ip-2>      <hostname-2>-priv
```

**SCAN via DNS** — wajib 3 A record dengan nama sama, round robin aktif (contoh Windows Server DNS):
```powershell
Add-DnsServerResourceRecordA -Name "<scan-name>" -ZoneName "<domain>" -IPv4Address "<ip-1>"
Add-DnsServerResourceRecordA -Name "<scan-name>" -ZoneName "<domain>" -IPv4Address "<ip-2>"
Add-DnsServerResourceRecordA -Name "<scan-name>" -ZoneName "<domain>" -IPv4Address "<ip-3>"
Set-DnsServerSetting -RoundRobin $true
```
3 IP SCAN wajib **berbeda** dari Public/VIP semua node, dan di subnet public (bukan private). Kalau tidak ada DNS server tersedia, boleh fallback 1 IP di `/etc/hosts`, tapi kehilangan fungsi failover SCAN yang sebenarnya.

---

## 2. Shared Storage untuk ASM

```bash
# Di KVM host — buat raw disk image di lokasi pilihan (bukan harus /var/lib/libvirt/images)
qemu-img create -f raw <path>/asm_ocr.img 5G
qemu-img create -f raw <path>/asm_data.img 40G
qemu-img create -f raw <path>/asm_reco.img 20G

# PENTING: kalau SELinux Enforcing di host dan path bukan default libvirt, relabel dulu:
getenforce
semanage fcontext -a -t svirt_image_t "<path>/asm_.*\.img"
restorecon -Rv <path>/asm_*.img

# Attach ke KEDUA VM, device target sesuaikan yang masih kosong (cek virsh domblklist dulu)
for vm in <domain-node1> <domain-node2>; do
virsh attach-device $vm /dev/stdin --config <<EOF
<disk type='file' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native'/>
  <source file='<path>/asm_ocr.img'/>
  <target dev='vdc' bus='virtio'/>
  <shareable/>
  <serial>asmocr01</serial>
</disk>
EOF
done
# ulangi untuk asm_data.img->vdd (serial asmdata01), asm_reco.img->vde (serial asmreco01)
```

Restart kedua VM (atau tambah `--live` di atas kalau file image sudah pasti ada saat attach). Verifikasi di guest — device path & ukuran harus identik di kedua node:
```bash
lsblk
```

Ownership device di guest — **group harus `asmdba`**, bukan `asmadmin` (beda dari intuisi umum, CVU expect ini), di kedua node:
```bash
cat > /etc/udev/rules.d/99-asm-disks.rules <<'EOF'
KERNEL=="vdc", OWNER="grid", GROUP="asmdba", MODE="0660"
KERNEL=="vdd", OWNER="grid", GROUP="asmdba", MODE="0660"
KERNEL=="vde", OWNER="grid", GROUP="asmdba", MODE="0660"
EOF
udevadm control --reload-rules
udevadm trigger
ls -la /dev/vdc /dev/vdd /dev/vde   # harus grid:asmdba
```

---

## 3. OS Preparation (kedua node, identik)

`oracle-database-preinstall-*` RPM biasanya **tidak tersedia di RHEL** (paket khas Oracle Linux, repo `yum.oracle.com` kerap 404) — setup manual:

```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 asmadmin
groupadd -g 54324 asmdba
groupadd -g 54325 asmoper
useradd -u 54331 -g oinstall -G asmadmin,asmdba,asmoper grid
useradd -u 54332 -g oinstall -G dba,asmdba oracle
passwd grid
passwd oracle
```

Kernel parameters:
```bash
cat >> /etc/sysctl.d/98-oracle.conf <<EOF
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 4398046511104
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
EOF
sysctl --system
```

Resource limits — **termasuk hard limit stack**, jangan cuma soft (prerequisite check Grid akan gagal kalau cuma soft):
```bash
cat >> /etc/security/limits.d/98-oracle.conf <<EOF
grid soft nproc 16384
grid hard nproc 16384
grid soft nofile 1024
grid hard nofile 65536
grid soft stack 10240
grid hard stack 10240
oracle soft nproc 16384
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 10240
EOF
```
Setelah edit limits.conf, **buka sesi shell/SSH baru** sebelum lanjut instalasi — perubahan tidak berlaku di sesi yang sudah terbuka.

Disable avahi-daemon (wajib — CVU minta service ini **mati**, bukan nyala):
```bash
systemctl stop avahi-daemon
systemctl disable avahi-daemon
systemctl mask avahi-daemon
```

Firewalld off, SELinux guest permissive (lab), chrony aktif:
```bash
systemctl disable --now firewalld
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
setenforce 0
systemctl enable --now chronyd
```

Ownership folder software (ganti `/oracle` dengan mount/path pilihan kamu — pastikan chown di level **parent**, bukan cuma subfolder):
```bash
mkdir -p /oracle/app/grid/26ai/grid_1 /oracle/stage
chown -R grid:oinstall /oracle/app
chmod 775 /oracle/app
mkdir -p /oracle/app/oracle/product/26ai/db_1
chown -R oracle:oinstall /oracle/app/oracle
```

Passwordless SSH grid↔grid dan oracle↔oracle antar kedua node — **termasuk self-loop** (node ke dirinya sendiri), wajib sebelum gridSetup.sh:
```bash
su - grid
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id grid@<hostname-1>
ssh-copy-id grid@<hostname-2>
# ulangi di node lain, dan untuk user oracle
```

VNC untuk jalankan installer GUI (minimal package, bukan full desktop):
```bash
dnf install -y xterm tigervnc-server xorg-x11-fonts-Type1 xorg-x11-twm
mkdir -p /etc/tigervnc
echo ":1=grid" > /etc/tigervnc/vncserver.users
su - grid -c vncpasswd
systemctl daemon-reload
systemctl enable --now vncserver@:1.service
```
Connect VNC client ke `<public-ip>:5901`.

---

## 4. Install Grid Infrastructure

```bash
su - grid
cd /oracle/app/grid/26ai/grid_1   # extract zip Grid HANYA di node ini, node2 dibiarkan kosong
export DISPLAY=:1
./gridSetup.sh
```

**Pilihan kunci di wizard, urut:**
1. "Configure Oracle Grid Infrastructure for a New Cluster"
2. Cluster Name bebas, SCAN Name = FQDN yang sudah didaftarkan di DNS (bukan nama default auto-generate)
3. Cluster Node Information → **Add** node kedua manual (Public + Virtual hostname), lalu klik **"SSH connectivity..."** untuk verifikasi sebelum Next
4. Network Interface Usage → interface public = Public, interface private = "ASM & Private" (default installer, tidak perlu diubah)
5. Storage Option → "Use Oracle Flex ASM for storage"
6. Create ASM Disk Group → **ganti Discovery Path ke `/dev/vd*`** (default `/dev/sd*` tidak cocok device virtio) → nama diskgroup untuk OCR/Voting (contoh `OCRVOTE`, bukan `DATA` — hindari bentrok nama dengan diskgroup data nanti) → redundancy **External** (kalau cuma 1 disk)
7. Automatic Self Correction → biarkan tidak dicentang (untuk lab/troubleshooting manual)
8. Failure Isolation (IPMI) → "Do not use" (VM tidak punya BMC fisik)
9. Management Options → jangan centang Register EM Cloud Control
10. Operating System Groups → asmadmin/asmdba sudah auto-detect, OSOPER boleh kosong
11. Installation Location → Oracle base & software location sesuai path yang sudah di-chown
12. Root script configuration → **jangan centang "Automatically run configuration scripts"** — jalankan `root.sh` manual (lebih mudah diagnosa kalau gagal)
13. Prerequisite Checks → perbaiki semua item **Failed** sebelum lanjut (lihat Lampiran untuk pola kegagalan umum), pakai "Fix & Check Again" untuk yang Fixable=Yes, manual untuk sisanya, lalu "Check Again"
14. Install Product — proses lama, jangan interupsi. Kalau ada prompt jalankan `root.sh` manual, buka terminal root terpisah di **masing-masing node**, jalankan **node1 dulu, tunggu selesai, baru node2**:
```bash
/oracle/app/grid/26ai/grid_1/root.sh
```

**Verifikasi setelah selesai** (set PATH dulu kalau `crsctl` "command not found"):
```bash
export ORACLE_HOME=/oracle/app/grid/26ai/grid_1
export PATH=$ORACLE_HOME/bin:$PATH
crsctl check cluster -all
crsctl stat res -t
```
Semua resource inti (`ora.asm`, diskgroup OCR, listener, VIP, 3× SCAN vip/listener) harus `ONLINE` di kedua node. Resource seperti `ora.cdp*.cdp`, `ora.helper`, `ora.rhpserver` yang `OFFLINE` itu normal (on-demand, bukan kegagalan).

Simpan permanen ke `.bash_profile` user `grid` di kedua node:
```bash
cat >> ~/.bash_profile <<EOF
export ORACLE_HOME=/oracle/app/grid/26ai/grid_1
export PATH=\$ORACLE_HOME/bin:\$PATH
EOF
```

---

## 5. Buat ASM Diskgroup DATA & RECO

```bash
asmca
```
(kalau "command not found" meski PATH sudah benar, coba path lengkap: `/oracle/app/grid/26ai/grid_1/bin/asmca`, lalu `hash -r`)

Klik **Create...** → Disk Group Name `DATA`, redundancy **External**, discovery path `/dev/vd*` kalau disk tidak muncul, centang `/dev/vdd` → OK. Ulangi untuk `RECO` dari `/dev/vde`.

Verifikasi: ketiga diskgroup (OCRVOTE, DATA, RECO) statusnya `MOUNTED(2 of 2)`.

---

## 6. Install Database Software

```bash
su - oracle
# Cek dulu isi kedua file zip edelivery untuk pastikan mana Grid vs Database:
unzip -l <file>.zip | head -20   # cari 'runInstaller' (Database) vs 'gridSetup.sh' (Grid)

unzip -q <db-zip> -d /oracle/app/oracle/product/26ai/db_1
cd /oracle/app/oracle/product/26ai/db_1
export DISPLAY=:1
./runInstaller
```

**Pilihan kunci di wizard:**
1. Select Configuration Option → **"Set Up RAC Software Only"** (bukan "Create and configure a single instance database" — software-only dulu, DBCA jalan terpisah)
2. Select Database Installation Option → "Oracle Real Application Clusters database installation"
3. Select List of Nodes → kedua node sudah auto-tercentang, klik **"SSH connectivity..."** untuk verifikasi
4. Installation Location → software location `/oracle/app/oracle/product/26ai/db_1`
5. Operating System Groups → OSDBA/OSBACKUPDBA/OSDGDBA/OSKMDBA/OSRACDBA semua `dba` (auto-detect)
6. Root script configuration → boleh centang "Automatically run configuration scripts" (root.sh untuk software-only jauh lebih ringan dari Grid, risiko kecil) atau jalankan manual seperti Grid — keduanya valid
7. Prerequisite Checks → **HugePages Existence** dan **Current clock source** cuma Warning, boleh "Ignore All". Kalau **Soft Limit stack** Failed lagi (kemungkinan besar terulang), penyebabnya sama seperti di Grid: sesi terminal `runInstaller` sudah kadung dibuka sebelum fix limits.conf — **tutup terminal, buka baru, ulangi dari `./runInstaller`**

Update `.bash_profile` user `oracle` di kedua node setelah selesai:
```bash
cat >> ~/.bash_profile <<EOF
export ORACLE_HOME=/oracle/app/oracle/product/26ai/db_1
export PATH=\$ORACLE_HOME/bin:\$PATH
EOF
```

---

## 7. Buat Database dengan DBCA

```bash
su - oracle
dbca
```

**Pilihan kunci di wizard, urut:**
1. Select Database Operation → "Create a database"
2. Creation Mode → **"Advanced configuration"** (bukan Typical — supaya bisa pilih diskgroup ASM spesifik)
3. Deployment Type → Database type "Oracle Real Application Cluster (RAC)", Management Policy "Automatic", template "General Purpose or Transaction Processing"
4. Nodes Selection → kedua node auto-tercentang
5. Database Identification → **GDB name TIDAK BOLEH mengandung hyphen** (mis. `dsirac.ds-inovasi.com`, bukan `dsi-oracle26ai-rac...` — domain bagian setelah titik boleh hyphen, bagian nama database yang tidak boleh). SID Prefix maksimal 8 karakter. Container database (CDB dengan 1 PDB) — biarkan tercentang, standar arsitektur modern
6. Storage Option → Automatic Storage Management (ASM), lokasi `+DATA/{DB_UNIQUE_NAME}`, "Use Oracle-Managed Files" tercentang
7. Fast Recovery Option → **wajib centang "Specify Fast Recovery Area"** (default tidak tercentang — kalau dilewatkan, diskgroup RECO yang sudah disiapkan jadi tidak terpakai), arahkan ke `+RECO`. "Enable archiving" opsional, boleh skip untuk lab
8. Data Vault Option → biarkan kedua checkbox kosong (fitur security lanjutan, tidak perlu untuk lab)
9. Configuration Options → Memory: "Automatic Shared Memory Management" (default SGA/PGA cukup, turunkan kalau host ketat RAM); Character sets: AL32UTF8; Connection mode: Dedicated server
10. Management Options → "Run CVU checks periodically" tercentang, EM Cloud Control tidak
11. User Credentials → isi password SYS/SYSTEM/PDBADMIN/DBSNMP (minimal 8 karakter, kombinasi huruf besar-kecil-angka) — **catat di tempat aman**
12. Creation Option → "Create database" tercentang, sisanya default
13. Prerequisite Checks → HugePages & clock source Warning saja, Ignore All aman
14. Summary → review, **Finish**

Setelah selesai: **PDBADMIN terkunci secara default** — buka "Password Management..." di step Finish kalau perlu unlock untuk akses PDB langsung.

---

## 8. Verifikasi Akhir

```bash
srvctl status database -d dsirac
srvctl status scan_listener
/oracle/app/grid/26ai/grid_1/bin/crsctl stat res -t | grep -i dsirac
```
Hasil terverifikasi: `Instance dsirac1 is running on node oracle26ai-01`, `Instance dsirac2 is running on node oracle26ai-02`, ketiga SCAN listener running (2 di node1, 1 di node2).

Test koneksi:
```bash
sqlplus sys/<password>@dsirac.ds-inovasi.com:1521/dsirac as sysdba
SQL> select instance_name, host_name, status from gv$instance;
```
Harus keluar 2 baris, status `OPEN` di kedua instance.

## 9. Autostart (Wajib Cek — Bukan Otomatis di Semua Lapis)

Ada 2 lapis autostart yang terpisah, keduanya harus benar supaya database otomatis jalan lagi setelah restart:

**Lapis 1 — VM autostart di KVM host** (kalau **host fisik** restart, bukan cuma VM). Default installer VM biasanya `disable`, cek dan aktifkan manual:
```bash
# Di KVM host
virsh dominfo ORACLE26AI_RAC_1 | grep Autostart
virsh dominfo ORACLE26AI_RAC_2 | grep Autostart
# kalau "disable":
virsh autostart ORACLE26AI_RAC_1
virsh autostart ORACLE26AI_RAC_2
```

**Lapis 2 — dalam guest, kalau VM sendiri di-restart.** Biasanya sudah otomatis dari instalasi:
```bash
systemctl is-enabled ohasd   # harus 'enabled' — Grid Infrastructure start saat boot
srvctl config database -d dsirac | grep -i "management policy"   # harus 'AUTOMATIC'
```
Dengan Management Policy `AUTOMATIC` (dipilih waktu DBCA), Grid Infrastructure otomatis start database resource begitu cluster stack-nya sendiri up — tidak perlu start manual.

**Test:** reboot salah satu VM lewat OS (`reboot`, bukan `virsh destroy`), tunggu boot selesai, cek `crsctl stat res -t` dan `srvctl status database -d dsirac` — semua harus kembali online tanpa command manual.

---

## 10. Koneksi ke Database via JDBC

**Poin penting yang sering ketuker:** host dalam connection string (JDBC maupun SQL*Plus) itu **SCAN name**, bukan Global Database Name. `dsirac.ds-inovasi.com` adalah nama database (service), `dsi-scan.ds-inovasi.com` adalah nama host untuk connect — dua hal berbeda meski sama-sama berbentuk FQDN.

**JDBC Thin URL — format dasar:**
```
jdbc:oracle:thin:@dsi-scan.ds-inovasi.com:1521/dsirac.ds-inovasi.com
```
Driver: `ojdbc11.jar` (atau `ojdbc8.jar` tergantung versi JDK) — ada di `$ORACLE_HOME/jdbc/lib/` pada DB Home (`/oracle/app/oracle/product/26ai/db_1/jdbc/lib/`), copy ke classpath aplikasi kamu.

**JDBC Thin URL — dengan Connection Description lengkap (lebih robust untuk RAC, ada LOAD_BALANCE & FAILOVER eksplisit):**
```
jdbc:oracle:thin:@(DESCRIPTION=
  (LOAD_BALANCE=on)
  (FAILOVER=on)
  (ADDRESS=(PROTOCOL=TCP)(HOST=dsi-scan.ds-inovasi.com)(PORT=1521))
  (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=dsirac.ds-inovasi.com))
)
```
`LOAD_BALANCE=on` memanfaatkan 3 IP round-robin SCAN yang sudah kita setup di DNS — koneksi baru otomatis tersebar ke instance manapun yang sedang lebih longgar. `FAILOVER=on` bikin JDBC otomatis retry ke instance lain kalau salah satu node down.

**Contoh kode Java minimal:**
```java
import java.sql.*;

String url = "jdbc:oracle:thin:@dsi-scan.ds-inovasi.com:1521/dsirac.ds-inovasi.com";
Connection conn = DriverManager.getConnection(url, "system", "<password>");
```

**Untuk connection pool (HikariCP/aplikasi produksi-style)** — pakai format Connection Description dengan LOAD_BALANCE di atas sebagai JDBC URL pool, supaya pool benar-benar distribusi koneksi ke kedua instance, bukan cuma nembak 1 IP SCAN yang kebetulan resolve duluan.

**Verifikasi service name yang benar-benar terdaftar** (kalau `dsirac.ds-inovasi.com` tidak jalan, cek nama service aktual):
```bash
lsnrctl status
```
Cari baris `Service "..." has 2 instance(s)` — pakai nama persis itu di `SERVICE_NAME`/URL.

**Firewall:** port 1521 (SCAN listener) harus reachable dari mesin client — kalau firewalld kembali aktif di kedua node (kita disable untuk lab), buka port ini dulu.

---

## Lampiran: Pola Kegagalan Umum & Perbaikan

Referensi cepat kalau replikasi ke server lain — kemungkinan besar kamu akan ketemu pola yang sama:

| Gejala | Akar Penyebab | Perbaikan |
|---|---|---|
| `virsh attach-device` sukses secara silent tapi disk tidak muncul di `dumpxml`/guest | SELinux Enforcing di KVM host menolak akses ke path custom (bukan default `/var/lib/libvirt/images/`) | `semanage fcontext -a -t svirt_image_t "<path>.*\.img"` + `restorecon -Rv` sebelum attach |
| Prerequisite Check: Device/ACL check Failed untuk ASM disk | Udev rule pakai `GROUP="asmadmin"`, CVU expect `asmdba` | Ganti ke `asmdba`, reload+trigger udev |
| Prerequisite Check: stack size Failed meski `ulimit -s` manual sudah benar | Soft limit saja tidak cukup (butuh hard juga), atau proses installer sudah inherit ulimit lama dari sesi sebelum perbaikan | Set `hard stack 10240` juga; buka **sesi shell baru** sebelum retry, jangan reuse shell lama |
| Prerequisite Check: avahi-daemon Failed | CVU minta service ini **dimatikan**, arah kebalikan dari intuisi | `systemctl stop/disable/mask avahi-daemon` |
| Node kedua "will be ignored" saat set installation location | Software sudah ter-extract juga di node kedua (harusnya cuma node1) | Kosongkan folder Grid Home di node2 sebelum instalasi |
| `mkdir`/`unzip` Permission Denied di folder software | Ownership **parent** folder (bukan cuma subfolder) balik ke root | `chown` + `chmod 775` di level parent |
| ASM Disk Group step: disk candidate list kosong | Discovery path default `/dev/sd*`, device sebenarnya `/dev/vd*` (virtio) | Ganti discovery path manual di wizard/asmca |
| `[INS-30510] Insufficient number of ASM disks` | Redundancy Normal/High butuh multi-disk mirroring, diskgroup cuma 1 disk | Pakai redundancy **External** |
| `crsctl`/`asmca`/`dbca` "command not found" | `$ORACLE_HOME`/`$PATH` tidak ter-set di sesi shell aktif | Set manual + permanen di `.bash_profile`; kalau masih gagal, panggil pakai path lengkap + `hash -r` |
| SCAN cuma resolve 1 IP, tidak failover | `/etc/hosts` cuma bisa map 1 nama ke 1 IP | Pindah ke DNS 3× A record round-robin; **hapus** entry SCAN dari `/etc/hosts` (nsswitch prioritaskan hosts dulu, DNS jadi percuma kalau hosts masih ada) |
| `dnf install oracle-database-preinstall-*` gagal | Paket khas Oracle Linux, tidak ada di RHEL; URL repo Oracle sering berubah/404 | Setup user/group/kernel-param/limits manual |
| `[DBT-10316] Specified GDB Name contains characters that are not allowed` | Hyphen (`-`) tidak diizinkan di bagian nama database (beda dari aturan hostname/FQDN biasa) | Hilangkan hyphen dari GDB name portion sebelum domain, mis. `dsirac.domain.com` bukan `dsi-oracle-rac.domain.com` |
| Fast Recovery Area tidak aktif meski diskgroup RECO sudah dibuat | Checkbox "Specify Fast Recovery Area" di DBCA **tidak tercentang secara default** | Centang manual di step "Fast Recovery Option" saat create database |
| `ORA-12262: Cannot connect... Could not resolve hostname` saat sqlplus/JDBC | Host di connection string ditulis pakai **Global Database Name** (`dsirac.ds-inovasi.com`), padahal itu bukan hostname yang bisa di-resolve — harus pakai **SCAN name** | `sqlplus user/pass@<scan-name>:1521/<service-name> as sysdba` — SCAN sebagai host, GDB name/service name di bagian akhir setelah port |
| VM tidak otomatis nyala setelah host fisik restart | `virsh dominfo` autostart default `disable` | `virsh autostart <domain>` di KVM host untuk kedua VM |

**Tiga kategori akar masalah yang paling sering muncul, cek ini duluan di server baru:** (1) SELinux enforcing di level host maupun guest, (2) environment variable/PATH yang tidak konsisten antar sesi shell, (3) asumsi default installer (device naming, discovery path, redundancy) yang tidak cocok topologi virtual.
