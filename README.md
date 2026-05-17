# Ansible Provisioning Node untuk Ubuntu/OpenStack

Project ini menyediakan playbook dan role Ansible untuk provisioning node Ubuntu pada environment OpenStack. Scope saat ini mencakup bootstrap sistem dasar dan provisioning network berbasis `netplan`, sehingga penambahan node baru tidak perlu lagi banyak konfigurasi manual.

## Fitur

- Setup hostname per node dari variabel host.
- Setup timezone, NTP, dan `local-rtc` menggunakan `timedatectl`.
- Sinkronisasi `/etc/hosts` untuk semua node `controller` dan `compute` berdasarkan segmen network.
- Mencegah cloud-init menimpa `/etc/hosts` saat reboot dengan menonaktifkan `update_etc_hosts` dan `manage_etc_hosts`.
- Disable konfigurasi network dari cloud-init.
- Discovery MAC address berdasarkan nama interface yang saat ini ada di host.
- Rename interface secara opsional menggunakan `match.macaddress` dan `set-name`.
- Render file netplan dinamis untuk ethernet, bond, dan VLAN.
- Validasi input sebelum render dan validasi `netplan generate` sebelum `netplan apply`.
- Skalabel untuk penambahan node baru melalui inventory dan `host_vars`.

## Struktur Folder

```text
.
|-- ansible.cfg
|-- inventory/
|   |-- hosts.yml
|   |-- group_vars/
|   |   |-- all/
|   |   |   |-- network.yml
|   |   |   `-- system.yml
|   |   |-- controller/network.yml
|   |   `-- compute/network.yml
|   `-- host_vars/
|       |-- controller01.yml
|       `-- compute01.yml
|-- playbooks/
|   `-- provision-network.yml
`-- roles/
    |-- system_bootstrap/
    |   |-- defaults/main.yml
    |   |-- tasks/
    |   `-- templates/hosts.managed.j2
    `-- network_provision/
        |-- defaults/main.yml
        |-- tasks/
        `-- templates/netplan.yaml.j2
```

## Alur Role

1. Validasi variabel system bootstrap.
2. Sesuaikan `cloud-init` agar tidak mengelola `/etc/hosts` lagi.
3. Set hostname node.
4. Set timezone, NTP, dan `local-rtc` dengan `timedatectl`.
5. Render block managed `/etc/hosts` untuk alias hostname lintas node.
6. Validasi schema variabel network.
7. Disable network config dari cloud-init.
8. Hapus file legacy `/etc/netplan/50-cloud-init.yaml` bila diaktifkan.
9. Discover interface aktual dengan `ip -j link show`.
10. Ambil MAC address berdasarkan `current_name`.
11. Render file `/etc/netplan/60-openstack-network.yaml`.
12. Jalankan `netplan generate` lalu `netplan apply` jika ada perubahan.

## Variabel System Bootstrap

### `system_hostname`

Hostname target untuk node. Variabel ini wajib diisi per host.

### `system_timezone`

Timezone target untuk node, misalnya `Asia/Jakarta`.

### `system_ntp_enabled`

Flag boolean untuk mengaktifkan atau menonaktifkan sinkronisasi NTP melalui `timedatectl`.

### `system_local_rtc`

Flag boolean untuk mengatur apakah RTC hardware menggunakan local time.

### `system_manage_etc_hosts`

Flag boolean untuk mengaktifkan sinkronisasi block managed `/etc/hosts`.

### `system_hosts_target_groups`

Daftar grup inventory yang ikut dimasukkan ke `/etc/hosts`. Default saat ini adalah `controller` dan `compute`.

### `system_hosts_primary_segment_name`

Nama segmen yang diperlakukan sebagai management utama. Segmen ini akan mendapat dua alias: short hostname dan FQDN segmen, misalnya `compute01` dan `compute01.mgmt`.

### `system_manage_cloud_init_hosts`

Flag boolean untuk mengaktifkan perbaikan `cloud-init` pada `/etc/cloud/cloud.cfg` agar `/etc/hosts` tidak ditimpa saat reboot.

### `system_cloud_cfg_file`

Path file `cloud-init` utama yang akan dikelola. Default saat ini adalah `/etc/cloud/cloud.cfg`.

Contoh:

```yaml
system_hostname: controller01
system_timezone: Asia/Jakarta
system_ntp_enabled: true
system_local_rtc: false
system_manage_etc_hosts: true
system_hosts_target_groups:
  - controller
  - compute
