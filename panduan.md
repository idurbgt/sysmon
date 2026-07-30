# Panduan Instalasi Sysmon + Template SwiftOnSecurity

**Versi Sysmon:** v15.21 | **Template:** SwiftOnSecurity/sysmon-config | **Tanggal:** 30 Juli 2026

---

# 1. Instalasi dan Konfigurasi Sysmon di Windows 10

Jalankan seluruh perintah pada **PowerShell as Administrator**.

## 1.1 Unduh berkas

```powershell
New-Item -ItemType Directory -Path C:\Sysmon -Force | Out-Null
Set-Location C:\Sysmon

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Sysmon\Sysmon.zip"
Expand-Archive -Path "C:\Sysmon\Sysmon.zip" -DestinationPath "C:\Sysmon" -Force

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Sysmon\sysmonconfig.xml"
```

## 1.2 Verifikasi berkas

```powershell
Get-AuthenticodeSignature C:\Sysmon\Sysmon64.exe | Format-List Status, SignerCertificate
Get-FileHash C:\Sysmon\Sysmon64.exe    -Algorithm SHA256
Get-FileHash C:\Sysmon\sysmonconfig.xml -Algorithm SHA256
```

Status harus `Valid`.

## 1.3 Validasi konfigurasi

```powershell
.\Sysmon64.exe -accepteula -c .\sysmonconfig.xml
```

## 1.4 Install

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

## 1.5 Naikkan ukuran event log

```powershell
wevtutil sl Microsoft-Windows-Sysmon/Operational /ms:1073741824
wevtutil gl Microsoft-Windows-Sysmon/Operational
```

## 1.6 Verifikasi

```powershell
Get-Service Sysmon64, SysmonDrv | Format-Table Name, Status, StartType

.\Sysmon64.exe -c | Select-Object -First 40

Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
  Select-Object TimeCreated, Id, @{n='Task';e={$_.TaskDisplayName}} | Format-Table -AutoSize
```

## 1.7 Uji fungsional

```powershell
whoami /all
nslookup github.com

Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1} -MaxEvents 5 |
  ForEach-Object { $_.Properties[10].Value }
```

## 1.8 Baseline noise

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5000 |
  Group-Object Id | Sort-Object Count -Descending |
  Select-Object Count, Name | Format-Table -AutoSize
```

## 1.9 Tuning: nonaktifkan DNS logging

Hapus atau komentari seluruh blok `<DnsQuery>` di `sysmonconfig.xml`.

## 1.10 Tuning: exclude antivirus

Tambahkan sebelum penutup `</EventFiltering>`:

```xml
<RuleGroup name="AV Exclusions" groupRelation="or">
  <FileCreate onmatch="exclude">
    <Image condition="begin with">C:\Program Files\Windows Defender\</Image>
    <Image condition="begin with">C:\ProgramData\Microsoft\Windows Defender\</Image>
  </FileCreate>
  <ProcessCreate onmatch="exclude">
    <Image condition="is">C:\Program Files\Windows Defender\MpCmdRun.exe</Image>
  </ProcessCreate>
</RuleGroup>
```

## 1.11 Terapkan perubahan konfigurasi

```powershell
.\Sysmon64.exe -c C:\Sysmon\sysmonconfig.xml
```

## 1.12 Referensi perintah

| Tujuan | Perintah |
|---|---|
| Install | `Sysmon64.exe -accepteula -i sysmonconfig.xml` |
| Update konfigurasi | `Sysmon64.exe -c sysmonconfig.xml` |
| Dump konfigurasi aktif | `Sysmon64.exe -c` |
| Reset ke default | `Sysmon64.exe -c --` |
| Cetak schema | `Sysmon64.exe -s` |
| Versi schema | `Sysmon64.exe -? config` |
| Install manifest | `Sysmon64.exe -m` |
| Uninstall | `Sysmon64.exe -u` |
| Uninstall paksa | `Sysmon64.exe -u force` |
| Upgrade versi | `Sysmon64.exe -u force` lalu `Sysmon64.exe -accepteula -i sysmonconfig.xml` |

---

# 2. Instalasi dan Konfigurasi Sysmon di Windows Server 2025

## 2.1 Siapkan direktori dengan ACL ketat

```powershell
$dir = "C:\Windows\Sysmon"
New-Item -ItemType Directory -Path $dir -Force | Out-Null

