# Lanjutan Panduan Sysmon — Tahap 2.8 dst

**Server:** Windows Server 2025 — Domain Controller + DNS Server
**Konfigurasi:** `sysmonconfig-dc.xml`
**Sudah selesai:** 2.1 – 2.7

Jalankan di **PowerShell as Administrator**.

---

## 2.8 Terapkan konfigurasi

```powershell
Set-Location C:\Windows\Sysmon
.\Sysmon64.exe -c C:\Windows\Sysmon\sysmonconfig.xml
```

---

## 2.9 Verifikasi

### 2.9.1 Service dan driver

```powershell
Get-Service Sysmon64, SysmonDrv | Format-Table Name, Status, StartType
```

Keduanya harus `Running`.

### 2.9.2 Konfigurasi aktif

```powershell
Set-Location C:\Windows\Sysmon
.\Sysmon64.exe -c > C:\Windows\Sysmon\active-config-dump.txt
Select-String -Path C:\Windows\Sysmon\active-config-dump.txt -Pattern 'ArchiveDirectory|lsass|RawAccessRead|dns\.exe|dfsrs'
```

Empat hal wajib muncul: `SysmonArchive`, rule `lsass.exe`, blok `RawAccessRead`, dan `dns.exe`.

### 2.9.3 Event log

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 10 |
  Select-Object TimeCreated, Id, @{n='Task';e={$_.TaskDisplayName}} | Format-Table -AutoSize
```

### 2.9.4 Ukuran log — pastikan sudah angka DC

```powershell
wevtutil gl Microsoft-Windows-Sysmon/Operational
```

Jika `maxSize` belum 4294967296:

```powershell
wevtutil sl Microsoft-Windows-Sysmon/Operational /ms:4294967296
```

### 2.9.5 Uji fungsional

```powershell
whoami /all
nslookup github.com

Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1} -MaxEvents 5 |
  ForEach-Object { $_.Properties[10].Value }
```

---

## 2.10 Baseline dan EPS — tunggu 30 menit setelah 2.8

### 2.10.1 Distribusi Event ID

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 20000 |
  Group-Object Id | Sort-Object Count -Descending |
  Select-Object Count, Name | Format-Table -AutoSize
```

### 2.10.2 EPS rata-rata

```powershell
$start = (Get-Date).AddMinutes(-10)
$count = (Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'; StartTime=$start
} -ErrorAction SilentlyContinue).Count
"EPS rata-rata: {0:N1}" -f ($count / 600)
```

Acuan DC: 100–500 EPS wajar. Di atas 800 EPS berarti filter perlu diperketat.

### 2.10.3 Tuning Event ID 10 — WAJIB

Event 10 baru pertama kali aktif di server ini (basis SwiftOnSecurity mematikannya), jadi belum ada baseline.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=10} -MaxEvents 500 -ErrorAction SilentlyContinue |
  ForEach-Object {
    [PSCustomObject]@{
      Source        = $_.Properties[4].Value
      GrantedAccess = $_.Properties[8].Value
    }
  } | Group-Object Source, GrantedAccess | Sort-Object Count -Descending |
  Select-Object Count, Name | Format-Table -AutoSize
```

Untuk setiap kombinasi bervolume tinggi yang terbukti sah, tambahkan `RuleGroup` baru di `sysmonconfig.xml` sebelum `</EventFiltering>` dengan pola berikut:

```xml
<RuleGroup name="LSASS-NamaProsesQueryOnly" groupRelation="and">
  <ProcessAccess onmatch="exclude">
    <SourceImage condition="is">FULL\PATH\KE\PROSES.exe</SourceImage>
    <GrantedAccess condition="is any">0x1000;0x1400;0x101400;0x100000;0x1410</GrantedAccess>
  </ProcessAccess>
</RuleGroup>
```

Jangan meng-exclude berdasarkan `SourceImage` saja — itu membuka celah bagi attacker yang menyuntikkan kode ke proses tersebut.

### 2.10.4 Tuning Event ID 9

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=9} -MaxEvents 500 -ErrorAction SilentlyContinue |
  ForEach-Object { $_.Properties[3].Value } | Group-Object |
  Sort-Object Count -Descending | Select-Object Count, Name | Format-Table -AutoSize
```

Tambahkan full path proses sah ke blok `TODO-BACKUP` di dalam `<RawAccessRead onmatch="exclude">`.

### 2.10.5 Cek TODO yang belum diisi

```powershell
Select-String -Path C:\Windows\Sysmon\sysmonconfig.xml -Pattern 'TODO-SHARE|TODO-BACKUP'

Get-SmbShare | Where-Object { $_.Name -notlike '*$' } | Select-Object Name, Path
Get-Service | Where-Object { $_.DisplayName -match 'Veeam|Acronis|Backup|Commvault|Nakivo|Bacula' } |
  Select-Object Name, DisplayName, Status
```

