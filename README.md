# 📊 Powerpipe + Steampipe AWS Dashboards – README

Bu doküman, **Steampipe + Powerpipe** kullanarak **AWS dashboard’larını UI üzerinden çalıştırmak, rapor almak ve servisi kalıcı hale getirmek** için hazırlanmıştır.

---

## 1️⃣ Temel Gereksinimler (EN ÖNEMLİ KISIM)

### ✅ Kullanıcı
Her şey **aynı kullanıcı** ile çalıştırılmalıdır (örnek: `ubuntu`).

```bash
whoami
# ubuntu
```

---

### ✅ AWS Credentials (ZORUNLU)
Steampipe, AWS API’lerine erişmek için **shared credentials** ister.

#### Dosyalar:
```bash
~/.aws/credentials
~/.aws/config
```

#### Örnek içerik:

**~/.aws/credentials**
```ini
[default]
aws_access_key_id=AKIAxxxxxxxxxxxx
aws_secret_access_key=xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**~/.aws/config**
```ini
[default]
region=eu-central-1
output=json
```

İzinler:
```bash
chmod 700 ~/.aws
chmod 600 ~/.aws/*
```

---

### ✅ Steampipe AWS Connection
```bash
nano ~/.steampipe/config/aws.spc
```

```hcl
connection "aws_account_a" {
  plugin  = "aws"
  profile = "default"
  regions = ["eu-central-1"]
}
```

---

### ✅ Steampipe Servisini Başlat
```bash
steampipe plugin install aws

steampipe service stop
rm -rf ~/.steampipe/data
steampipe service start --database-listen network --database-port 9193
```

#### Test:
```bash
steampipe query "select * from aws_account_a.aws_account limit 1;"
```

---

## 2️⃣ Powerpipe Mod Kurulumu

### AWS Insights Modu
```bash
cd ~/steampipe-mod-aws-insights
powerpipe mod install
```

---

## 3️⃣ Powerpipe Dashboard Server (UI)

### Manuel çalıştırma
```bash
powerpipe server \
  --mod-location /home/ubuntu/steampipe-mod-aws-insights/.powerpipe/mods/github.com/turbot/steampipe-mod-aws-insights@v1.2.0 \
  --listen network \
  --port 9194
```

Tarayıcı:
```
http://SERVER_IP:9194
```

---

## 4️⃣ Powerpipe Server’ı Kalıcı Hale Getirme (SYSTEMD)

### Servis dosyası
```bash
sudo nano /etc/systemd/system/powerpipe.service
```

```ini
[Unit]
Description=Powerpipe Dashboard Server
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/steampipe-mod-aws-insights
ExecStart=/usr/bin/powerpipe server --mod-location /home/ubuntu/steampipe-mod-aws-insights/.powerpipe/mods/github.com/turbot/steampipe-mod-aws-insights@v1.2.0 --listen network --port 9194
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> ⚠️ `which powerpipe` ile path kontrol et (`/usr/bin/powerpipe` farklı olabilir)

### Aktifleştir:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now powerpipe
```

### Kontrol:
```bash
sudo systemctl status powerpipe
ss -lntp | grep 9194
```

---

## 5️⃣ Dashboard & Rapor Alma

### Dashboard listesi
```bash
powerpipe dashboard list
```

### Snapshot (rapor) alma
```bash
powerpipe dashboard run aws_insights.dashboard.aws_account_report \
  --export snapshot \
  --snapshot-location aws_account_report.sps
```

### Değişkenli dashboard
```bash
powerpipe dashboard run aws_insights.dashboard.ec2_instance_dashboard \
  --var region=eu-central-1 \
  --export snapshot \
  --snapshot-location ec2_eu.sps
```

---

## 6️⃣ Günlük Yönetim Komutları

```bash
sudo systemctl restart powerpipe
sudo systemctl stop powerpipe
sudo systemctl start powerpipe
```

Log izleme:
```bash
sudo journalctl -u powerpipe -f
```

---

## 7️⃣ SORUN OLURSA (TROUBLESHOOTING – EN ALT)

### ❌ `failed to get shared config profile`
➡ AWS credentials yok / okunamıyor

```bash
ls -la ~/.aws
cat ~/.aws/credentials
cat ~/.aws/config
```

Sahiplik:
```bash
sudo chown -R ubuntu:ubuntu ~/.aws ~/.steampipe
```

---

### ❌ `relation aws_* does not exist`
➡ Steampipe AWS plugin schema yüklenmemiş

```bash
steampipe plugin install aws
steampipe service stop
rm -rf ~/.steampipe/data
steampipe service start --database-listen network --database-port 9193
```

---

### ❌ `Mod defines more than one resource`
➡ Aynı mod **iki kere** yükleniyor

**Çözüm:**  
`powerpipe server` mutlaka **.powerpipe/mods/...** içinden çalıştırılmalı:

```bash
--mod-location /home/ubuntu/steampipe-mod-aws-insights/.powerpipe/mods/github.com/turbot/steampipe-mod-aws-insights@v1.2.0
```

---

### ❌ UI açılıyor ama boş
➡ Steampipe çalışmıyor veya AWS erişimi yok

```bash
steampipe query "select * from aws_account_a.aws_account limit 1;"
```

---

## ✅ ÇALIŞIYOR DEME KRİTERLERİ
- `ss -lntp | grep 9193` → Steampipe açık
- `ss -lntp | grep 9194` → Powerpipe UI açık
- Dashboard UI veri gösteriyor
- Snapshot export çalışıyor

---

## 📌 Notlar
- AWS CLI zorunlu değildir
- Powerpipe yalnızca Steampipe üzerinden veri çeker
- UI = görselleştirme, Steampipe = veri motoru
