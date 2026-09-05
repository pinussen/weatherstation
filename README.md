# Weather Station – ESP32 + ESPHome

Den här forken av [byte4geek/weatherstation](https://github.com/byte4geek/weatherstation)
är anpassad för **ESP32 och Home Assistant via ESPHome**. ESP32-C6 är huvudprofilen.
Firmware använder ESP-IDF och ESPHomes inbyggda sensordrivrutiner, krypterat lokalt API,
OTA, loggar och diagnostik. MQTT-broker behövs inte.

## Kort och inkoppling

| Profil i `esphome/` | Chip | SDA | SCL | Vindpuls | Regnpuls |
|---|---|---|---|---|---|
| `weatherstation-c6.yaml` | ESP32-C6 DevKit | GPIO6 | GPIO7 | GPIO10 | GPIO11 |
| `weatherstation-esp32.yaml` | ESP32 / WROOM DevKit | GPIO21 | GPIO22 | GPIO25 | GPIO26 |
| `weatherstation-c3.yaml` | ESP32-C3 DevKit | GPIO4 | GPIO5 | GPIO6 | GPIO7 |
| `weatherstation-s3.yaml` | ESP32-S3 DevKit | GPIO8 | GPIO9 | GPIO4 | GPIO5 |
| `weatherstation-s2.yaml` | ESP32-S2 DevKit | GPIO8 | GPIO9 | GPIO4 | GPIO5 |

Profilerna använder generisk chipvariant och 4 MB flash, utan krav på PSRAM eller
inbyggd LED. Kort med större flash kan också använda denna 4 MB-layout. Kontrollera
att pinnarna är utdragna och lediga på just ditt kort; ändra `substitutions` vid behov.
En C6 Super Mini eller XIAO har inte nödvändigtvis samma tillgängliga pinnar som en
DevKitC. [C6 DevKitC:s pinout](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32c6/esp32-c6-devkitc-1/user_guide.html).

Alla I²C-sensorer ansluts parallellt till SDA/SCL och gemensam jord, med pull-ups till
**3,3 V**. Pulssignalerna använder intern pull-up och aktiv låg signal. ESP32-ingångar
ska inte matas med 5 V. A3144 kan behöva 5 V matning: använd dess open-collector-utgång
med pull-up till 3,3 V, eller nivåomvandling om modulens utgång dras upp till 5 V.
Använd GPIO-numren ovan, inte ESP8266:s D1/D2-beteckningar eller gamla kopplingsbild.

## Installera och hantera från Home Assistant

1. Installera och starta **ESPHome Device Builder** i Home Assistant OS.
   [Officiell installationsguide](https://esphome.io/install/getting-started/).
   Kör du Home Assistant Container behövs en separat ESPHome Device Builder-container.
2. Ladda ned denna fork. Kopiera `esphome/weatherstation-c6.yaml` och hela
   `esphome/packages/` till Home Assistants `/config/esphome/`, till exempel med
   Studio Code Server eller Samba. Välj motsvarande YAML för ett annat chip.
3. Lägg till nycklarna från `esphome/secrets.example.yaml` i
   `/config/esphome/secrets.yaml`. Behåll eventuella befintliga secrets.
   Fyll i ditt WiFi och **egna** OTA/AP-lösenord. API-nyckeln ska vara 32 slumpbyte
   i base64; använd en nyckel från Device Builders enhetsguide eller generera en med:

   ```sh
   python3 -c 'import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())'
   ```

   Exempelnyckeln i repot är offentlig och endast avsedd för byggkontroller.
4. Öppna profilen i Device Builder. Ta bort paket för sensorer du saknar och justera
   pinnar/adresser vid behov. `air_quality` kräver `environment` för kompensation;
   ta bort båda om AHT20 saknas. Klicka **Validate**, sedan **Install**.
5. Första installationen görs via USB. Anslut till HA-maskinen och välj dess serieport,
   eller välj **Manual download → Factory format** och installera via
   [ESPHome Web](https://web.esphome.io/) i Chrome/Edge på datorn med USB-kabeln.
   Om kortet inte hittas, håll BOOT medan du trycker RESET och försök igen.
6. Lägg till den upptäckta ESPHome-enheten under **Inställningar → Enheter och tjänster**.
   Ange API-nyckeln om HA frågar. Vid utebliven upptäckt kan du lägga till ESPHome
   manuellt med enhetens IP-adress.
7. Därefter används **Install → Wirelessly**, **Logs** och **Edit** i Device Builder.
   Sensorer, omstart, säkert läge och nordkalibrering finns i Home Assistant.

WiFi använder 2,4 GHz. Vid anslutningsproblem startas ett lösenordsskyddat reservnät
med captive portal. HA måste kunna nå enhetens API (TCP 6053), och Device Builder
måste kunna nå OTA (TCP 3232). Enheten fortsätter samla data när HA är avstängt.

## Alternativ: hämta konfigurationen direkt från GitHub

För en komplett C6-station kan du skapa en enhet i Device Builder och ersätta dess
YAML med följande. Samma secrets som ovan behövs i Device Builders `secrets.yaml`.

```yaml
substitutions:
  name: weatherstation-c6
  friendly_name: Weather Station
packages:
  station:
    url: https://github.com/pinussen/weatherstation
    ref: main
    files:
      - esphome/weatherstation-c6.yaml
    refresh: 1d
```

Byt filnamnet för ett annat chip. Egna substitutions för pinnar och kalibrering kan
läggas i samma block. ESPHome hämtar även de inkluderade paketfilerna automatiskt.
`main` följer ändringar i forken vid nästa hämtning/bygge; använd ett commit-ID som
`ref` om du vill låsa versionen. Ingen firmware installeras automatiskt vid en
repoändring – du väljer fortfarande **Install**. Använd lokal kopiering enligt ovan
om du vill redigera paket eller välja bort sensorer.

## Sensorer och kalibrering

| Paket | Funktion | Standardadress |
|---|---|---|
| `environment` | AHT20 temperatur/fukt, BMP280 stations- och havsnivåtryck | `0x38`, `0x76` |
| `air_quality` | ENS160 TVOC, eCO2 och AQI | `0x53` |
| `light` | BH1750 ljusstyrka | `0x23` |
| `wind` | Pulsgivare, vindhastighet och högsta värde senaste 10 minuterna | GPIO |
| `direction` | AS5600 vindriktning, cirkulärt medel och nordkalibrering | `0x36` |
| `rain` | Pulsgivare, regnintensitet och sparad total nederbörd | GPIO |

Paket väljs uttryckligen; det finns ingen automatisk aktivering av sensorer.
I²C-skanningen visas i loggen. BMP280 kan behöva `bmp280_address: '0x77'`, ENS160
`ens160_address: '0x52'`. Saknade sensorer ger loggfel och otillgängliga värden.
Två AHT20 på samma adress kan inte dela buss; använd bara en ansluten AHT20.

Lägg till önskade värden under profilens befintliga `substitutions`, exempelvis:

```yaml
  altitude_m: '125.0'
  temperature_offset: '-0.5'
  humidity_offset: '0.0'
  pressure_offset: '0.0'
  rain_mm_per_tip: '0.6314'
  wind_radius_mm: '80.0'
  wind_magnets: '1.0'
  wind_aerodynamic_factor: '3.0'
  light_transmission: '1.0'
```

Radie, magnetantal och regnvolym måste kalibreras mot din mekanik; originalets
standardvärden är bara utgångsvärden. Radie, magnetantal och regn per puls måste vara
positiva. Ljusets transmission ska vara större än 0 och högst 1.
Rikta vindflöjeln mot norr och tryck **Calibrate north** i HA. Kontrollera också att
öst ger cirka 90°; AS5600:s DIR ska vara stabilt ansluten enligt sensorboardens anvisning.

Vindhastighet visas i m/s, med medel över fem tvåsekundersprov. Vindby är max över
300 sådana prov (10 minuter), inte meteorologiskt certifierad bymätning. Pulsfri vind
går till noll efter 10 sekunder, innan medelvärdesfiltret. Regnintensiteten beräknas
mellan pulser och går till noll efter fem minuter utan puls; första pulsen ökar totalen
men ger ännu ingen intensitet. ENS160:s eCO2 är ett uppskattat värde, inte en CO2-mätare.
[ENS160-dokumentationen](https://esphome.io/components/sensor/ens160/) beskriver uppvärmningstiden.

Regntotal och nordoffset sparas i flash med fem minuters skrivintervall för att minska
slitage. Ett strömavbrott kan förlora de senaste fem minuternas ändringar. Historik och
valfria tim-/dygnsmätare hanteras i Home Assistant, exempelvis med hjälparen
**Utility Meter** och `Rain total` som källa. Dessa är kalenderperioder; originalets
rullande 1h/24h-värden och dagliga vindby finns inte i denna version.

## Utveckling och verifiering

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
cp esphome/secrets.example.yaml esphome/secrets.yaml
.venv/bin/esphome config esphome/weatherstation-c6.yaml
.venv/bin/esphome compile esphome/weatherstation-c6.yaml
```

Byggverktygen är låsta till ESPHome 2026.8.2; ESPHome väljer kompatibla ESP-IDF-
och drivrutinsversioner. GitHub Actions validerar och bygger samtliga fem profiler.
CI använder offentliga test-secrets och dess firmware ska inte installeras i drift.
Profilerna behöver även provas med riktig hårdvara: I²C-adresser, pulser, riktning,
WiFi och OTA. Ett lyckat bygge garanterar inte ett visst tredjepartskorts pinout.

Originalets ESP8266-program finns kvar i `src/`, `include/` och `platformio.ini` som
referens, med [ursprunglig README](README.esp8266.md). Det byggs **inte** av ESPHome.
De gamla `.bin`-filerna gäller endast ESP8266. Webbgränssnitt, REST-API, MQTT,
WiFiManager och originalets externa telemetri ingår inte i ESP32-firmware.
Mekanik, bilder och materiallistor finns kvar för den som behöver dem.

Ursprunglig upphovsrätt och MIT-licens finns i [LICENSE.md](LICENSE.md).
