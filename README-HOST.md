# RustDesk HOST — build terkunci (SNR)

Fork: https://github.com/sinar-bonar/rustdesk (dari `rustdesk/rustdesk` v1.5.0)
Branch kerja: `host-lock`

## Apa yang dikunci

Seluruh kebijakan dipasang di `src/common.rs` (`apply_host_policy()`), dipanggil paling awal di
`load_custom_client()` — sehingga ikut jalan di **semua** proses (service, tray, UI) dan tidak bisa
dibatalkan dari UI maupun command line.

| Setelan | Nilai | Efek |
|---|---|---|
| `custom-rendezvous-server` | `109.123.235.183` | hanya server VPS SNR |
| `relay-server` | `109.123.235.183` | relay hanya lewat VPS |
| `key` | `EZrdsMw29…` | hanya client dengan key sama bisa masuk |
| `direct-server` | `N` | tidak ada listener direct-IP |
| `enable-lan-discovery` | `N` | tidak ditemukan lewat LAN |
| `disable-udp` | `Y` | tanpa jalur UDP/punch |
| `allow-websocket` | `N` | tetap di port 21115–21117 |
| `allow-remote-config-modification` | `N` | peer tidak bisa mengubah setelan kita |
| `enable-check-update` / `allow-auto-update` | `N` | tidak bisa menukar diri dengan build stok |
| `conn-type` (HARD) | `incoming` | incoming-only: bisa dikontrol, tidak bisa mengontrol |
| `disable-settings` (HARD) | `Y` | UI Settings mati + perubahan lewat CLI diblokir |
| `password` + `salt` (HARD) | preset hash | password permanen di-bake, tidak bisa diubah |
| `disable-change-permanent-password` (BUILTIN) | `Y` | ganti password ditolak |
| `hide-*-settings` (BUILTIN) | `Y` | halaman pengaturan disembunyikan |

Relay-only juga ditegakkan di sisi host (`src/rendezvous_mediator.rs`): `relay = true` untuk setiap
permintaan punch, dan jalur WebRTC dimatikan. Jadi walau ada client lain yang memaksa P2P, host tetap
menolak jalur langsung.

**Password permanen** yang di-bake: lihat file pekerjaan sesi ini (tidak ditulis di repo).
Storage preset berbentuk `00` + base64(sha256(password+salt)), sama dengan format preset hbbs.

## Build

Workflow khusus Windows x86_64: `.github/workflows/build-host.yml` (`workflow_dispatch`, tidak
menyentuh platform lain, tanpa langkah signing/MSI/release).

```
gh workflow run build-host.yml -R sinar-bonar/rustdesk --ref host-lock
gh run watch -R sinar-bonar/rustdesk
gh run download -R sinar-bonar/rustdesk -n rustdesk-host-installer-x86_64
```

Dua artefak:
- `rustdesk-host-installer-x86_64` → `rustdesk-1.5.0-x86_64.exe` (installer self-extract, ini yang dikirim ke PC klien)
- `rustdesk-unsigned-windows-x86_64` → folder aplikasi mentah (debug)

## Pasang di PC klien

1. Jalankan `rustdesk-1.5.0-x86_64.exe` → install (service jalan sebagai LocalSystem, auto-start).
2. Tidak ada konfigurasi yang perlu diisi; server + key + password sudah menyatu di dalam binary.
3. Dari sisi admin: connect ke ID mesin itu, masukkan password yang di-bake.

## Mengganti server / password / setelan

Ubah konstanta `HOST_POLICY_*` di `src/common.rs`, commit, jalankan workflow lagi. Storage password
dihitung: `00` + base64(sha256(password + salt)).