icacls $dir /inheritance:r
icacls $dir /grant:r "SYSTEM:(OI)(CI)F"
icacls $dir /grant:r "BUILTIN\Administrators:(OI)(CI)F"
icacls $dir /grant:r "BUILTIN\Users:(OI)(CI)RX"
```

## 2.2 Unduh berkas

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:TEMP\Sysmon.zip"
Expand-Archive -Path "$env:TEMP\Sysmon.zip" -DestinationPath "$env:TEMP\Sysmon" -Force
Copy-Item "$env:TEMP\Sysmon\Sysmon64.exe" -Destination "C:\Windows\Sysmon\" -Force

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Windows\Sysmon\sysmonconfig.xml"
```

## 2.3 Cek prasyarat khusus Server 2025

```powershell
# SMB signing (wajib default di Server 2025 — cek sebelum distribusi via file share)
Get-SmbClientConfiguration | Select-Object RequireSecuritySignature, EnableSecuritySignature

# Credential Guard / VBS
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Select-Object SecurityServicesConfigured, SecurityServicesRunning

# Identifikasi role terpasang (menentukan filter di 2.7)
Get-WindowsFeature | Where-Object Installed | Select-Object Name, DisplayName
```

## 2.4 Pindahkan ArchiveDirectory keluar volume data

Edit header `sysmonconfig.xml`:

```xml
<Sysmon schemaversion="4.82">
  <ArchiveDirectory>SysmonArchive</ArchiveDirectory>
  <HashAlgorithms>SHA256,IMPHASH</HashAlgorithms>
  <CheckRevocation>False</CheckRevocation>
```

## 2.5 Validasi dan install

```powershell
Set-Location C:\Windows\Sysmon
.\Sysmon64.exe -accepteula -c .\sysmonconfig.xml
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

## 2.6 Naikkan ukuran event log

```powershell
# Server aplikasi umum
wevtutil sl Microsoft-Windows-Sysmon/Operational /ms:2147483648

# Domain Controller / IIS trafik tinggi
wevtutil sl Microsoft-Windows-Sysmon/Operational /ms:4294967296
```

## 2.7 Terapkan filter sesuai role

Tambahkan blok berikut sebelum penutup `</EventFiltering>`, sesuai role server.

### Domain Controller

```xml
<RuleGroup name="DC noise" groupRelation="or">
  <DnsQuery onmatch="exclude">
    <Image condition="is">C:\Windows\System32\dns.exe</Image>
  </DnsQuery>
  <FileCreate onmatch="exclude">
    <Image condition="is">C:\Windows\System32\dfsrs.exe</Image>
  </FileCreate>
  <FileCreateTime onmatch="exclude">
    <Image condition="is">C:\Windows\System32\dfsrs.exe</Image>
  </FileCreateTime>
</RuleGroup>
<RuleGroup name="DC high value" groupRelation="or">
  <ProcessCreate onmatch="include">
    <Image condition="end with">ntdsutil.exe</Image>
    <Image condition="end with">esentutl.exe</Image>
    <Image condition="end with">vssadmin.exe</Image>
    <Image condition="end with">wbadmin.exe</Image>
  </ProcessCreate>
</RuleGroup>
```

Jangan meng-exclude ProcessAccess ke `lsass.exe`.

### IIS / Web Server

```xml
<RuleGroup name="IIS high value" groupRelation="or">
  <ProcessCreate onmatch="include">
    <ParentImage condition="end with">w3wp.exe</ParentImage>
    <ParentImage condition="end with">php-cgi.exe</ParentImage>
  </ProcessCreate>
  <FileCreate onmatch="include">
    <TargetFilename condition="begin with">C:\inetpub\wwwroot</TargetFilename>
  </FileCreate>
</RuleGroup>
<RuleGroup name="IIS noise" groupRelation="or">
  <FileCreate onmatch="exclude">
    <TargetFilename condition="contains">\Temporary ASP.NET Files\</TargetFilename>
    <TargetFilename condition="contains">\Config\MACHINE\</TargetFilename>
  </FileCreate>
</RuleGroup>
```

### File Server

```xml
<RuleGroup name="File server noise" groupRelation="or">
  <FileCreate onmatch="exclude">
    <TargetFilename condition="begin with">D:\Shares\</TargetFilename>
    <TargetFilename condition="begin with">E:\Data\</TargetFilename>
  </FileCreate>
  <FileCreateStreamHash onmatch="exclude">
    <TargetFilename condition="begin with">D:\Shares\</TargetFilename>
  </FileCreateStreamHash>
</RuleGroup>
```

### SQL Server

```xml
<RuleGroup name="SQL" groupRelation="or">
  <ProcessCreate onmatch="include">
    <ParentImage condition="end with">sqlservr.exe</ParentImage>
  </ProcessCreate>
  <FileCreate onmatch="exclude">
    <Image condition="end with">sqlservr.exe</Image>
  </FileCreate>
