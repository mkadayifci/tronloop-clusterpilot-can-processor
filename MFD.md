# tronloop-clusterpilot-engine — Proje ve firmware bağlamı

Son inceleme: 2026-09-18.

Bu dosya, sonraki geliştirmelerde kullanılacak proje bağlamını, mevcut kodun durumunu ve firmware ile uzlaştırılması gereken protokol ayrıntılarını tutar. Kaynaklar: bu depodaki C# dosyaları ve kullanıcının firmware agentından aktardığı açıklama. Firmware kaynakları veya gerçek CAN trafiği bu incelemede doğrulanmadı. Aşağıdaki öneriler henüz uygulanmış özellikler değildir.

## Amaç

STM32L476 tabanlı Tronloop batarya test cihazından CAN üzerinden gelen veriyi almak, ISO-TP payload'larını ayrıştırmak ve ilgili işleyicilere dispatch etmek. Bilgisayar tarafı, üniversite araştırmasında uzun süreli şarj/deşarj deneyleri, batarya yaşlanma analizi ve bilimsel yayın için zaman damgalı, deney koşullarıyla ilişkilendirilmiş, izlenebilir veri toplamalıdır.

Ham verinin korunması, bağlantı kesintilerinin ve veri boşluklarının görünür olması temel gereksinimlerdir. Bir komutun alınması/gönderilmesi ile cihazda fiziksel olarak uygulanması ayrı durumlardır.

## Firmware bağlamı — kullanıcıdan aktarılan

- MCU: STM32L476.
- BQ25756 şarj/reverse mode kontrolünü, GPIO bus switch güç yolu kontrolünü yürütür.
- INA226 ve BQ34Z100 üzerinden ölçüm alınır.
- `g_tl_context`, ölçümleri ve cihaz durumunu tutar.
- Birimler: mV, mA, ms ve onda bir °C. Pozitif akım şarj, negatif akım deşarjdır.
- Senaryo oynatıcı 32 adımlık bir dizi üzerinden şarj, deşarj, zaman koşulu ve stop işlemlerini yürütür. Diğer koşullar ve bazı eylemler geliştirme aşamasındadır.

### Ölçüm ve durum sınırlamaları

- Batarya akımı ve SOC güncellenir; gerilim/sıcaklık güncellemeleri henüz tamamlanmamıştır.
- SOC mevcut hızlı telemetri paketinde bulunmaz.
- Hızlı telemetride `state` sabit `1` gönderilir; gerçek cihaz durumunun kanıtı olarak kullanılmamalıdır.
- Heartbeat'in kullandığı context durumu ile gerçek senaryo oynatıcı durumu henüz eşitlenmemiştir.
- Sıfır değerler otomatik olarak geçerli ölçüm kabul edilmemelidir. Buna karşılık tüm sıfırları otomatik geçersiz saymak da doğru değildir; ham değer korunmalı, geçerlilik ayrıca belirtilmelidir.

## Haberleşme sözleşmesi — doğrulanacak mevcut biçim

CAN hızı **500 kbps**, bildirilen cihaz ID'si **`0x100`**. Telemetri ve komutlar ISO-TP kullanır; firmware RX/TX tamponları 1024 bayttır. CAN çerçeveleri uygulama payload'ı ayrıştırılmadan önce ISO-TP ile birleştirilmelidir.

Bu depoda Linux `CAN_ISOTP` soketi kullanıldığı için birleştirme kernel tarafında yapılır; `read` ile alınan veri uygulama payload'ıdır. İkinci bir ISO-TP birleştirme katmanı eklenmemelidir.

Yerel ayarlar PC bakış açısından RX=`0x100`, TX=`0x101` kullanır. Firmware açıklamasındaki cihaz ID'si tek başına iki yönün adreslemesini doğrulamaz. Cihaz TX/PC RX, cihaz RX/PC TX ve flow-control adresleri firmware agentıyla netleştirilmelidir. Kod CAN arayüzünün bitrate ayarını yapmaz.

### Cihaz → PC: telemetri

Payload'lar doğrudan packed C yapılarından üretilir ve mevcut STM32 üzerinde çok baytlı alanlar little-endian gönderilir. **Bu mesajlarda 4 baytlık TLP komut başlığı yoktur.**