system_hosts_primary_segment_name: mgmt
system_manage_cloud_init_hosts: true
system_cloud_cfg_file: /etc/cloud/cloud.cfg
```

## Variabel Utama

### `network_ethernets`

Daftar interface fisik yang dikelola. Field penting:

- `current_name`: nama interface yang saat ini ada di OS.
- `name`: nama target baru. Jika dikosongkan atau tidak diisi, interface tidak di-rename.
- `segment_name`: nama segmen network untuk generate alias `/etc/hosts`, misalnya `mgmt`, `api_internal`, `storage`, atau `tenant`.
- `dhcp4`, `dhcp6`, `addresses`, `gateway4`, `gateway6`, `nameservers`, `routes`, `mtu`, `optional`: opsi netplan per interface.

Contoh:

```yaml
network_ethernets:
  - current_name: ens18
    name: eth0
    segment_name: mgmt
    dhcp4: false
    addresses:
      - 10.10.10.11/24
    gateway4: 10.10.10.1
    nameservers:
      addresses:
        - 10.10.10.2
        - 8.8.8.8
  - current_name: ens19
    name: eth1
  - current_name: ens20
    name: eth2
```

### `network_bonds`

Daftar bond yang akan dibentuk dari nama interface final.

```yaml
network_bonds:
  - name: bond0
    interfaces:
      - eth1
      - eth2
    parameters:
      mode: 802.3ad
      mii-monitor-interval: 100
      transmit-hash-policy: layer3+4
```

### `network_vlans`

Daftar VLAN yang dibentuk di atas ethernet atau bond.

```yaml
network_vlans:
  - name: bond0.101
    id: 101
    link: bond0
    segment_name: api_internal
    addresses:
      - 172.16.101.11/24
```

## Pola Alias /etc/hosts

- Jika `segment_name` sama dengan `system_hosts_primary_segment_name`, hasil alias:
  - `IP system_hostname system_hostname.segment_name`
- Jika `segment_name` bukan primary management, hasil alias:
  - `IP system_hostname.segment_name`

Contoh hasil:

```text
10.10.10.21 compute01 compute01.mgmt
172.16.101.21 compute01.api_internal
172.16.201.21 compute01.tenant
```

## Integrasi Cloud-Init

- Role `system_bootstrap` juga memperbaiki `/etc/cloud/cloud.cfg` agar `cloud-init` tidak menghapus hasil sinkronisasi `/etc/hosts` saat node reboot.
- Module `update_etc_hosts` akan dihapus dari `cloud.cfg`.
- Key `manage_etc_hosts: false` akan dipastikan ada agar pengelolaan `/etc/hosts` tetap berada di Ansible.

## Cara Menambah Node Baru

1. Tambahkan host baru ke grup `controller` atau `compute` di `inventory/hosts.yml`.
2. Buat file `inventory/host_vars/<hostname>.yml`.
3. Isi variabel system bootstrap dan network sesuai kebutuhan host, termasuk `segment_name` pada interface/bond/VLAN yang ingin ikut dirender ke `/etc/hosts`.
4. Jalankan playbook provisioning network.

Contoh menambah host baru:

```yaml
compute:
  hosts:
    compute02:
      ansible_host: 192.168.122.22
      ansible_user: ubuntu
```

Lalu buat `inventory/host_vars/compute02.yml` dengan schema yang sama seperti contoh host lain, misalnya:

```yaml
system_hostname: compute02
system_timezone: Asia/Jakarta
system_ntp_enabled: true
system_local_rtc: false
system_manage_etc_hosts: true
system_hosts_primary_segment_name: mgmt

network_ethernets:
  - current_name: ens18
    segment_name: mgmt
    dhcp4: false
    addresses:
      - 10.10.10.22/24
    gateway4: 10.10.10.1
  - current_name: ens19
    name: eth1
  - current_name: ens20
    name: eth2

network_bonds:
  - name: bond0
    interfaces:
      - eth1
      - eth2

network_vlans:
  - name: bond0.101
    id: 101
    link: bond0
    segment_name: api_internal
    addresses:
      - 172.16.101.22/24
```

## Menjalankan Playbook

```bash
ansible-playbook playbooks/provision-network.yml
```

## Catatan Operasional

- Playbook berjalan `serial: 1` agar perubahan network diterapkan per-host, bukan sekaligus.
- Role `system_bootstrap` dijalankan lebih dulu sebelum `network_provision`.
- `/etc/hosts` dikelola dengan block managed, jadi entri lokal di luar block Ansible tetap dipertahankan.
- `cloud-init` juga disesuaikan agar tidak mengambil alih `/etc/hosts` lagi setelah reboot.
- IP yang dipakai untuk alias `/etc/hosts` adalah IP pertama pada `addresses`.
- Default saat ini adalah apply langsung setelah `netplan generate` sukses.
- Jika management interface sudah benar namanya, cukup isi `current_name` tanpa `name`.
- Bond harus selalu mereferensikan nama interface final, bukan nama interface sebelum rename.
- VLAN harus mereferensikan `link` ke ethernet final atau ke bond yang sudah didefinisikan.
