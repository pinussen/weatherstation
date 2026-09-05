# Väderstation för ESP32 och ESPHome

## Komplett program för ditt ESP32-C6

**[Öppna weatherstation.yaml](weatherstation.yaml)** · **[Visa hela filen som text (Raw)](https://raw.githubusercontent.com/pinussen/weatherstation/main/weatherstation.yaml)**

Detta är hela ESPHome-programmet i en enda fil. Alla sensorer, WiFi, Home Assistant-API,
OTA, loggar och kalibrering finns i filen. Inga `packages`, `!include` eller separata
`secrets.yaml` behövs. ESPHome bygger programmet till firmware åt dig.

### Installera från Home Assistant

1. Öppna **ESPHome Device Builder** i Home Assistant (ESPHome 2026.8.2 eller senare).
   Saknas den, följ [installationsguiden](https://esphome.io/install/getting-started/).
2. Skapa en ny enhet för ESP32-C6 och välj **Edit**.
3. Kopiera **hela** [weatherstation.yaml](https://raw.githubusercontent.com/pinussen/weatherstation/main/weatherstation.yaml)
   och ersätt allt innehåll i enhetens editor.
4. Ändra följande under `substitutions` högst upp i filen:
   - `wifi_ssid` och `wifi_password`: ditt 2,4 GHz WiFi.
   - `api_key`: din egen krypteringsnyckel från Device Builders enhetsguide.
   - `ota_password` och `ap_password`: egna lösenord (AP minst åtta tecken).
   - Behåll enhetens befintliga `name` om du ersätter en redan installerad ESPHome-enhet.
5. Klicka **Save → Validate → Install**. Första installationen görs via USB:
   anslut kortet till HA-maskinen och välj serieporten, eller välj
   **Manual download → Factory format** och installera med [ESPHome Web](https://web.esphome.io/)
   i Chrome/Edge på datorn med USB-kabeln. Om kortet inte hittas, håll BOOT medan du
   trycker RESET och försök igen.
6. Lägg till den upptäckta ESPHome-enheten under **Inställningar → Enheter och tjänster**.
   Ange samma API-nyckel. Vid utebliven upptäckt kan du lägga till den med IP-adressen.
7. Därefter används **Install → Wirelessly**, **Logs** och **Edit** i Device Builder.

Exempelnyckeln och lösenorden i filen är offentliga byggexempel; ersätt dem före
installation. API-nyckeln ska vara 32 slumpbyte i base64. Den kan även genereras med:

```sh
python3 -c 'import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())'
```

Du kan senare flytta uppgifterna till ESPHomes `secrets.yaml` om du vill.
Dela inte din redigerade fil med lösenord i. Kör du Home Assistant Container behövs
ESPHome Device Builder separat. MQTT-broker behövs inte.

## Kort och inkoppling

Alla nedanstående filer är kompletta, fristående program; välj **en** för ditt chip.

| Fil | Chip | SDA | SCL | Vindpuls | Regnpuls |
|---|---|---|---|---|---|
| **[weatherstation.yaml](weatherstation.yaml)** | **ESP32-C6 DevKit** | **GPIO6** | **GPIO7** | **GPIO10** | **GPIO11** |
| [ESP32/WROOM](esphome/weatherstation-esp32.yaml) | ESP32 DevKit | GPIO21 | GPIO22 | GPIO25 | GPIO26 |
| [C3](esphome/weatherstation-c3.yaml) | ESP32-C3 | GPIO4 | GPIO5 | GPIO6 | GPIO7 |
| [S3](esphome/weatherstation-s3.yaml) | ESP32-S3 | GPIO8 | GPIO9 | GPIO4 | GPIO5 |
| [S2](esphome/weatherstation-s2.yaml) | ESP32-S2 | GPIO8 | GPIO9 | GPIO4 | GPIO5 |

Profilerna använder ESP-IDF, 4 MB flash och inget PSRAM. Kort med större flash kan
också använda layouten. Kontrollera att pinnarna är lediga och utdragna på just ditt
kort; ändra dem i `substitutions` vid behov. C6 Super Mini och XIAO kan ha andra
pinnar än [C6 DevKitC](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32c6/esp32-c6-devkitc-1/user_guide.html).

Alla I²C-sensorer ansluts parallellt till SDA/SCL och gemensam jord, med pull-ups till
**3,3 V**. Pulssignalerna använder intern pull-up och aktiv låg signal. ESP32-ingångar
ska inte matas med 5 V. A3144 kan behöva 5 V matning: använd dess open-collector-utgång
med pull-up till 3,3 V, eller nivåomvandling om modulens utgång dras upp till 5 V.

## Sensorer och inställningar

| Sensor | Funktion | Standardadress |
|---|---|---|
| AHT20 | Temperatur och fukt | `0x38` |
| BMP280 | Stations- och havsnivåtryck | `0x76` |
| ENS160 | TVOC, uppskattad eCO2 och AQI | `0x53` |
| BH1750 | Ljusstyrka | `0x23` |
| AS5600 | Vindriktning och nordkalibrering | `0x36` |
| Pulsgivare | Vindhastighet, vindby, regnintensitet och regntotal | GPIO |

Alla sensorer är aktiverade i programmet. Saknade I²C-sensorer ger loggfel och
saknade värden; resten av enheten kan användas ändå. I²C-skanningen visas i loggen.
Ta vid behov bort sensorernas block under `sensor:`. Tar du bort AHT20 måste du också
ta bort ENS160:s `compensation`-block eller hela ENS160-sensorn. Tar du bort BMP280,
ta även bort dess `copy`-sensor för havsnivåtryck. Vind-, regn- och riktningssensorerna
har interna ID-referenser; behåll deras sammanhörande block eller ta bort hela gruppen.

Justera värdena som redan finns under `substitutions`:

- `bmp280_address` kan behöva vara `'0x77'`, `ens160_address` kan behöva vara `'0x52'`.
- `altitude_m`, `temperature_offset`, `humidity_offset` och `pressure_offset` kalibrerar miljövärdena.
- `rain_mm_per_tip` är nederbörd per puls, standard `0.6314` mm.
- `wind_radius_mm`, `wind_magnets` och `wind_aerodynamic_factor` anpassas till vindmätaren.
- `light_transmission` kompenserar ett ljusdämpande skydd, större än 0 och högst 1.

Radie, magnetantal och regn per puls måste vara positiva. Standardvärdena kommer från
originalets mekanik och behöver kalibreras. Två AHT20 på samma adress kan inte dela buss.
Rikta vindflöjeln mot norr och tryck **Calibrate north** i Home Assistant. Kontrollera
också att öst ger cirka 90° och att AS5600:s DIR är stabilt ansluten enligt sensorboarden.

Vindhastigheten visas i m/s med medel över fem tvåsekundersprov. Vindby är max över
10 minuter. Vind går till noll efter 10 sekunder utan puls, innan medelvärdesfiltret.
Regnintensiteten beräknas mellan pulser och går till noll efter fem minuter utan puls;
första pulsen ökar totalen men ger ännu ingen intensitet. ENS160 behöver
[uppvärmningstid](https://esphome.io/components/sensor/ens160/).

Regntotal och nordoffset sparas i flash med fem minuters skrivintervall. Strömavbrott
kan förlora de senaste fem minuternas ändringar. Historik och tim-/dygnsmätare hanteras
i Home Assistant, till exempel med **Utility Meter** och `Rain total` som källa.
Dessa är kalenderperioder, inte rullande 1h/24h-värden.

## Bygga lokalt

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
.venv/bin/esphome config weatherstation.yaml
.venv/bin/esphome compile weatherstation.yaml
```

GitHub Actions validerar och bygger samtliga fem program med ESPHome 2026.8.2.
CI använder filernas offentliga exempeluppgifter och dess firmware är inte för drift.
Hårdvarutest av inkoppling, sensorer och OTA behöver göras på ditt kort.

Fork av [byte4geek/weatherstation](https://github.com/byte4geek/weatherstation).
ESP8266-kod, gamla binärfiler, webbgränssnitt och övrigt originalmaterial har tagits
bort från denna version och finns kvar i Git-historiken och originalrepot.
Ursprunglig upphovsrätt och [MIT-licens](LICENSE.md) är bevarade.
