# ESP32 Asenkron Web Sunucusu ile LED Kontrolü

Bu proje, ESP32 mikrodenetleyicisine bağlı fiziksel bir LED'i; HTML, CSS ve yerel JavaScript (Fetch API) kullanarak oluşturulan asenkron bir web sunucusu üzerinden tarayıcıyı yenilemeden kontrol etmeyi gösterir.

## 🚀 Özellikler
* **Asenkron Mimari:** Ana döngüyü (`loop`) bloke etmeden arka planda gelen istekleri işlemek için `ESPAsyncWebServer` kütüphanesi kullanılmıştır.
* **Sayfa Yenilemesiz Akış:** JavaScript `Fetch API` kullanılarak arka planda sessizce HTTP GET istekleri gönderilir. Bu sayede modern ve akıcı bir kullanıcı deneyimi sunulur.
* **Duyarlı (Responsive) Arayüz:** Hem masaüstü hem de mobil tarayıcılarla tam uyumlu, gözü yormayan karanlık tema (Dark Mode) bir kontrol paneline sahiptir.

## 🛠️ Donanım Gereksinimleri
* ESP32 Geliştirme Kartı (Örn: DOIT NodeMCU ESP32)
* Harici bir LED ve uygun direnç
* Type-C-USB Kablosu

## 📦 Gerekli Kütüphaneler
Projenin derlenebilmesi için Arduino IDE'nize şu iki kütüphaneyi eklemeniz gerekir:
1. **AsyncTCP:** Arduino Kütüphane Yöneticisi (Library Manager) üzerinden doğrudan aratılarak yüklenebilir.
2. **ESPAsyncWebServer:** [GitHub](https://github.com/me-no-dev/ESPAsyncWebServer) üzerinden ZIP olarak indirilip `Taslak > Kütüphane Ekle > .ZIP Kütüphanesi Ekle` adımlarıyla yüklenmelidir.

---

## 💻 Kaynak Kodları (Source Code)

Arduino IDE üzerinde yeni bir taslak açarak aşağıdaki kodları yapıştırabilirsiniz. Kodun en üstündeki Wi-Fi bilgilerini kendi ağınıza göre düzenlemeyi unutmayın!

```cpp
#include <WiFi.h>
#include <ESPAsyncWebServer.h>

// Wi-Fi Ağ Bilgileri
const char* ssid = "AG_ADINIZ";        
const char* password = "AG_SIFRENIZ"; 

const int ledPin = 15; // Harici LED (GPIO 15)
bool ledState = LOW;

AsyncWebServer server(80);

// Web Sayfası İçeriği (RAM'i şişirmemek için PROGMEM ile Flaş Hafızada saklanır)
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ESP32 LED Kontrol Paneli</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            text-align: center;
            background-color: #1e1e24;
            color: #fff;
            margin-top: 80px;
        }
        h1 { color: #00adb5; }
        .btn {
            padding: 20px 40px;
            font-size: 22px;
            font-weight: bold;
            cursor: pointer;
            border: none;
            border-radius: 50px;
            color: white;
            box-shadow: 0 4px 15px rgba(0,0,0,0.3);
            transition: all 0.3s ease;
        }
        .btn-off { background-color: #ff2e63; }
        .btn-off:hover { background-color: #e21f4f; transform: scale(1.05); }
        .btn-on { background-color: #21bf73; }
        .btn-on:hover { background-color: #1ca362; transform: scale(1.05); }
    </style>
</head>
<body>

    <h1>ESP32 Akıllı Ev Paneli</h1>
    <p>Aşağıdaki butona basarak ESP32 üzerindeki LED'i anlık olarak kontrol edebilirsiniz.</p>
    <br><br>
    
    <button id="ledBtn" class="btn btn-off" onclick="toggleLED()">LED'i AÇ</button>

    <script>
        let ledDurumu = false;

        function toggleLED() {
            ledDurumu = !ledDurumu;
            const btn = document.getElementById('ledBtn');
            let url = ledDurumu ? "/led/on" : "/led/off";
            
            // Arka planda asenkron HTTP isteği gönderme
            fetch(url)
                .then(response => {
                    if(response.ok) {
                        if (ledDurumu) {
                            btn.innerText = "LED'i KAPAT";
                            btn.className = "btn btn-on";
                        } else {
                            btn.innerText = "LED'i AÇ";
                            btn.className = "btn btn-off";
                        }
                    }
                })
                .catch(err => {
                    console.error("Bağlantı hatası:", err);
                    alert("ESP32 ile bağlantı kurulamadı!");
                    ledDurumu = !ledDurumu; 
                });
        }
    </script>
</body>
</html>
)rawliteral";

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, ledState);

  // Wi-Fi Bağlantısı
  WiFi.begin(ssid, password);
  Serial.print("Wi-Fi ağına bağlanılıyor");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("\nBağlantı Başarılı!");
  Serial.print("Tarayıcıya yazacağınız IP Adresi: ");
  Serial.println(WiFi.localIP()); 

  // --- Web Sunucu Rotaları (Routes) ---
  
  // 1. Ana Sayfa İsteği (HTML içeriğini gönderir)
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/html", index_html);
  });

  // 2. LED Yakma İsteği
  server.on("/led/on", HTTP_GET, [](AsyncWebServerRequest *request){
    digitalWrite(ledPin, HIGH);
    request->send(200, "text/plain", "OK");
  });

  // 3. LED Söndürme İsteği
  server.on("/led/off", HTTP_GET, [](AsyncWebServerRequest *request){
    digitalWrite(ledPin, LOW);
    request->send(200, "text/plain", "OK");
  });

  // Sunucuyu Başlat
  server.begin();
}

void loop() {
  // ESPAsyncWebServer arka planda çalıştığı için loop içi boş kalabilir.
}