</RuleGroup>
```

### Hyper-V Host

```xml
<RuleGroup name="Hyper-V noise" groupRelation="or">
  <FileCreate onmatch="exclude">
    <Image condition="end with">vmwp.exe</Image>
    <Image condition="end with">vmms.exe</Image>
    <TargetFilename condition="end with">.vhdx</TargetFilename>
    <TargetFilename condition="end with">.avhdx</TargetFilename>
  </FileCreate>
  <ProcessAccess onmatch="exclude">
    <SourceImage condition="end with">vmms.exe</SourceImage>
  </ProcessAccess>
</RuleGroup>
```

### Agen backup

```xml
<RawAccessRead onmatch="exclude">
  <Image condition="is">C:\Program Files\Veeam\Backup and Replication\VeeamAgent.exe</Image>
</RawAccessRead>
```

## 2.8 Terapkan konfigurasi

```powershell
.\Sysmon64.exe -c C:\Windows\Sysmon\sysmonconfig.xml
```

## 2.9 Verifikasi (Server Core, tanpa Event Viewer)

```powershell
Get-Service Sysmon64, SysmonDrv | Format-Table Name, Status, StartType
.\Sysmon64.exe -c | Select-Object -First 40
wevtutil gl Microsoft-Windows-Sysmon/Operational

Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20000 |
  Group-Object Id | Sort-Object Count -Descending |
  Select-Object Count, Name | Format-Table -AutoSize
```

## 2.10 Ukur EPS aktual

```powershell
$start = (Get-Date).AddMinutes(-10)
$count = (Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-Sysmon/Operational'; StartTime=$start
}).Count
"EPS rata-rata: {0:N1}" -f ($count / 600)
```

## 2.11 Health check terjadwal

```powershell
New-EventLog -LogName Application -Source "SysmonHealthCheck" -ErrorAction SilentlyContinue

