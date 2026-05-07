# Windows Server 2025 İleri Seviye DNS Yapılandırması Rehberi

Permalink: Windows Server 2025 İleri Seviye DNS Yapılandırması Rehberi

## DNS Manager ile Güvenlik, Performans ve Yüksek Erişilebilirlik Yapılandırmaları

Bu rehber, Windows Server 2025 sisteminde DNS Server rolü ve temel zone/kayıt yapılandırmaları **zaten tamamlanmış** olduğu varsayılarak; DNSSEC, Conditional Forwarding, DNS Policies, Zone Transfer güvenliği, logging, performans optimizasyonu ve yüksek erişilebilirlik gibi ileri seviye konuları adım adım açıklar.

* * *

## 📑 İçindekiler

Permalink: 📑 İçindekiler

- [DNSSEC ile DNS Güvenliği](#-dnssec-ile-dns-güvenliği)
- [Zone Transfer Güvenliği ve Secondary DNS](#-zone-transfer-güvenliği-ve-secondary-dns)
- [Conditional Forwarding Yapılandırması](#-conditional-forwarding-yapılandırması)
- [DNS Policies ile Akıllı Yönlendirme](#-dns-policies-ile-akıllı-yönlendirme)
- [Query Logging ve Güvenlik Denetimi](#-query-logging-ve-güvenlik-denetimi)
- [DNS Performans Optimizasyonu](#-dns-performans-optimizasyonu)
- [Yüksek Erişilebilirlik: Secondary Zone Replikasyonu](#-yüksek-erişilebilirlik-secondary-zone-replikasyonu)
- [Split-Horizon DNS (Views)](#-split-horizon-dns-views)
- [PowerShell ile İleri Seviye Otomasyon](#-powershell-ile-ileri-seviye-otomasyon)
- [Sık Karşılaşılan İleri Seviye Sorunlar](#-sık-karşılaşılan-ileri-seviye-sorunlar)

* * *

## 🔐 DNSSEC ile DNS Güvenliği

Permalink: 🔐 DNSSEC ile DNS Güvenliği

DNSSEC (DNS Security Extensions), DNS yanıtlarının dijital imzalanmasını sağlayarak, "man-in-the-middle" saldırılarına ve DNS spoofing'e karşı koruma sunar.

### DNSSEC Ön Koşulları

```powershell
# Zone'un AD-integrated olduğundan emin olun
Get-DnsServerZone -Name "serifselen.local" | Select-Object IsDsIntegrated

# Zone transfer ayarlarını kontrol edin
Get-DnsServerZone -Name "serifselen.local" | Select-Object SecureSecondaries
```

### Adım 1: Zone Signing Wizard ile DNSSEC Etkinleştirme

**1. DNS Manager'da Zone'a Sağ Tıklayın:**

`serifselen.local` zone'una sağ tıklayın > **DNSSEC** > **Sign the Zone...**

**2. Signing Sihirbazı Başlar:**

![DNSSEC Signing Wizard Başlangıç](images/dnssec-wizard-start.png)

**Next** butonuna tıklayın.

**3. Key Storage Provider Seçimi:**

- ☑️ **Microsoft Software Key Storage Provider** (önerilen - test ortamı)
- ⚪ Hardware Security Module (HSM) (üretim ortamı)

**4. Key Master Seçimi:**

- ☑️ **This server** (Bu sunucu anahtarları yönetir)

**5. Key Parametreleri:**

| Parametre | Önerilen Değer | Açıklama |
|-----------|---------------|----------|
| **Algorithm** | ECDSAP256SHA256 | Modern, güvenli algoritma |
| **Key Length** | 256 bit | ECDSA için standart |
| **KSK Lifetime** | 365 gün | Key Signing Key ömrü |
| **ZSK Lifetime** | 30 gün | Zone Signing Key ömrü |

**6. Rollover Ayarları:**

- ☑️ **Enable automatic key rollover** (otomatik anahtar değişimi)

**7. Signing Tamamlanır:**

Finish butonuna tıklayın. Zone imzalanmaya başlar.

### Adım 2: DNSSEC Durumunu Doğrulama

```powershell
# Zone DNSSEC durumunu kontrol etme
Get-DnsServerDnsSecZone -Name "serifselen.local"

# Anahtarları listeleyerek doğrulama
Get-DnsServerDnsSecKey -ZoneName "serifselen.local" | 
    Select-Object KeyType, Algorithm, Length, CreatedTime

# İmza kayıtlarını (RRSIG) sorgulama
Get-DnsServerResourceRecord -ZoneName "serifselen.local" -RRType RRSIG | 
    Select-Object HostName, RecordData | Format-Table -AutoSize
```

### Adım 3: DS Record Oluşturma (Parent Zone İçin)

```powershell
# DS record için gerekli bilgileri export etme
Export-DnsServerDnsSecPublicKey -ZoneName "serifselen.local" -FilePath "C:\ds_record.txt"

# DS record formatı (registrar'a gönderilecek):
# serifselen.local. IN DS 12345 13 2 ABCDEF1234567890...
```

> ⚠️ **Önemli:** DS record'ı, domain registrar'ınızın kontrol paneline eklenmelidir. Bu işlem tamamlanmadan DNSSEC tam olarak çalışmaz.

### Adım 4: İstemci Tarafında DNSSEC Validation

```powershell
# DNSSEC validation'ı etkinleştirme (istemci)
Set-DnsClient -ConnectionSpecificSuffix "serifselen.local" -ValidateDNSSEC $true

# DNSSEC ile sorgu testi
Resolve-DnsName -Name "www.serifselen.local" -Type A -DnssecOk
```

**Başarılı DNSSEC Yanıtı:**

```
Name                           Type   TTL   Section    RecordData
----                           ----   ---   -------    ----------
www.serifselen.local           A      3600  Answer     10.0.2.15
www.serifselen.local           RRSIG  3600  Answer     [imza verisi]
```

* * *

## 🔗 Zone Transfer Güvenliği ve Secondary DNS

Permalink: 🔗 Zone Transfer Güvenliği ve Secondary DNS

Zone transfer, primary DNS sunucusundan secondary DNS sunucusuna zone verilerinin kopyalanması işlemidir. Bu işlem doğru kısıtlanmazsa, saldırganlar tüm DNS kayıtlarınızı ele geçirebilir.

### Zone Transfer Kısıtlama (Primary DNS)

**1. DNS Manager'da Zone Özellikleri:**

`serifselen.local` zone'una sağ tıklayın > **Properties** > **Zone Transfers** sekmesi

![Zone Transfer Settings](images/zone-transfer-settings.png)

**2. Transfer İzinlerini Yapılandırın:**

- ☑️ **Allow zone transfers**
- ⚪ To any server (❌ GÜVENLİK RİSKİ)
- ☑️ **Only to servers listed on the Name Servers tab** (önerilen)
- ⚪ Only to the following servers: (manuel IP ekleme)

**3. Name Servers Tab'ını Kontrol Edin:**

**Name Servers** sekmesinde sadece yetkili secondary DNS sunucularının IP/FQDN'leri olmalıdır.

### PowerShell ile Zone Transfer Güvenliği

```powershell
# Zone transfer'ı sadece secure servers'a izin verme
Set-DnsServerPrimaryZone -Name "serifselen.local" `
    -SecureSecondaries "TransferToSecureServers"

# Manuel IP listesi ile kısıtlama
Set-DnsServerPrimaryZone -Name "serifselen.local" `
    -SecondaryServers "10.0.2.50", "10.0.2.51"

# Mevcut ayarı doğrulama
Get-DnsServerZone -Name "serifselen.local" | 
    Select-Object ZoneName, SecureSecondaries, SecondaryServers
```

### Secondary DNS Zone Yapılandırma

**Secondary DNS Sunucusunda:**

**1. New Zone Sihirbazı:**

- Zone Type: **Secondary zone**
- Zone name: `serifselen.local`
- Master DNS Servers: `10.0.2.10` (Primary DNS IP)

**2. Zone Transfer Başlatma (Manuel):**

```powershell
# Secondary'de manuel zone transfer başlatma
Invoke-DnsServerZoneTransfer -Name "serifselen.local" -FullTransfer

# Transfer durumunu kontrol etme
Get-DnsServerZone -Name "serifselen.local" | 
    Select-Object ZoneName, IsAutoCreated, LastSuccessfulSoaCheck
```

**3. Otomatik Replikasyon Kontrolü:**

```powershell
# Secondary zone refresh interval ayarı
Set-DnsServerSecondaryZone -Name "serifselen.local" `
    -RefreshInterval 01:00:00  # Her 1 saatte bir kontrol

# Notify ayarı (Primary'de)
Set-DnsServerPrimaryZone -Name "serifselen.local" `
    -Notify "NotifyServers" -NotifyServers "10.0.2.50"
```

* * *

## 🎯 Conditional Forwarding Yapılandırması

Permalink: 🎯 Conditional Forwarding Yapılandırması

Conditional Forwarding, belirli domain'lere ait sorguların, yerel DNS sunucusu tarafından çözümlenememesi durumunda, belirtilen başka bir DNS sunucusuna yönlendirilmesini sağlar.

**Kullanım Senaryoları:**
- Partner şirket domain'leri (`partner.com`)
- Şube ofisleri (`branch.serifselen.local`)
- Cloud servisleri (`internal.azure.local`)

### Adım Adım Conditional Forwarder Ekleme

**1. DNS Manager'da Conditional Forwarders:**

Sol panelde **Conditional Forwarders** düğümüne sağ tıklayın > **New Conditional Forwarder...**

**2. Domain ve DNS Sunucuları:**

![New Conditional Forwarder](images/conditional-forwarder-new.png)

**Alanlar:**

- **DNS Domain:** `partner.com`
- **Master servers:** 
  - `172.16.10.5` (Partner DNS IP)
  - **Enter** ile ekleyin
- ☑️ **Store this conditional forwarder in Active Directory** (AD ortamı için)

**3. Replication Scope (AD Entegrasyonu):**

- ☑️ **To all DNS servers running on domain controllers in this domain**

**4. OK** butonuna tıklayın.

### PowerShell ile Conditional Forwarder

```powershell
# Conditional forwarder ekleme
Add-DnsServerConditionalForwarderZone `
    -Name "partner.com" `
    -MasterServers "172.16.10.5", "172.16.10.6" `
    -ReplicationScope "Domain"

# Conditional forwarder'ları listeleyerek doğrulama
Get-DnsServerZone -Conditional | 
    Select-Object ZoneName, MasterServers, ReplicationScope

# Conditional forwarder test etme
Resolve-DnsName -Name "app.partner.com" -Server "10.0.2.10"
```

### Conditional Forwarder Yönetimi

```powershell
# Forwarder güncelleme (yeni DNS sunucusu ekleme)
Set-DnsServerConditionalForwarderZone `
    -Name "partner.com" `
    -MasterServers "172.16.10.5", "172.16.10.6", "172.16.10.7"

# Forwarder silme
Remove-DnsServerConditionalForwarderZone -Name "partner.com" -Force

# Forwarder istatistiklerini görüntüleme
Get-DnsServerZoneStatistics -Name "partner.com"
```

* * *

## 🧠 DNS Policies ile Akıllı Yönlendirme

Permalink: 🧠 DNS Policies ile Akıllı Yönlendirme

DNS Policies, gelen DNS sorgularına göre dinamik yanıtlar oluşturmanızı sağlar. Client subnet'ine, saat dilimine, query type'ına göre farklı IP'ler döndürebilirsiniz.

### Kullanım Senaryoları

| Senaryo | Policy Türü | Açıklama |
|---------|-------------|----------|
| Geo-Load Balancing | Client Subnet | Avrupa'dan gelenlere Frankfurt, ABD'den gelenlere NY sunucusu |
| Maintenance Window | Time Range | Bakım saatinde "under maintenance" IP'si döndür |
| Bot Koruması | Query Type/Rate | Şüpheli sorgu oranına sahip IP'leri engelle |
| Internal/External | Client Subnet | İç ağdan gelenlere private IP, dışarıdan public IP |

### Adım 1: Client Subnet Policy Oluşturma

**PowerShell ile Geo-Load Balancing Policy:**

```powershell
# Avrupa subnet'leri için policy
Add-DnsServerQueryResolutionPolicy `
    -Name "Europe-Policy" `
    -Action ALLOW `
    -ClientSubnet "eq,192.168.10.0/24;192.168.11.0/24" `
    -ZoneScope "Europe-Scope" `
    -ZoneName "serifselen.local"

# ABD subnet'leri için policy
Add-DnsServerQueryResolutionPolicy `
    -Name "US-Policy" `
    -Action ALLOW `
    -ClientSubnet "eq,10.10.1.0/24;10.10.2.0/24" `
    -ZoneScope "US-Scope" `
    -ZoneName "serifselen.local"
```

### Adım 2: Zone Scope'ları Oluşturma

```powershell
# Europe Scope: Avrupa sunucuları
Add-DnsServerZoneScope `
    -Name "Europe-Scope" `
    -ZoneName "serifselen.local"

Add-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -ZoneScope "Europe-Scope" `
    -Name "www" `
    -IPv4Address "10.0.2.100"  # Frankfurt sunucusu

# US Scope: ABD sunucuları
Add-DnsServerZoneScope `
    -Name "US-Scope" `
    -ZoneName "serifselen.local"

Add-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -ZoneScope "US-Scope" `
    -Name "www" `
    -IPv4Address "10.0.3.100"  # New York sunucusu
```

### Adım 3: Time-Based Policy (Bakım Penceresi)

```powershell
# Bakım saati için time range tanımlama
Add-DnsServerQueryResolutionPolicy `
    -Name "Maintenance-Window" `
    -Action ALLOW `
    -TimeRange "02:00-04:00" `
    -ZoneScope "Maintenance-Scope" `
    -ZoneName "serifselen.local"

# Maintenance Scope: "under maintenance" IP'si
Add-DnsServerZoneScope `
    -Name "Maintenance-Scope" `
    -ZoneName "serifselen.local"

Add-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -ZoneScope "Maintenance-Scope" `
    -Name "www" `
    -IPv4Address "10.0.2.254"  # Maintenance sayfası
```

### Policy'leri Yönetme ve Test Etme

```powershell
# Tüm policy'leri listeleyerek görüntüleme
Get-DnsServerQueryResolutionPolicy -ZoneName "serifselen.local" | 
    Select-Object Name, ProcessingOrder, Action, ClientSubnet, ZoneScope

# Belirli bir policy'yi düzenleme
Set-DnsServerQueryResolutionPolicy `
    -Name "Europe-Policy" `
    -ClientSubnet "eq,192.168.10.0/24;192.168.11.0/24;192.168.12.0/24"

# Policy'yi devre dışı bırakma
Set-DnsServerQueryResolutionPolicy `
    -Name "Maintenance-Window" `
    -Enabled $false

# Policy'yi silme
Remove-DnsServerQueryResolutionPolicy `
    -Name "US-Policy" `
    -ZoneName "serifselen.local" `
    -Force

# Test: Farklı subnet'ten sorgu simülasyonu
# (Not: Gerçek test için farklı subnet'ten istemci gerekir)
Resolve-DnsName -Name "www.serifselen.local" -Server "10.0.2.10"
```

* * *

## 📋 Query Logging ve Güvenlik Denetimi

Permalink: 📋 Query Logging ve Güvenlik Denetimi

DNS sorgularının loglanması, güvenlik denetimi, troubleshooting ve anomali tespiti için kritik öneme sahiptir.

### Query Logging Etkinleştirme

**Yöntem 1: DNS Manager ile**

**1. DNS Server Properties:**

DNS Manager'da sunucu adına sağ tıklayın > **Properties** > **Debug Logging** sekmesi

**2. Logging Ayarları:**

![DNS Debug Logging](images/dns-debug-logging.png)

- ☑️ **Log packets for debugging**
- **Log file location:** `C:\Windows\System32\dns\dns.log`
- **Packet direction:** ☑️ Incoming ☑️ Outgoing
- **Transport protocols:** ☑️ UDP ☑️ TCP
- **Packet details:** ☑️ Queries/Transfers ☑️ Answers ☑️ Notifications

**3. Apply > OK**

> ⚠️ **Uyarı:** Debug logging yüksek disk kullanımı oluşturabilir. Sadece troubleshooting veya güvenlik denetimi için geçici olarak etkinleştirin.

**Yöntem 2: PowerShell ile**

```powershell
# Query logging'i etkinleştirme
Set-DnsServerDiagnostics `
    -LogFilePath "C:\Windows\System32\dns\dns.log" `
    -LogFullPackets $true `
    -LogIncomingPackets $true `
    -LogOutgoingPackets $true `
    -LogQueries $true `
    -LogAnswers $true `
    -EnableLogFileRollover $true `
    -MaxLogFiles 10

# Mevcut diagnostic ayarlarını görüntüleme
Get-DnsServerDiagnostics | Format-List *
```

### Log Analizi ve Anomali Tespiti

**PowerShell ile Log Analizi:**

```powershell
# DNS log dosyasını okuma ve analiz etme
$LogFile = "C:\Windows\System32\dns\dns.log"

# En çok sorgu yapan IP'ler
Get-Content $LogFile | 
    Select-String "RECV" | 
    ForEach-Object { ($_ -split '\s+')[4] } | 
    Sort-Object | Get-Unique -Count | 
    Sort-Object Count -Descending | 
    Select-Object -First 10

# NXDOMAIN hataları (olmayan domain sorguları)
Get-Content $LogFile | 
    Select-String "NXDOMAIN" | 
    ForEach-Object { ($_ -split '\s+')[-1] } | 
    Sort-Object | Get-Unique -Count | 
    Sort-Object Count -Descending | 
    Select-Object -First 10

# Zone transfer denemeleri
Get-Content $LogFile | 
    Select-String "AXFR" | 
    ForEach-Object { 
        $parts = $_ -split '\s+'
        [PSCustomObject]@{
            Time = $parts[0]
            SourceIP = $parts[4]
            Zone = $parts[-1]
        }
    }
```

### Event Log ile DNS İzleme

```powershell
# DNS Server event log'undan hata ve uyarıları çekme
Get-WinEvent -LogName "DNS Server" -FilterXPath "*[System[(Level=2 or Level=3)]]" -MaxEvents 50 | 
    Select-Object TimeCreated, Id, LevelDisplayName, Message | 
    Format-Table -Wrap

# Belirli Event ID'leri için filtreleme
# Event ID 501: Zone transfer başladı
# Event ID 502: Zone transfer tamamlandı
# Event ID 503: Zone transfer başarısız
# Event ID 708: Dynamic update başarısız

Get-WinEvent -LogName "DNS Server" -FilterHashtable @{
    Id = 501, 502, 503, 708
    StartTime = (Get-Date).AddHours(-24)
} | Select-Object TimeCreated, Id, Message
```

### Otomatik Alert Sistemi (Basit)

```powershell
# DNS güvenlik alert scripti
function Test-DnsSecurityAlerts {
    param([int]$Threshold = 100)
    
    $LogPath = "C:\Windows\System32\dns\dns.log"
    $LastHour = (Get-Date).AddHours(-1)
    
    # Son 1 saatte aynı IP'den gelen sorgu sayısı
    $SuspiciousIPs = Get-Content $LogPath | 
        Where-Object { $_ -match "RECV" } |
        ForEach-Object { ($_ -split '\s+')[4] } |
        Group-Object | 
        Where-Object { $_.Count -gt $Threshold } |
        Select-Object Name, Count
    
    if ($SuspiciousIPs) {
        Write-Warning "⚠️ Şüpheli aktivite tespit edildi!"
        $SuspiciousIPs | ForEach-Object {
            Write-Warning "IP: $($_.Name) - Sorgu: $($_.Count)"
        }
        
        # Email alert (SMTP ayarları gerektirir)
        # Send-MailMessage -To "admin@serifselen.local" ...
    }
    else {
        Write-Host "✓ Anomali tespit edilmedi" -ForegroundColor Green
    }
}

# Scheduled Task ile her 15 dakikada bir çalıştırma
# Register-ScheduledJob -Name "DNS-Security-Monitor" -ScriptBlock { Test-DnsSecurityAlerts } -Trigger (New-JobTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 15))
```

* * *

## ⚡ DNS Performans Optimizasyonu

Permalink: ⚡ DNS Performans Optimizasyonu

Yüksek trafikli ortamlarda DNS sunucusunun performansını artırmak için aşağıdaki ayarları uygulayabilirsiniz.

### Cache ve TTL Optimizasyonu

```powershell
# Cache ayarlarını optimize etme
Set-DnsServerCache `
    -MaxCacheSize 512MB `
    -MaxNegativeTtl 300 `
    -MaxTtl 86400 `
    -PollutionProtection $true

# Zone TTL değerlerini optimize etme
# Düşük TTL: Sık değişen kayıtlar (load balancer, failover)
# Yüksek TTL: Stabil kayıtlar (static servers)

Set-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -Name "lb-app" `
    -IPv4Address "10.0.2.200" `
    -TimeToLive 00:05:00  # 5 dakika TTL

Set-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -Name "dc1" `
    -IPv4Address "10.0.2.10" `
    -TimeToLive 24:00:00  # 24 saat TTL
```

### Recursive Query Optimizasyonu

```powershell
# Recursive query ayarları
Set-DnsServerRecursion `
    -EnableRecursion $true `
    -AdditionalTimeout 1000 `
    -SecureResponses $true

# Recursive query için forwarder optimizasyonu
Set-DnsServerForwarder `
    -IpAddress "8.8.8.8", "1.1.1.1", "9.9.9.9" `
    -EnableReordering $true `
    -Timeout 3

# Forwarder istatistiklerini izleme
Get-DnsServerForwarder | Select-Object IpAddress, Timeout, EnableReordering
```

### Performans Sayaçları ile İzleme

```powershell
# DNS performans sayaçlarını okuma
$Counters = @(
    "\DNS\Total Query Received/sec",
    "\DNS\Total Response Sent/sec",
    "\DNS\Recursive Queries/sec",
    "\DNS\Cache Hits/sec",
    "\DNS\Cache Misses/sec",
    "\Memory\Available MBytes",
    "\Processor(_Total)\% Processor Time"
)

# 5'er saniye aralıklarla 3 örnek al
Get-Counter -Counter $Counters -SampleInterval 5 -MaxSamples 3 | 
    Select-Object -ExpandProperty CounterSamples | 
    Select-Object Path, CookedValue, Timestamp |
    Format-Table -AutoSize
```

### Baseline ve Alert Threshold'ları

| Performans Metriği | Normal Aralık | Alert Threshold | Aksiyon |
|-------------------|---------------|-----------------|---------|
| Query/sec | < 1000 | > 5000 | Rate limiting kontrolü |
| Cache Hit Ratio | > 80% | < 50% | TTL ayarlarını gözden geçir |
| Recursive Query/sec | < 100 | > 500 | Forwarder optimizasyonu |
| Response Time (ms) | < 50ms | > 200ms | Sunucu kaynaklarını kontrol et |

```powershell
# Baseline raporu oluşturma
function Get-DnsPerformanceBaseline {
    $Metrics = @{}
    
    $Metrics.QueryRate = (Get-Counter "\DNS\Total Query Received/sec").CounterSamples.CookedValue
    $Metrics.ResponseRate = (Get-Counter "\DNS\Total Response Sent/sec").CounterSamples.CookedValue
    $Metrics.CacheHits = (Get-Counter "\DNS\Cache Hits/sec").CounterSamples.CookedValue
    $Metrics.CacheMisses = (Get-Counter "\DNS\Cache Misses/sec").CounterSamples.CookedValue
    
    $Metrics.CacheHitRatio = if ($Metrics.CacheHits + $Metrics.CacheMisses -gt 0) {
        [math]::Round(($Metrics.CacheHits / ($Metrics.CacheHits + $Metrics.CacheMisses)) * 100, 2)
    } else { 0 }
    
    return [PSCustomObject]$Metrics
}

# Baseline'ı çalıştırma ve sonuçları görüntüleme
$Baseline = Get-DnsPerformanceBaseline
$Baseline | Format-List *

# Alert kontrolü
if ($Baseline.CacheHitRatio -lt 50) {
    Write-Warning "⚠️ Cache hit ratio düşük: $($Baseline.CacheHitRatio)%"
}
```

* * *

## 🔄 Yüksek Erişilebilirlik: Secondary Zone Replikasyonu

Permalink: 🔄 Yüksek Erişilebilirlik: Secondary Zone Replikasyonu

DNS servisinin kesintisiz çalışması için primary-secondary replikasyon yapısı kurulmalıdır.

### Replikasyon Topolojisi

```
                    ┌─────────────────┐
                    │  Primary DNS    │
                    │  10.0.2.10      │
                    │  serifselen.local │
                    └────────┬────────┘
                             │ Zone Transfer
            ┌────────────────┴────────────────┐
            ▼                                 ▼
┌────────────────────┐      ┌────────────────────┐
│ Secondary DNS #1   │      │ Secondary DNS #2   │
│ 10.0.2.50          │      │ 10.0.2.51          │
│ serifselen.local   │      │ serifselen.local   │
│ (Read-Only)        │      │ (Read-Only)        │
└────────────────────┘      └────────────────────┘
```

### Secondary DNS Kurulumu (Adım Adım)

**Secondary Sunucuda:**

**1. DNS Server Rolünü Kurun:**

```powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools
```

**2. Secondary Zone Oluşturun:**

DNS Manager > Reverse Lookup Zones'a sağ tık > **New Zone...**

- Zone Type: **Secondary zone**
- Zone name: `serifselen.local`
- Master DNS Servers: `10.0.2.10` (Primary IP)
- **Finish**

**3. Zone Transfer'ı Tetikleme:**

```powershell
# Manuel zone transfer başlatma
Invoke-DnsServerZoneTransfer -Name "serifselen.local" -FullTransfer

# Transfer durumunu kontrol etme
Get-DnsServerZone -Name "serifselen.local" | 
    Select-Object ZoneName, IsReadOnly, LastSuccessfulSoaCheck, NextSoaPollTime
```

### Primary DNS'te Notify ve Replikasyon Ayarları

```powershell
# Secondary sunucuları notify listesi ekleme
Set-DnsServerPrimaryZone `
    -Name "serifselen.local" `
    -Notify "NotifyServers" `
    -NotifyServers "10.0.2.50", "10.0.2.51"

# Refresh interval ayarı (secondary'lerin ne sıklıkla kontrol edeceği)
Set-DnsServerSecondaryZone `
    -Name "serifselen.local" `
    -RefreshInterval 00:15:00  # 15 dakika

# Expire interval (secondary'nin veriyi ne kadar süre tutacağı)
Set-DnsServerSecondaryZone `
    -Name "serifselen.local" `
    -ExpireInterval 1.00:00:00  # 1 gün
```

### Replikasyon Sağlığını İzleme

```powershell
# Tüm DNS sunucularında zone durumunu kontrol etme
$Servers = @("10.0.2.10", "10.0.2.50", "10.0.2.51")

foreach ($Server in $Servers) {
    $Zone = Get-DnsServerZone -Name "serifselen.local" -ComputerName $Server -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Server = $Server
        ZoneName = $Zone.ZoneName
        Type = $Zone.ZoneType
        IsReadOnly = $Zone.IsReadOnly
        LastCheck = $Zone.LastSuccessfulSoaCheck
        Status = if ($Zone) { "OK" } else { "ERROR" }
    }
} | Format-Table -AutoSize

# Zone serial numarası tutarlılığını kontrol etme
$PrimarySerial = (Get-DnsServerResourceRecord -ZoneName "serifselen.local" -RRType SOA -ComputerName "10.0.2.10").RecordData.SerialNumber

foreach ($Server in @("10.0.2.50", "10.0.2.51")) {
    $SecondarySerial = (Get-DnsServerResourceRecord -ZoneName "serifselen.local" -RRType SOA -ComputerName $Server -ErrorAction SilentlyContinue).RecordData.SerialNumber
    if ($SecondarySerial -eq $PrimarySerial) {
        Write-Host "✓ $Server : Serial uyumlu ($SecondarySerial)" -ForegroundColor Green
    } else {
        Write-Warning "✗ $Server : Serial uyumsuz (Beklenen: $PrimarySerial, Alınan: $SecondarySerial)"
    }
}
```

* * *

## 🌐 Split-Horizon DNS (Views)

Permalink: 🌐 Split-Horizon DNS (Views)

Split-Horizon DNS, aynı domain adı için farklı client'lara farklı IP adresleri döndürmenizi sağlar. İç ağ ve dış ağ için farklı yanıtlar üretmek için kullanılır.

### Senaryo: Internal vs External Access

| Client Kaynağı | www.serifselen.local Yanıtı | Açıklama |
|---------------|----------------------------|----------|
| İç Ağ (192.168.1.0/24) | 10.0.2.100 (Private IP) | Direct internal access |
| Dış Ağ (Public) | 203.0.113.10 (Public IP) | Firewall/NAT üzerinden |

### Adım 1: Zone Scopes Oluşturma

```powershell
# Internal Scope
Add-DnsServerZoneScope -Name "Internal-Scope" -ZoneName "serifselen.local"
Add-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -ZoneScope "Internal-Scope" `
    -Name "www" `
    -IPv4Address "10.0.2.100"

# External Scope
Add-DnsServerZoneScope -Name "External-Scope" -ZoneName "serifselen.local"
Add-DnsServerResourceRecordA `
    -ZoneName "serifselen.local" `
    -ZoneScope "External-Scope" `
    -Name "www" `
    -IPv4Address "203.0.113.10"
```

### Adım 2: Client Subnet Policy'leri Tanımlama

```powershell
# Internal network policy
Add-DnsServerQueryResolutionPolicy `
    -Name "Internal-View" `
    -Action ALLOW `
    -ClientSubnet "eq,192.168.1.0/24;10.0.0.0/8" `
    -ZoneScope "Internal-Scope" `
    -ZoneName "serifselen.local" `
    -ProcessingOrder 1

# External network policy (default)
Add-DnsServerQueryResolutionPolicy `
    -Name "External-View" `
    -Action ALLOW `
    -ClientSubnet "ne,192.168.1.0/24;ne,10.0.0.0/8" `
    -ZoneScope "External-Scope" `
    -ZoneName "serifselen.local" `
    -ProcessingOrder 2
```

### Adım 3: Policy'leri Test Etme

```powershell
# Policy listesini görüntüleme
Get-DnsServerQueryResolutionPolicy -ZoneName "serifselen.local" | 
    Select-Object Name, ProcessingOrder, Action, ClientSubnet, ZoneScope | 
    Format-Table -AutoSize

# Internal scope kayıtlarını doğrulama
Get-DnsServerResourceRecord `
    -ZoneName "serifselen.local" `
    -ZoneScope "Internal-Scope" `
    -Name "www" `
    -RRType A

# External scope kayıtlarını doğrulama
Get-DnsServerResourceRecord `
    -ZoneName "serifselen.local" `
    -ZoneScope "External-Scope" `
    -Name "www" `
    -RRType A
```

> 💡 **Not:** Gerçek test için farklı subnet'lerden istemcilerle `nslookup` veya `Resolve-DnsName` kullanmanız gerekir.

* * *

## 🖥️ PowerShell ile İleri Seviye Otomasyon

Permalink: 🖥️ PowerShell ile İleri Seviye Otomasyon

### Komple DNS Güvenlik ve Performans Scripti

```powershell
<#
.SYNOPSIS
    DNS Security & Performance Hardening Script for Windows Server 2025

.DESCRIPTION
    Bu script, DNS sunucusu için güvenlik ve performans ayarlarını otomatik olarak yapılandırır.
#>

param(
    [string]$ZoneName = "serifselen.local",
    [string[]]$SecondaryServers = @("10.0.2.50", "10.0.2.51"),
    [string]$LogPath = "C:\Windows\System32\dns"
)

Write-Host "🔐 DNS Hardening Script Başlatılıyor..." -ForegroundColor Cyan

# 1. Zone Transfer Güvenliği
Write-Host "`n[1/6] Zone Transfer güvenliği yapılandırılıyor..." -ForegroundColor Yellow
Set-DnsServerPrimaryZone -Name $ZoneName -SecureSecondaries "TransferToSecureServers"
Set-DnsServerPrimaryZone -Name $ZoneName -Notify "NotifyServers" -NotifyServers $SecondaryServers

# 2. Dynamic Update Güvenliği
Write-Host "[2/6] Dynamic Update güvenliği yapılandırılıyor..." -ForegroundColor Yellow
Set-DnsServerPrimaryZone -Name $ZoneName -DynamicUpdate "Secure"

# 3. Cache Optimizasyonu
Write-Host "[3/6] Cache ayarları optimize ediliyor..." -ForegroundColor Yellow
Set-DnsServerCache -MaxCacheSize 512MB -MaxNegativeTtl 300 -PollutionProtection $true

# 4. Recursion Güvenliği
Write-Host "[4/6] Recursive query güvenliği yapılandırılıyor..." -ForegroundColor Yellow
Set-DnsServerRecursion -EnableRecursion $true -SecureResponses $true

# 5. Query Logging (Opsiyonel)
Write-Host "[5/6] Query logging yapılandırılıyor..." -ForegroundColor Yellow
Set-DnsServerDiagnostics `
    -LogFilePath "$LogPath\dns.log" `
    -LogFullPackets $false `
    -LogQueries $true `
    -EnableLogFileRollover $true `
    -MaxLogFiles 7

# 6. DNSSEC Hazırlık Kontrolü
Write-Host "[6/6] DNSSEC hazır mı kontrol ediliyor..." -ForegroundColor Yellow
$Zone = Get-DnsServerZone -Name $ZoneName
if ($Zone.IsDsIntegrated) {
    Write-Host "✓ Zone AD-integrated, DNSSEC için hazır" -ForegroundColor Green
} else {
    Write-Warning "⚠️ Zone AD-integrated değil, DNSSEC için önce dönüştürün"
}

# Rapor Oluşturma
$Report = @"
DNS HARDENING RAPORU
====================
Tarih: $(Get-Date)
Zone: $ZoneName
Secondary Servers: $($SecondaryServers -join ', ')

✅ Zone Transfer: Secure only
✅ Dynamic Update: Secure only
✅ Cache: 512MB, Pollution Protection: ON
✅ Recursion: Secure Responses: ON
✅ Logging: 7-day rollover
"@

$Report | Out-File "$LogPath\hardening_report_$(Get-Date -Format 'yyyyMMdd').txt" -Encoding UTF8
Write-Host "`n✅ Hardening tamamlandı! Rapor: $LogPath\hardening_report_*.txt" -ForegroundColor Green
```

### Otomatik Backup ve Recovery Scripti

```powershell
function Backup-DnsConfiguration {
    param(
        [string]$BackupPath = "D:\DNS_Backup",
        [string]$ZoneName = "serifselen.local"
    )
    
    $Timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $ZoneBackup = "$BackupPath\Zones\$ZoneName`_$Timestamp"
    $ConfigBackup = "$BackupPath\Config\config_$Timestamp.xml"
    
    New-Item -ItemType Directory -Path $ZoneBackup, (Split-Path $ConfigBackup) -Force | Out-Null
    
    # Zone kayıtlarını export etme
    Get-DnsServerResourceRecord -ZoneName $ZoneName | 
        Export-Clixml "$ZoneBackup\records.xml"
    
    # Zone ayarlarını export etme
    Get-DnsServerZone -Name $ZoneName | 
        Select-Object * -ExcludeProperty ReplicationDirectory |
        Export-Clixml "$ZoneBackup\zone_config.xml"
    
    # Server config export
    Export-DnsServerDnsSecPublicKey -ZoneName $ZoneName -FilePath "$ZoneBackup\ds_record.txt" -ErrorAction SilentlyContinue
    
    # PowerShell script backup
    Get-DnsServerQueryResolutionPolicy -ZoneName $ZoneName | 
        Export-Clixml "$ZoneBackup\policies.xml"
    
    Write-Host "✓ Backup tamamlandı: $ZoneBackup" -ForegroundColor Green
    return $ZoneBackup
}

function Restore-DnsConfiguration {
    param(
        [string]$BackupPath,
        [string]$ZoneName = "serifselen.local"
    )
    
    if (-not (Test-Path $BackupPath)) {
        Write-Error "Backup dizini bulunamadı: $BackupPath"
        return
    }
    
    # Zone config restore
    $ZoneConfig = Import-Clixml "$BackupPath\zone_config.xml"
    # (Zone recreation logic here - dikkatli kullanın!)
    
    # Records restore
    $Records = Import-Clixml "$BackupPath\records.xml"
    foreach ($Record in $Records) {
        # Record recreation logic
        Write-Host "⚠️ Record restore manuel müdahale gerektirebilir: $($Record.HostName)" -ForegroundColor Yellow
    }
    
    Write-Host "✓ Restore işlemi tamamlandı (manuel doğrulama önerilir)" -ForegroundColor Cyan
}

# Kullanım örnekleri:
# Backup-DnsConfiguration -ZoneName "serifselen.local"
# Restore-DnsConfiguration -BackupPath "D:\DNS_Backup\Zones\serifselen.local_20240507_143022"
```

* * *

## 🛠️ Sık Karşılaşılan İleri Seviye Sorunlar

Permalink: 🛠️ Sık Karşılaşılan İleri Seviye Sorunlar

### 1. DNSSEC İmza Doğrulama Başarısız

**Sorun:** `Resolve-DnsName -DnssecOk` komutu hata veriyor.

**Çözüm:**
```powershell
# DS record registrar'da doğru eklendi mi kontrol et
# https://dnssec-debugger.verisignlabs.com/ serifselen.local

# Zone serial güncel mi?
Get-DnsServerResourceRecord -ZoneName "serifselen.local" -RRType SOA | 
    Select-Object RecordData.SerialNumber

# Key rollover durumu
Get-DnsServerDnsSecKey -ZoneName "serifselen.local" | 
    Select-Object KeyType, State, PublishDate, ActivateDate

# Gerekirse manual signing
Invoke-DnsServerZoneSign -ZoneName "serifselen.local" -Force
```

### 2. Secondary Zone Transfer Başarısız

**Sorun:** Secondary DNS zone'u güncellenmiyor.

**Çözüm:**
```powershell
# Primary'de zone transfer log'larını kontrol et
Get-WinEvent -LogName "DNS Server" -FilterXPath "*[System[EventID=501 or EventID=503]]" -MaxEvents 20

# Secondary'de manuel transfer dene
Invoke-DnsServerZoneTransfer -Name "serifselen.local" -FullTransfer -ComputerName "10.0.2.50"

# Firewall kurallarını kontrol et (TCP 53)
Test-NetConnection -ComputerName "10.0.2.10" -Port 53 -Protocol TCP

# Notify ayarlarını doğrula
Get-DnsServerPrimaryZone -Name "serifselen.local" | 
    Select-Object Notify, NotifyServers
```

### 3. DNS Policy Çalışmıyor

**Sorun:** Policy tanımlandı ama farklı yanıt dönmüyor.

**Çözüm:**
```powershell
# Policy processing order kontrolü (düşük sayı = yüksek öncelik)
Get-DnsServerQueryResolutionPolicy -ZoneName "serifselen.local" | 
    Select-Object Name, ProcessingOrder, Enabled, ZoneScope

# Policy enabled mı?
Set-DnsServerQueryResolutionPolicy -Name "Internal-View" -Enabled $true

# Client subnet eşleşme testi
# Policy'deki subnet ile istemci IP'si aynı range'de mi?
$PolicySubnet = "192.168.1.0/24"
$ClientIP = "192.168.1.50"
# PowerShell ile subnet match kontrolü (basit)
```

### 4. Query Logging Disk Dolduruyor

**Sorun:** dns.log dosyası çok hızlı büyüyor.

**Çözüm:**
```powershell
# Logging seviyesini düşür
Set-DnsServerDiagnostics `
    -LogFullPackets $false `
    -LogIncomingPackets $true `
    -LogOutgoingPackets $false `
    -MaxLogFiles 3  # Daha az dosya tut

# Eski log'ları temizle
Get-ChildItem "C:\Windows\System32\dns\dns*.log" | 
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) } | 
    Remove-Item -Force

# Log rotation ayarını doğrula
Get-DnsServerDiagnostics | Select-Object EnableLogFileRollover, MaxLogFiles
```

### 5. Split-Horizon Yanlış IP Döndürüyor

**Sorun:** Internal client external IP alıyor.

**Çözüm:**
```powershell
# Policy order kontrolü (internal policy daha düşük order numarası almalı)
Get-DnsServerQueryResolutionPolicy -ZoneName "serifselen.local" | 
    Sort-Object ProcessingOrder | 
    Select-Object Name, ProcessingOrder, ClientSubnet

# Client subnet tanımını doğrula
# "eq,192.168.1.0/24" doğru yazılmış mı?

# Zone scope kayıtlarını karşılaştır
$Internal = Get-DnsServerResourceRecord -ZoneName "serifselen.local" -ZoneScope "Internal-Scope" -Name "www" -RRType A
$External = Get-DnsServerResourceRecord -ZoneName "serifselen.local" -ZoneScope "External-Scope" -Name "www" -RRType A
Write-Host "Internal: $($Internal.RecordData.IPv4Address)" -ForegroundColor Green
Write-Host "External: $($External.RecordData.IPv4Address)" -ForegroundColor Yellow
```

* * *

## 📊 Kaynaklar ve Referanslar

Permalink: 📊 Kaynaklar ve Referanslar

- [Microsoft Docs: DNSSEC on Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/dns/dnssec)
- [Microsoft Docs: DNS Policies](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-policies-scenario)
- [RFC 4033-4035: DNSSEC Specifications](https://datatracker.ietf.org/doc/html/rfc4033)
- [DNS Performance Tuning Guide](https://learn.microsoft.com/en-us/windows-server/networking/dns/performance-tuning)
- [SANS DNS Security Cheat Sheet](https://www.sans.org/security-resources/)

* * *

## 📜 Doküman Bilgileri

Permalink: 📜 Doküman Bilgileri

| Özellik | Değer |
|---------|-------|
| **Yazar** | Serif SELEN |
| **Tarih** | 7 Mayıs 2026 |
| **Versiyon** | 1.0 |
| **Seviye** | İleri Seviye |
| **Platform** | Windows Server 2025 Standard/Datacenter |
| **Zone Adı** | `serifselen.local` |
| **Ön Koşul** | Temel DNS kurulumu tamamlanmış olmalı |

**Kapsanan İleri Konular:**
- ✅ DNSSEC ile dijital imzalama
- ✅ Zone Transfer güvenliği ve secondary replikasyon
- ✅ Conditional Forwarding yapılandırması
- ✅ DNS Policies ile akıllı yönlendirme
- ✅ Query logging ve güvenlik denetimi
- ✅ Performans optimizasyonu ve baseline
- ✅ Split-Horizon (Views) yapılandırması
- ✅ PowerShell otomasyon scriptleri

**Değişiklik Geçmişi:**
- **v1.0**: İlk versiyon - İleri seviye güvenlik, performans ve yüksek erişilebilirlik konuları

> ⚠️ **Uyarı:** Bu doküman eğitim ve test ortamları için hazırlanmıştır. Üretim ortamında değişiklik yapmadan önce mutlaka test ortamında doğrulama yapın ve değişiklik yönetim sürecinizi takip edin.

> 🔐 **Güvenlik Notu:** DNSSEC, zone transfer güvenliği ve logging gibi özellikler, DNS altyapınızın saldırı yüzeyini azaltır. Bu ayarları üretimde mutlaka uygulayın.

> 📧 **İletişim**: mserifselen@gmail.com
>
> 🔗 **GitHub**: https://github.com/serifselen/
>
> 🌐 **Blog**: https://serifselen.com/

* * *
