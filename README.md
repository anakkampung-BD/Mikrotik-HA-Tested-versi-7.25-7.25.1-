# HA MikroTik Versi 7 (Legacy)

Paket High Availability untuk **pair CCR2116-12G-4S+** pada **RouterOS v7**.

Status: **legacy / production-proven** — failover VRRP dan sync config (otomatis + manual) sudah teruji di lapangan.

| Item | Nilai |
|------|--------|
| Versi paket | lihat `VERSION` (setelah generate) |
| Hardware teruji | CCR2116-12G-4S+ (2 unit) |
| RouterOS teruji | **7.25** dan **7.25.1** |
| File install | `HA_init.rsc` (hasil generate) |
| Generator | `HA_generate.py` (Python 3) |

Dasar: [ha-mikrotik](https://github.com/svlsResearch/ha-mikrotik) (cabang v7) + perbaikan identity/scheduler/permission untuk ROS7.

---

## Isi repo GitHub (hanya 3 file)

```
HA_generate.py    ← berisi seluruh paket (ter-embed) + generator .rsc
README.md         ← file ini
OPERATIONS.md     ← perintah operasional harian
```

Clone / download ketiga file, lalu ekspansi:

```bash
python3 HA_generate.py
```

Perintah itu akan menghasilkan struktur lengkap seperti:

```
.
├── HA_generate.py
├── README.md
├── OPERATIONS.md
├── HA_init.rsc              ← import ke router
├── VERSION
├── STEP-BY-STEP.md          ← install dari nol
├── FILES.md
├── .gitignore
├── scripts/                 ← sumber script HA
├── examples/
│   ├── INSTALL-COMMANDS.txt
│   ├── OPERATIONAL-COMMANDS.txt
│   └── fix_identity_node01.rsc
└── heartbeat/
    ├── HA_heartbeat.rsc
    ├── scripts/
    └── examples/
```

---

## Apa yang dilakukan paket ini?

Dua router saling terhubung lewat **satu kabel dedicated** (isi placeholder `CHANGE-ME-HA-IF` dengan `default-name` port Anda):

1. **VRRP** — menentukan siapa ACTIVE dan siapa STANDBY  
2. **FTP sync** — ACTIVE mendorong backup + file ke STANDBY  
3. **(Opsional) Heartbeat** — jika interface produksi di ACTIVE down, standby mengambil alih  

| Placeholder | Arti |
|-------------|------|
| `CHANGE-ME-HA-IF` | Port kabel HA (`$HAInstall interface=...`) |
| `CHANGE-ME-MGMT-IF` | Port management / OOB (`HA_RESCUE`) |
| `CHANGE-ME-MONITOR-IF1` / `IF2` | Interface dipantau heartbeat |
| `CHANGE-ME-BASE-IDENTITY` | Prefiks `/system identity` |

| Peran | Alamat |
|-------|--------|
| Node A (macA) | `169.254.23.1/24` |
| Node B (macB) | `169.254.23.2/24` |
| VRRP VIP | `169.254.23.10` |
| User sync | `ha` (hanya dari jaringan HA) |

Identity sticky:

| MAC peer HA | Node | Format nama |
|-------------|------|-------------|
| macA | node0 | `{BASE}_node0_ACTIVE` / `_STANDBY` |
| macB | node1 | `{BASE}_node1_ACTIVE` / `_STANDBY` |

---

## Mulai cepat

1. `python3 HA_generate.py`  
2. Ganti placeholder identity di `scripts/ha_config.script` (+ fallback `ha_setidentity.script`)  
3. `python3 HA_generate.py --no-extract`  
4. Baca **`STEP-BY-STEP.md`**  
5. Upload **`HA_init.rsc`** → `/import` di calon ACTIVE  
6. `$HAInstall ...` → paste bootstrap ke B → `$HASyncStandby`  

Perintah harian: **`OPERATIONS.md`**.

---

## Perintah generator

```bash
python3 HA_generate.py              # extract paket + generate HA_init.rsc + HA_heartbeat.rsc
python3 HA_generate.py --no-extract # setelah edit scripts/, regenerate .rsc saja
python3 HA_generate.py --check      # verifikasi HA_init.rsc
python3 HA_generate.py --pack       # maintainer: pack ulang payload sebelum push
```

Alur maintainer (setelah ubah scripts/docs di folder hasil extract):

```bash
python3 HA_generate.py --pack
# commit & push hanya: HA_generate.py, README.md, OPERATIONS.md
```

---

## Perintah operasional (ringkas)

```
/system script run ha_functions
$HASyncStandby
$HAPushStandby
$HASwitchRole
```

Heartbeat (setelah import `HA_heartbeat.rsc`):

```
/system script run ha_hb_functions
$HAHeartbeatInstall interfaces="CHANGE-ME-MONITOR-IF1" interval=5s
$HAHeartbeatStatus
```

Detail: **`OPERATIONS.md`** · cheatsheet: `examples/OPERATIONAL-COMMANDS.txt` (setelah generate).

---

## Peringatan

- Siapkan **akses serial/console** saat install dan uji failover.  
- Bootstrap standby akan **reset config** unit B.  
- Ganti semua `CHANGE-ME-*` sebelum production use.  
- Jangan ubah config produksi di **STANDBY** — akan tertimpa sync.

---

## Sumber & validasi

- Upstream: https://github.com/svlsResearch/ha-mikrotik  
- Issue ROS7 terkait: https://github.com/svlsResearch/ha-mikrotik/issues/23  
- Validasi lapangan: CCR2116, ROS **7.25** → **7.25.1**, failover + sync tetap normal.