`packed`, enum alanını tek bayta indirmez. Aşağıdaki ofsetler her iki enum türünün de **4 bayt** olduğu varsayımına dayanır; firmware derleyici ayarları ve `sizeof`/alan ofsetleriyle doğrulanmalıdır. Beklenen boyutlar fast telemetry için **11**, heartbeat için **8 bayt**tır.

#### Fast telemetry — hedef periyot 100 ms

| Ofset | Boyut | Alan | Yorum |
| --- | --- | --- | --- |
| 0 | 4 | `payload_type` enum | `0x01`; beklenen baytlar `01 00 00 00` |
| 4 | 2 | `uint16_t battery_voltage_mv` | mV |
| 6 | 2 | `int16_t battery_current_ma` | mA; işaret korunmalı |
| 8 | 2 | `int16_t battery_temp_decic` | °C = ham değer / 10.0 |
| 10 | 1 | `uint8_t state` | Mevcut firmware'de sabit `1` |

#### Heartbeat — hedef periyot 500 ms

| Ofset | Boyut | Alan | Yorum |
| --- | --- | --- | --- |
| 0 | 4 | `payload_type` enum | `0x02`; beklenen baytlar `02 00 00 00` |
| 4 | 4 | `ScenarioPlayerState_t scenario_player_state` enum | Sayısal enum eşlemesi henüz paylaşılmadı |

100/500 ms değerleri hedef gönderim periyotlarıdır; her paketin tam bu aralıkla geleceği varsayılmamalıdır. Bu biçimde cihaz zaman damgası, ölçüm sıra numarası veya telemetri sürüm alanı bildirilmemiştir.

### PC → cihaz: TLP komutları

Başlık 4 bayttır: `command`, `version=0x01`, `sequence`, `flags` (her biri 1 bayt). Ardından komuta özgü payload gelir.

| Komut | Kod | Başlık sonrası payload |
| --- | --- | --- |
| Ping | `0x70` | Bildirilen ek alan yok |
| Charger enable | `0x21` | Bildirilen ek alan yok |
| Charger disable | `0x22` | Bildirilen ek alan yok |
| Script start | `0x10` | 1 bayt script ID |
| Script stop | `0x11` | Bildirilen ek alan yok |
| Charge limits | `0x23` | Little-endian `uint16` mV + `int16` mA |

Bu komutların çoğu firmware'de henüz yalnızca log üretir. Fiziksel işlem tamamlandı varsayılmamalıdır. `sequence` yönetimi, `flags` anlamları, yanıt/ACK biçimi, timeout ve tekrar deneme davranışı açıklamada tanımlı değildir; uydurulmamalıdır.

## Bu deponun mevcut durumu — koddan incelenen

### Çalışma yapısı

- `Tronloop.ClusterPilot.Engine.csproj`: .NET 10 Worker; `Microsoft.Extensions.Hosting` ve `Microsoft.Extensions.Hosting.Systemd` 10.0.9, MQTTnet 5.2.0.1603 referansları.
- `Program.cs`: generic host oluşturur ve `Worker` servis kaydını yapar. Systemd paketi mevcut olsa da burada özel systemd entegrasyon çağrısı yoktur.
- `CanIsoTpListener.cs`: `libc` P/Invoke üzerinden Linux SocketCAN ISO-TP soketi açar, okur ve yazar. Mevcut taşıma kodu Linux'a yöneliktir.
- `Worker.cs`: CAN dinleyicilerini başlatır, dummy CAN gönderimini ve MQTT bağlantısını yürütür.
- `appsettings.json`: `can0`, RX=`0x100`, TX=`0x101`; virgülle ayrılmış birden fazla RX/TX çifti desteklenir. Çift sayıları eşit değilse CAN dinleyicileri başlatılmaz.

### CAN alımı ve ayrıştırma

