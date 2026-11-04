# Operasional HA (setelah instalasi)

Semua perintah `$HA...` butuh fungsi ter-load:

```
/system script run ha_functions
```

Untuk heartbeat (addon):

```
/system script run ha_hb_functions
```

(Biasanya sudah otomatis setelah boot lewat `ha_startup` / `ha_hb_boot`.)

File siap-tempel lengkap: **`examples/OPERATIONAL-COMMANDS.txt`**.

---

## Cheat sheet perintah

### Status & diagnosa

| Perintah | Fungsi |
|----------|--------|
| `/system identity print` | Nama node + role (ACTIVE/STANDBY) |
| `/interface vrrp print detail` | Siapa master/backup + priority |
| `:put $isMaster` | `true` = ACTIVE |
| `:put $isStandbyInSync` | `true` = sync terakhir GOOD |
| `:put $haCheckMasterVer` / `$haCheckStandbyVer` | Fingerprint config |
| `:put $haStartupHAVersion` | Versi paket HA yang running |
| `/ping 169.254.23.2 count=3` | Peer HA reachable? |
| `/log print where message~"ha_"` | Log HA |

### Sync & failover (inti)

| Perintah | Fungsi | Di mana |
|----------|--------|---------|
| `$HASyncStandby` | Cek versi; push otomatis jika beda | ACTIVE |
| `$HAPushStandby` | Paksa push backup + reboot standby | ACTIVE |
| `$HASwitchRole` | Sync lalu reboot active (graceful failover) | ACTIVE |
| `$HALoopPushStandby` | Stress test push berulang (**lab only**) | ACTIVE |

### Identity & rescue

| Perintah | Fungsi |
|----------|--------|
| `/system script run ha_setidentity` | Paksa set nama node0/node1 + ACTIVE/STANDBY |
| `/interface ethernet set [find default-name=CHANGE-ME-MGMT-IF] comment=HA_RESCUE` | Port OOB tidak di-disable saat standby |

### Heartbeat (addon)

| Perintah | Fungsi |
|----------|--------|
| `$HAHeartbeatInstall interfaces="CHANGE-ME-MONITOR-IF1,CHANGE-ME-MONITOR-IF2"` | Pasang monitor + scheduler |
| `$HAHeartbeatSet ...` | Ubah setting / `enabled=no` |
| `$HAHeartbeatStatus` | Lihat status streak / latched / priority |
| `$HAHeartbeatCheckNow` | Jalankan cek sekali sekarang |
| `$HAHeartbeatForceFailover` | Paksa demote VRRP (uji) |
| `$HAHeartbeatEnsureIfs` | Pastikan IF heartbeat enabled |

### Install ulang / rebuild

| Perintah | Fungsi |
|----------|--------|
| `$HAInstall interface=... macA=... macB=... password=...` | Install / rebuild peer (hati-hati) |

---

## Cek status cepat

```
/system identity print
/interface vrrp print detail
:put $isMaster
:put $isStandbyInSync
:put $haCheckMasterVer
:put $haCheckStandbyVer
/log print where message~"ha_"
```

| Kondisi sehat | Active | Standby |
|---------------|--------|---------|
| Identity | `..._node0_ACTIVE` atau `..._node1_ACTIVE` | `..._nodeX_STANDBY` (node beda) |
| VRRP | master | backup |
| `$isStandbyInSync` di active | `true` setelah `$HASyncStandby` GOOD | — |

---

## Sync config ACTIVE → STANDBY

```
$HASyncStandby
```

- Bandingkan fingerprint config.  
- Jika beda → push otomatis (standby reboot).  
- Jika sama → cetak `GOOD`.

Paksa push:

```
$HAPushStandby
```

---

## Switch role (graceful)

Di **ACTIVE**:

```
$HASwitchRole
```

Urutan: push backup → tunggu standby up → delay → reboot active. Standby menjadi active.

---

## Generate / ekspansi paket (laptop)

Repo GitHub hanya 3 file. Ekspansi dulu:

```bash
python3 HA_generate.py
```

Setelah edit `scripts/*.script` lokal:

