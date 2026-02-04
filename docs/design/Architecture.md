# Architecture — SAURO İHA Drone Yazılımı

Bu doküman, SAURO İHA drone yazılımının **yüksek seviye mimarisini**, modül sınırlarını ve temel tasarım kararlarını anlatır.

> Amaç: Takım içinde ortak dil oluşturmak, yeni katılanların hızlı adapte olmasını sağlamak ve “profesyonel” seviyede sürdürülebilir bir kod tabanı kurmak.

---

## 1. Tasarım Hedefleri (NFR)

- **Güvenilirlik**: uçuş sırasında hata toleransı ve fail-safe davranışlar
- **Deterministik davranış**: görev akışının net ve izlenebilir olması (FSM)
- **Genişletilebilirlik**: yeni görev adımlarını / algı pipeline’larını kolay eklemek
- **Test edilebilirlik**: simülasyonda tekrarlanabilir test senaryoları
- **Taşınabilirlik**: Windows/Linux geliştirme; hedef cihazda çalışabilirlik
- **Gözlenebilirlik**: log + telemetri ile teşhis kolaylığı

---

## 2. Sistem Bağlamı

### 2.1 Aktörler / Dış Sistemler

- **Flight Controller** (örn. ArduPilot/PX4)  
  - MAVLink üzerinden telemetri + komut
- **Kamera / Görüntü Kaynağı**
  - USB kamera, CSI kamera veya simülasyon görüntüsü
- **SITL / Simülasyon**
  - Uçuş kontrol simülasyonu + dünya/target senaryoları
- **Yer İstasyonu (GCS)**
  - Debug ve operasyonel gözlem (ayrı uygulama olabilir)

---

## 3. Yüksek Seviye Bileşenler

Aşağıdaki bileşenler “katman” mantığında ayrılır:

### 3.1 Mission (FSM / Orchestrator)

**Sorumluluklar**
- Görev adımlarını tanımlar ve sırayla işletir
- Durum geçiş kurallarını uygular
- “Perception” ve “Comms” çıktılarıyla karar verir

**Önerilen örnek durumlar**
- `INIT`
- `ARMING`
- `TAKEOFF`
- `SEARCH`
- `APPROACH_TARGET`
- `TRACK`
- `LAND`
- `FAILSAFE`

> Durum geçişleri tek noktadan yönetilmelidir. Dağınık “if” zincirleri ileride bakım maliyetini artırır.

### 3.2 Perception (Vision)

**Sorumluluklar**
- Görüntü acquisition (kamera pipeline)
- Ön işleme: blur, threshold, HSV mask, morphology
- Tespit: contour / blob / marker
- Takip: basit centroid takip veya tracker
- Çıktı: (target_found, bbox, center, confidence) gibi bir data modeli

**İlk etapta hedef**
- Basit, hızlı ve açıklanabilir yöntemler (OpenCV)
- Kademeli olarak daha ileri yöntemlere açık yapı

### 3.3 Control / Navigation

**Sorumluluklar**
- Mission kararlarını “uçuş komutlarına” çevirir
- Hız/irtifa/heading gibi setpoint’ler üretir
- Güvenli limit kontrolleri ve saturasyon

Not: Kontrol katmanı mümkünse “Flight Controller”ın yeteneklerini kullanır, üst seviye yazılım sadece hedef/komut seviyesinde kalır.

### 3.4 Comms (MAVLink)

**Sorumluluklar**
- Bağlantı yönetimi (connect/reconnect)
- Telemetri alma (pose, speed, battery, mode)
- Komut gönderme (arm, takeoff, set_mode, set_position_target)
- Heartbeat ve timeout yönetimi

### 3.5 Config & Tooling

- Konfigürasyon dosyaları (yaml/json)
- Script’ler (sim/run/lint)
- Log çıktıları, debug araçları

---

## 4. Modül Sınırları ve Veri Akışı

Önerilen veri akışı:

1) `Perception` görüntüden **Algı Çıktısı** üretir  
2) `Comms` telemetriden **Drone Durumu** üretir  
3) `Mission` bu iki girdiyi değerlendirir ve **Niyet/Komut** üretir  
4) `Control` niyeti **setpoint/komut** formatına çevirir  
5) `Comms` komutu Flight Controller’a iletir

Bu yapı, test için “mock Perception” veya “mock Comms” ile kolay simülasyon sağlar.

---

## 5. Konfigürasyon Stratejisi

- Varsayılan: `config/default.yaml`
- SITL: `config/sitl.yaml`
- Donanım: `config/hardware.yaml`

**Prensipler**
- Uçuş kritik parametreler (min/max altitude, timeout) config’te olmalı
- Vision eşikleri config’te olmalı (farklı ışık koşulları için)

---

## 6. Hata Yönetimi ve Fail-safe

Fail-safe örnekleri:

- MAVLink heartbeat kesilirse → `FAILSAFE` durumuna geç
- Kamera kapanırsa → mission durdur, güvenli moda geç
- Hedef güven aralığı düşükse → yaklaşma yerine “yeniden arama” yap
- Batarya kritik eşik → görevi sonlandır, iniş planı uygula

---

## 7. Logging & Observability

- `logging` standart kütüphanesi
- Her bileşen kendi logger’ını kullanır: `mission`, `vision`, `comms`, `control`
- Format: timestamp, level, module, message
- Kritik olaylar ayrıca telemetri/console’a raporlanır

---

## 8. Yol Haritası

**MVP (ilk çalışan sistem)**
- MAVLink bağlantı + temel telemetri
- Basit FSM: init → takeoff → search → land
- OpenCV ile temel tespit (ör. renk/şekil)
- Simülasyonda uçtan uca demo

**Sonraki adımlar**
- Robust reconnect + timeout yönetimi
- Hedef takip/merkezleme kontrolü
- Test otomasyonu ve CI
- Konfigürasyon şablonları (example dosyaları)

---

## 9. Tasarım Kararları (Kısa Gerekçeler)

- **FSM kullanımı**: görev adımları deterministik ve denetlenebilir olur.
- **Katmanlı mimari**: vision/mission/comms ayrımı, ekiple paralel çalışma sağlar.
- **Config-first**: sahada hızlı ayar değiştirme, yeniden derleme gerektirmez.
- **Simülasyon önceliği**: güvenli ve tekrarlanabilir test.

---

## 10. İlgili Dokümanlar

- `README.md` (kullanım / kurulum)
- `docs/design/` (tasarım & mimari dokümanları)
- `config/` (ortam profilleri)