Jika `Get-SmbShare` hanya menampilkan `SYSVOL` dan `NETLOGON`, blok `TODO-SHARE` tidak perlu diisi.

### 2.10.6 Terapkan hasil tuning

```powershell
Set-Location C:\Windows\Sysmon
.\Sysmon64.exe -c .\sysmonconfig.xml
```

Tidak memutus telemetri, tidak perlu reinstall. Ulangi 2.10.1 – 2.10.4 sampai EPS stabil.

---

## 2.11 Health check terjadwal

```powershell
New-EventLog -LogName Application -Source 'SysmonHealthCheck' -ErrorAction SilentlyContinue

@'
$svc = Get-Service Sysmon64, SysmonDrv -ErrorAction SilentlyContinue
if (($svc | Where-Object Status -ne 'Running') -or ($svc.Count -lt 2)) {
    Write-EventLog -LogName Application -Source 'SysmonHealthCheck' -EventId 9001 `
      -EntryType Error -Message 'Sysmon service atau driver tidak berjalan'
}
'@ | Out-File C:\Windows\Sysmon\healthcheck.ps1 -Encoding UTF8

$action  = New-ScheduledTaskAction -Execute 'powershell.exe' `
             -Argument '-NoProfile -ExecutionPolicy Bypass -File C:\Windows\Sysmon\healthcheck.ps1'
$trigger = New-ScheduledTaskTrigger -Daily -At 07:00
Register-ScheduledTask -TaskName 'Sysmon Health Check' -Action $action -Trigger $trigger `
  -User 'SYSTEM' -RunLevel Highest -Force

Start-ScheduledTask -TaskName 'Sysmon Health Check'
Get-ScheduledTaskInfo -TaskName 'Sysmon Health Check' | Select-Object LastRunTime, LastTaskResult
```

`LastTaskResult` harus `0`.

---

## 2.12 Dokumentasi baseline

```powershell
$doc = 'C:\Windows\Sysmon\baseline-' + (Get-Date -Format 'yyyyMMdd') + '.txt'

"=== HOST ==="                                          | Out-File $doc
"$env:COMPUTERNAME | $(Get-Date -Format u)"              | Out-File $doc -Append
(Get-CimInstance Win32_OperatingSystem).Caption          | Out-File $doc -Append
"=== HASH ==="                                           | Out-File $doc -Append
Get-FileHash C:\Windows\Sysmon\Sysmon64.exe      -Algorithm SHA256 | Out-File $doc -Append
Get-FileHash C:\Windows\Sysmon\sysmonconfig.xml  -Algorithm SHA256 | Out-File $doc -Append
"=== VERSI SYSMON ==="                                   | Out-File $doc -Append
(Get-Item C:\Windows\Sysmon\Sysmon64.exe).VersionInfo.ProductVersion | Out-File $doc -Append
"=== SERVICE ==="                                        | Out-File $doc -Append
Get-Service Sysmon64, SysmonDrv | Format-Table -AutoSize | Out-File $doc -Append
"=== EVENT LOG ==="                                      | Out-File $doc -Append
wevtutil gl Microsoft-Windows-Sysmon/Operational         | Out-File $doc -Append

Get-Content $doc
```

Simpan file ini di luar server sebagai bukti konfigurasi untuk audit.

---

## 2.13 Prosedur upgrade versi Sysmon (untuk nanti)

```powershell
Set-Location C:\Windows\Sysmon
.\Sysmon64.exe -u force
# ganti Sysmon64.exe dengan versi baru
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
Get-Service Sysmon64, SysmonDrv
```

Ada jeda beberapa detik tanpa telemetri di antara kedua perintah. Catat waktunya agar tidak salah dibaca sebagai tampering saat investigasi.

Perubahan konfigurasi saja cukup `-c`, tanpa memutus telemetri.

---

# 3. Konfigurasi Wazuh Agent

## 3.1 Backup dan edit ossec.conf

```powershell
$conf = 'C:\Program Files (x86)\ossec-agent\ossec.conf'
Copy-Item $conf "$conf.bak-$(Get-Date -Format yyyyMMdd)" -Force
notepad $conf
```

Tambahkan sebelum `</ossec_config>`:

```xml
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>

  <client_buffer>
    <disabled>no</disabled>
    <queue_size>100000</queue_size>
    <events_per_second>1000</events_per_second>
  </client_buffer>
```

## 3.2 Filter sisi agent — hanya jika EPS di 2.10.2 di atas 500

```xml
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
    <query>Event/System[EventID != 5 and EventID != 22]</query>
  </localfile>