- Dinleme döngüsü `Task.Run` içinde çalışır; okuma tamponu 4096 bayttır. Bu boyut firmware'in 1024 baytlık sınırını genişletmez.
- Sokette 1 saniyelik receive timeout ayarlanır; ayarlanamazsa iptal kontrolü yapamayan bir okuma döngüsü başlatmamak için soket açılışı başarısız olur. Bazı okuma hatalarında soket kapatılıp yeniden açılır; belirli hata yollarında 2 saniye beklenir.
- `FastTelemetryPayload`, yalnızca gerilim, akım, sıcaklık ve state alanlarını içerir. `Pack=1` ile **7 bayttır** ve `payload_type` alanı yoktur.
- Parser sadece `bytesRead == Marshal.SizeOf<FastTelemetryPayload>()` koşuluyla çalışır. Aktarılan firmware biçimi 11 baytsa paket parse edilmeden geçilir.
- Yalnız uzunluğa bakılır; mesaj tipi doğrulanmaz. `MemoryMarshal.Read` açık bir little-endian protokol çözümlemesi yapmaz, yerel bellek düzenine dayanır.
- Heartbeat parser'ı, mesaj tipine göre dispatcher, event/channel çıkışı ve CAN telemetrisini MQTT'ye aktaran yol yoktur.
- RX hex metni üretilir ancak onu yazan satırlar yorumdadır. Ham payload kalıcı olarak saklanmaz; tanınmayan uzunluklar için kayıt yoktur.
- `Send` ham byte dizisini yazar; TLP komut oluşturma ve firmware boyut sınırı doğrulaması yoktur.

### Komutlar ve MQTT

- Her açılan CAN dinleyicisi için saniyede bir **12 bayt dummy payload** gönderilir: ilk 4 bayt sayaç, kalan baytlar sıfırdır. Bu TLP komut kodlayıcısı değildir ve gerçek cihaz kullanımından önce kaldırılmalı veya açık bir test seçeneğine bağlanmalıdır.
- Node ID kodda `A0` olarak sabittir.
- MQTT bağlantısı kodda `mqtt.tronloop-lab.com:1883` ve client ID `orchestrator-A0` kullanır. `appsettings.json` içindeki `Mqtt` bölümü mevcut bağlantı kurulurken okunmaz.
- Abonelikler: `tronloop/node/A0/cmd`, `tronloop/broadcast/cmd`.
- Yayınlar: `tronloop/orchestrator/A0/ack`, `/status`, `/heartbeat`. Orchestrator heartbeat'i 5 saniyede bir üretilir; cihazın CAN heartbeat'i değildir ve cihazın hayatta olduğunu kanıtlamaz.
- `start_charge`, `stop_charge`, `set_current`, `ping` MQTT komutları CAN'a gönderilmez. Yerel işleyici başarı ACK'si üretir; “Charge started” gibi yanıtlar fiziksel sonuç kanıtı değildir.
- `NodeCommand` yalnız `Id`, `Type`, `Value` taşır; çoklu cihaz için hedef cihaz alanı yoktur. `set_current` komutunu iki alan gerektiren charge-limits komutuna çevirmek için gerilim sınırının kaynağı/birimi ayrıca tanımlanmalıdır.
- Deney kaydı, kalıcı veri deposu, MQTT kesintisinde veri tamponlama ve MQTT reconnect döngüsü uygulanmamıştır.

### Yaşam döngüsüyle ilgili takip işleri

- İlk `Open()` çağrısı `Worker` içinde başarısız olursa o cihazın dinleme döngüsü başlamaz; döngü içindeki reconnect bu ilk açılış hatasını kapsamaz.
- `ObjectDisposedException` düzeltmesi: CAN görevleri host token'ına bağlı ayrı bir cancellation kaynağı kullanır. MQTT hata yolu dahil `finally` önce bu kaynağı iptal eder, tüm CAN görevlerini bekler ve ardından dinleyicileri dispose eder. Böylece host henüz durdurulmamış olsa bile dummy gönderimi dispose edilmiş dinleyiciyi kullanmaya devam etmez. MQTT reconnect hâlâ uygulanmamıştır.
- Soket açma/kapama kilitli olsa da gerçek `read`/`write` çağrıları kilit dışındadır; eşzamanlı gönderim, reconnect ve dispose davranışı gözden geçirilmelidir.

## Hedef veri akışı ve geliştirme sırası — öneri

Hedef akış: CAN/ISO-TP alımı → ham payload ve alım metadatasının kaydı → tip/uzunluk doğrulaması → açık little-endian ayrıştırma → tipli mesaj dispatch → deney kaydı/MQTT tüketicileri.