$script = @'
$svc = Get-Service Sysmon64, SysmonDrv -ErrorAction SilentlyContinue
if ($svc | Where-Object Status -ne 'Running') {
    Write-EventLog -LogName Application -Source "SysmonHealthCheck" `
      -EventId 9001 -EntryType Error -Message "Sysmon service/driver tidak berjalan"
}
'@
$script | Out-File C:\Windows\Sysmon\healthcheck.ps1 -Encoding UTF8

$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
             -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\Windows\Sysmon\healthcheck.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 07:00
Register-ScheduledTask -TaskName "Sysmon Health Check" -Action $action -Trigger $trigger `
  -User "SYSTEM" -RunLevel Highest -Force
```

## 2.12 Deployment massal via GPO Startup Script

Struktur OU dan GPO:

```
OU=Workstations           -> GPO Sysmon-Workstation -> sysmonconfig-workstation.xml
OU=Servers
  |- OU=DomainControllers -> GPO Sysmon-DC          -> sysmonconfig-dc.xml
  |- OU=WebServers        -> GPO Sysmon-IIS         -> sysmonconfig-iis.xml
  |- OU=FileServers       -> GPO Sysmon-File        -> sysmonconfig-file.xml
  |- OU=SQLServers        -> GPO Sysmon-SQL         -> sysmonconfig-sql.xml
```

Script (idempoten):

```batch
@echo off
set SRC=\\dc01\netlogon\sysmon
set DST=C:\Windows\Sysmon
set CFG=sysmonconfig-dc.xml

sc query Sysmon64 >nul 2>&1
if %ERRORLEVEL%==0 goto UPDATE

:INSTALL
if not exist "%DST%" mkdir "%DST%"
copy /Y "%SRC%\Sysmon64.exe" "%DST%\"
copy /Y "%SRC%\%CFG%"        "%DST%\sysmonconfig.xml"
"%DST%\Sysmon64.exe" -accepteula -i "%DST%\sysmonconfig.xml"
wevtutil sl Microsoft-Windows-Sysmon/Operational /ms:2147483648
goto END

:UPDATE
copy /Y "%SRC%\%CFG%" "%DST%\sysmonconfig.xml"
"%DST%\Sysmon64.exe" -c "%DST%\sysmonconfig.xml"

:END
exit /b 0
```

## 2.13 Prosedur upgrade versi Sysmon

```powershell
Set-Location C:\Windows\Sysmon
.\Sysmon64.exe -u force
Copy-Item "$env:TEMP\Sysmon\Sysmon64.exe" -Destination "C:\Windows\Sysmon\" -Force
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
Get-Service Sysmon64, SysmonDrv
```

## 2.14 Checklist

```
[ ] Uji 1 server non-produksi per role, minimal 7 hari
[ ] EPS aktual terukur (2.10)
[ ] ACL direktori Sysmon dibatasi SYSTEM + Administrators
[ ] ArchiveDirectory di luar volume data / volume ntds.dit
[ ] Ukuran event log dinaikkan sesuai role
[ ] Exclude AV/EDR, agen backup, agen monitoring diterapkan
[ ] Filter per role diterapkan
[ ] ProcessAccess ke lsass.exe TIDAK dikecualikan
[ ] SMB signing tidak memblokir jalur distribusi (2.3)
[ ] client_buffer dinaikkan pada server ber-EPS tinggi (3.3)
[ ] Alert tampering Event ID 4 dan 16 aktif
[ ] Health check terjadwal aktif (2.11)
[ ] Hash binary, hash konfigurasi, output `-c` didokumentasikan
```

---

# 3. Konfigurasi Wazuh Agent

## 3.1 Tambahkan Sysmon channel

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

## 3.2 Filter di sisi agent (opsional, server ber-EPS tinggi)

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
  <query>Event/System[EventID != 5 and EventID != 22]</query>
</localfile>
```

## 3.3 Naikkan client buffer

```xml
<client_buffer>
  <disabled>no</disabled>
  <queue_size>100000</queue_size>
  <events_per_second>1000</events_per_second>
</client_buffer>
```

## 3.4 Restart agent

```powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

## 3.5 Verifikasi di agent

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

## 3.6 Verifikasi di manager

```bash
/var/ossec/bin/agent_control -l
tail -f /var/ossec/logs/alerts/alerts.json | grep -i sysmon
```

## 3.7 Uji rule dengan logtest

```bash
/var/ossec/bin/wazuh-logtest
```

Tempelkan satu sample event Sysmon, catat nomor `id` rule yang cocok untuk dipakai sebagai `if_sid`.

## 3.8 Tambahkan custom rule

Edit `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="sysmon,windows,">

  <!-- Credential dumping -->
  <rule id="100500" level="12">
    <if_sid>61609</if_sid>
    <field name="win.eventdata.targetImage">\\\\lsass.exe$</field>
    <description>Sysmon: akses ke LSASS - indikasi credential dumping</description>
    <mitre><id>T1003.001</id></mitre>
  </rule>

  <!-- Webshell / RCE di IIS -->
  <rule id="100510" level="14">
    <if_sid>61603</if_sid>
    <field name="win.eventdata.parentImage">w3wp.exe$</field>
    <description>Sysmon: w3wp.exe menurunkan proses anak - indikasi webshell/RCE</description>
    <mitre><id>T1505.003</id></mitre>
  </rule>

  <!-- Ekstraksi NTDS.dit -->
  <rule id="100511" level="15">
    <if_sid>61603</if_sid>
    <field name="win.eventdata.image">ntdsutil.exe|esentutl.exe|vssadmin.exe</field>
    <description>Sysmon: eksekusi tool ekstraksi NTDS.dit</description>
    <mitre><id>T1003.003</id></mitre>
  </rule>

  <!-- xp_cmdshell -->
  <rule id="100512" level="14">
    <if_sid>61603</if_sid>
    <field name="win.eventdata.parentImage">sqlservr.exe$</field>
    <description>Sysmon: sqlservr.exe menurunkan proses anak - dugaan xp_cmdshell</description>
    <mitre><id>T1505.001</id></mitre>
  </rule>

  <!-- Tampering Sysmon -->
  <rule id="100513" level="15">
    <if_sid>61644</if_sid>
    <description>Sysmon: perubahan status service atau konfigurasi - dugaan tampering</description>
    <mitre><id>T1562.001</id></mitre>
  </rule>

</group>
```

Nomor `if_sid` di atas mengacu pada ruleset default dan dapat berbeda antar versi. Verifikasi dengan 3.7 sebelum dipakai.

## 3.9 Validasi dan restart manager

```bash
/var/ossec/bin/wazuh-logtest -t
systemctl restart wazuh-manager
systemctl status wazuh-manager
```

## 3.10 Verifikasi alert masuk

```bash
grep -c '"win.system.providerName":"Microsoft-Windows-Sysmon"' /var/ossec/logs/alerts/alerts.json
tail -f /var/ossec/logs/alerts/alerts.log
```

---

## Referensi

- Sysmon: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- SwiftOnSecurity/sysmon-config: https://github.com/SwiftOnSecurity/sysmon-config
- olafhartong/sysmon-modular: https://github.com/olafhartong/sysmon-modular
- Neo23x0/sysmon-config: https://github.com/Neo23x0/sysmon-config
- TrustedSec Sysmon Community Guide: https://github.com/trustedsec/SysmonCommunityGuide
- Credential Guard: https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/
- SMB security hardening: https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-security-hardening