```

Gunakan blok ini sebagai pengganti blok di 3.1, bukan tambahan.

## 3.3 Restart dan verifikasi agent

```powershell
Restart-Service WazuhSvc
Start-Sleep -Seconds 10
Get-Service WazuhSvc
Get-Content 'C:\Program Files (x86)\ossec-agent\ossec.log' -Tail 40
```

Cari baris yang menyebut `Microsoft-Windows-Sysmon/Operational`, pastikan tidak ada `ERROR`.

## 3.4 Verifikasi di Wazuh manager

```bash
/var/ossec/bin/agent_control -l
tail -f /var/ossec/logs/alerts/alerts.json | grep -i sysmon
```

## 3.5 Ambil nomor if_sid

```bash
/var/ossec/bin/wazuh-logtest
```

Tempel satu sample event Sysmon dari server, catat nomor `id` rule yang cocok. Nomor ini dipakai di 3.6.

## 3.6 Custom rule DC

Edit `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="sysmon,windows,dc,">

  <rule id="100500" level="14">
    <if_sid>61609</if_sid>
    <field name="win.eventdata.targetImage">\\\\lsass.exe$</field>
    <description>Sysmon: akses ke LSASS di DC - indikasi credential dumping</description>
    <mitre><id>T1003.001</id></mitre>
  </rule>

  <rule id="100501" level="15">
    <if_sid>61603</if_sid>
    <field name="win.eventdata.image">ntdsutil.exe|esentutl.exe|vssadmin.exe|diskshadow.exe</field>
    <description>Sysmon: eksekusi tool ekstraksi NTDS.dit</description>
    <mitre><id>T1003.003</id></mitre>
  </rule>

  <rule id="100502" level="13">
    <if_sid>61608</if_sid>
    <description>Sysmon: RawAccessRead di DC - dugaan pembacaan raw disk</description>
    <mitre><id>T1006</id></mitre>
  </rule>

  <rule id="100503" level="12">
    <if_sid>61600</if_sid>
    <field name="win.eventdata.targetFilename">SYSVOL</field>
    <description>Sysmon: penulisan file ke SYSVOL - dugaan persistensi via GPO script</description>
    <mitre><id>T1484.001</id></mitre>
  </rule>

  <rule id="100504" level="15">
    <if_sid>61644</if_sid>
    <description>Sysmon: perubahan status service atau konfigurasi - dugaan tampering</description>
    <mitre><id>T1562.001</id></mitre>
  </rule>

</group>
```

Ganti semua `if_sid` dengan hasil 3.5.

## 3.7 Restart manager dan verifikasi

```bash
systemctl restart wazuh-manager
systemctl status wazuh-manager
grep -c 'Microsoft-Windows-Sysmon' /var/ossec/logs/alerts/alerts.json
tail -f /var/ossec/logs/alerts/alerts.log
```

---

# 4. Referensi perintah

| Tujuan | Perintah |
|---|---|
| Validasi konfigurasi | `Sysmon64.exe -c sysmonconfig.xml` |
| Update konfigurasi | `Sysmon64.exe -c sysmonconfig.xml` |
| Dump konfigurasi aktif | `Sysmon64.exe -c` |
| Reset ke default | `Sysmon64.exe -c --` |
| Cetak schema | `Sysmon64.exe -s` |
| Versi schema | `Sysmon64.exe -? config` |
| Uninstall | `Sysmon64.exe -u force` |

---

# 5. Checklist sisa

```
[ ] 2.8  Konfigurasi diterapkan
[ ] 2.9  Sysmon64 + SysmonDrv Running
[ ] 2.9  Dump -c memuat lsass, RawAccessRead, dns.exe, SysmonArchive
[ ] 2.9  Ukuran event log 4 GB
[ ] 2.10 Baseline EPS terukur setelah 30 menit
[ ] 2.10 Event ID 10 di-tuning dengan pola groupRelation="and"
[ ] 2.10 Event ID 9 di-tuning
[ ] 2.10 TODO-SHARE dan TODO-BACKUP diperiksa
[ ] 2.11 Health check terjadwal, LastTaskResult = 0
[ ] 2.12 Baseline didokumentasikan dan disimpan di luar server
[ ] 3.1  ossec.conf di-backup sebelum diedit
[ ] 3.3  Agent restart tanpa ERROR di ossec.log
[ ] 3.4  Alert Sysmon masuk di manager
[ ] 3.5  if_sid diverifikasi via wazuh-logtest
[ ] 3.6  Custom rule aktif
[ ] Alert tampering Event ID 4 dan 16 aktif dengan level tinggi
```