1. Firmware ile enum boyutlarını, alan ofsetlerini ve iki yönlü CAN adreslemesini doğrula; her mesaj türü için gerçek örnek hex payload al.
2. Otomatik dummy gönderimini kaldır veya varsayılan kapalı test seçeneğine al. Uygulama ACK'sini cihazda uygulanmış işlem gibi sunmayı düzelt.
3. Taşıma ve parser sorumluluklarını ayır. Doğrulanmış enum genişliğiyle önce türü, sonra türe özgü tam uzunluğu kontrol et. `BinaryPrimitives` gibi açık little-endian okuyucularla signed/unsigned alanları çöz.
4. Fast telemetry ve heartbeat için ayrı tipli mesajlar ve işleyiciler ekle. Kaynak arayüz/RX/TX/cihaz kimliğini dispatch boyunca taşı. Bilinmeyen tür veya bozuk paket dinleme döngüsünü durdurmasın; ham veri ve parse hatası kaydedilsin.
5. Deney oturumu, kalıcı kayıt, kalite bilgisi, kesinti/boşluk takibi ve tüketici yavaşlaması davranışını uygula. MQTT gönderim gecikmesi CAN okuma ve kayıt yolunu kontrolsüzce bloke etmesin.
6. Firmware ile netleşen TLP komut kodlayıcısını ve komut yaşam döngüsünü ekle. Alındı, gönderildi ve cihaz tarafından doğrulandı durumlarını ayır; doğrulama protokolü yoksa fiziksel başarı bildirme.
7. MQTT ayarlarını konfigürasyondan oku; CAN/MQTT reconnect ve kapanış davranışını düzenle.

### Araştırma kaydı için tutulacak bilgiler — öneri

- PC alım anında UTC zaman damgası; aralık/gecikme analizi için monotonik saat değeri. PC alım zamanı cihaz ölçüm zamanı olarak etiketlenmemelidir.
- Deney/oturum kimliği, batarya/numune kimliği, senaryo ve deney koşulları; biliniyorsa firmware sürümü ve parser/protokol biçimi sürümü.
- Kaynak node/cihaz, CAN arayüzü, RX/TX ID'leri, ham ISO-TP payload ve uzunluğu.
- Ham integer ölçümler ve birimleri; parse sonucu, tanınmayan enum değerleri ve alan bazında geçerlilik/güncellik bilgisi.
- Bağlantı değişimleri, yeniden bağlanma, son telemetri/heartbeat zamanı ve beklenen periyoda göre gözlenen veri boşlukları.

Payload'da cihaz zaman damgası ve sıra numarası bulunmadığından kesin ölçüm anı veya kesin kayıp paket sayısı çıkarılamaz. Periyoda göre yapılan boşluk tahminleri tahmin olarak etiketlenmelidir. Ham ISO-TP payload saklamak ile tek tek CAN çerçevelerini saklamak farklıdır; mevcut soket API'si uygulamaya birleştirilmiş payload verir.

## Firmware agentıyla netleştirilecekler

- Gerçek `sizeof(payload_type enum)`, `sizeof(ScenarioPlayerState_t)`, iki payload yapısının boyutları ve alan ofsetleri; enum genişliğini değiştiren derleyici seçenekleri.
- PC RX/TX ve firmware RX/TX CAN ID eşlemesi, standard/extended ID kullanımı ve ISO-TP flow-control ayarları.
- `ScenarioPlayerState_t` sayısal değerleri, `state` anlamı ve context/oynatıcı durum eşitlemesinin tamamlanma durumu.
- Gerilim/sıcaklık ölçümlerinin hazır olma durumu ve alan geçerliliğinin nasıl bildirileceği.
- Gerçekte uygulanmış komutlar, desteklenen script ID'leri, `sequence`/`flags`, cihaz yanıtları ve komut tekrarının etkileri.
- Telemetriye sürüm, cihaz zaman damgası, sıra numarası ve kalite bayrakları eklenip eklenmeyeceği. Bunlar mevcut protokolün parçası sayılmamalıdır.

## Doğrulama kapsamı

İlk bağlam incelemesi kaynak kod ve kullanıcının firmware açıklamasıyla yapıldı. Sonraki `ObjectDisposedException` düzeltmesinde CAN görevlerinin iptal/dispose sıralaması ve receive timeout hata davranışı güncellendi. Donanım testi veya canlı MQTT/CAN bağlantısı çalıştırılmadı.

Parser uygulanırken doğrulanmış örnek payload'lar, negatif akım/sıcaklık, hatalı uzunluk ve bilinmeyen mesaj türü test edilmelidir. Ardından Linux CAN ortamında ISO-TP alımı, gerçek cihazla adresleme, kesinti/reconnect ve kayıt bütünlüğü doğrulanmalıdır.