```bash
python3 HA_generate.py --no-extract
```

Maintainer (sebelum push ke GitHub):

```bash
python3 HA_generate.py --pack
# push: HA_generate.py + README.md + OPERATIONS.md
```

---

## Upgrade paket HA (script baru)

1. Di laptop: `python3 HA_generate.py` (atau `--no-extract` jika sudah di-extract).  
2. Upload `HA_init.rsc` baru ke **ACTIVE**.  
3. `/import HA_init.rsc`  
4. `$HAPushStandby` — tunggu standby kembali.  
5. Cek log standby.  
6. `$HASyncStandby`  
7. (Opsional) `$HASwitchRole` agar unit yang tadi active juga menjalankan kode baru setelah jadi standby, lalu switch balik jika perlu.

---

## Upgrade RouterOS (contoh 7.25 → 7.25.1)

1. Pastikan sync `GOOD`.  
2. Upgrade **STANDBY** dulu → reboot → pastikan HA startup OK.  
3. `$HASwitchRole` (atau failover) agar standby jadi active.  
4. Upgrade unit yang sekarang standby.  
5. Verifikasi sync + uji failover lagi.

Paket ini sudah berjalan normal di **7.25.1** tanpa perubahan script.

---

## Rebuild standby (ganti hardware)

RouterOS standby baru harus versi kompatibel. Reset standby, sambungkan port HA (`CHANGE-ME-HA-IF`) sama seperti sebelumnya.

**Jika A active:**

```
$HAInstall interface=$haInterface macA=$haMacMe macB="AA:BB:CC:00:00:99" password=$haPassword
```

Ikuti paste bootstrap ke unit baru, lalu `$HASyncStandby`.

**Jika B active:** tukar peran `macA`/`macB` sesuai `$haMacMe`.

---

## Identity salah / sama di kedua node

Paket ini sudah memasang `ha_setidentity` sticky + scheduler `ha_fix_identity`.

Perbaikan manual:

```
/system script run ha_setidentity
/system identity print
```

Jika masih pakai script lama tanpa fix:

```
/import examples/fix_identity_node01.rsc
```

lalu `$HAPushStandby`.

Ubah prefiks nama: edit `haBaseIdentity` di `scripts/ha_config.script` (dan fallback `baseName` di `scripts/ha_setidentity.script`), ganti `CHANGE-ME-BASE-IDENTITY` dengan nama Anda, jalankan `python3 HA_generate.py --no-extract`, import ulang `HA_init.rsc`.

---

## Port management tetap hidup di standby

```
/interface ethernet set [find default-name=CHANGE-ME-MGMT-IF] comment=HA_RESCUE
```

Interface dengan `comment=HA_RESCUE` tidak di-disable oleh `ha_onbackup` / fail-safe startup.

---

## Heartbeat monitor interface (addon)

Folder `heartbeat/`. Setelah HA stabil:

```
/import HA_heartbeat.rsc
/system script run ha_hb_functions
$HAHeartbeatInstall interfaces="CHANGE-ME-MONITOR-IF1,CHANGE-ME-MONITOR-IF2" interval=5s
$HASyncStandby
$HAHeartbeatStatus
```

Operasional harian heartbeat:

```
$HAHeartbeatStatus
$HAHeartbeatCheckNow
$HAHeartbeatSet enabled=no
$HAHeartbeatSet enabled=yes
$HAHeartbeatForceFailover
```

Detail: `heartbeat/STEP-BY-STEP.md` dan `examples/OPERATIONAL-COMMANDS.txt`.

---

## Jangan lakukan

- Jangan edit config produksi di **STANDBY** — akan tertimpa sync.  
- Jangan isi IP sendiri di port HA / `bridge-<HA-IF>` / `HA_VRRP` tanpa comment `HA_AUTO`.  
- Jangan matikan FTP dari jaringan `169.254.23.0/24` — sync butuh FTP user `ha`.  
- Jangan jalankan `$HAInstall` ulang tanpa perlu.  
- Jangan pakai `$HALoopPushStandby` di produksi.
