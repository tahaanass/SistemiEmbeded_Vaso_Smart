# Vaso Smart Closed-Loop: Sistema Embedded IoT per il Microclima e l'Irrigazione del Basilico 

Sistema embedded per il monitoraggio ed il controllo autonomo in retroazione (*closed-loop*) del microclima e dell'irrigazione per una pianta di **basilico** coltivata in vaso da interni.

Il firmware è sviluppato per microcontrollore **ESP32** in ambiente Arduino C++ e adotta un'architettura **non bloccante** guidata da una **Macchina a Stati Finiti (FSM)**. Il sistema integra algoritmi di **isteresi dinamica circadiana (Giorno/Notte)**, strategie di **condizionamento del segnale** (oversampling con rigetto dei picchi), meccanismi di **Safety Interlock** hardware/software e un'infrastruttura di telemetria e controllo remoto bidirezionale via **MQTT** su Wi-Fi.

> **Progetto per il corso di Sistemi Embedded**  
> **Ateneo**: Università degli Studi di Milano — Dipartimento di Informatica  
> **Licenza**: GNU General Public License v3.0 (GPLv3)

---

## Indice della Documentazione
1. [Obiettivi Ingegneristici & Approccio Closed-Loop](#obiettivi-ingegneristici--approccio-closed-loop)
2. [Architettura del Sistema](#architettura-del-sistema)
3. [Mappa della Repository](#mappa-della-repository)
4. [Sintesi Hardware & Vincoli di Silicio (ADC1 vs ADC2)](#sintesi-hardware--vincoli-di-silicio-adc1-vs-adc2)
5. [Guida Rapida all'Avvio (Quickstart)](#guida-rapida-allavvio-quickstart)
6. [Panoramica Funzionale della FSM](#panoramica-funzionale-della-fsm)
7. [Infrastruttura IoT & Telemetria MQTT](#infrastruttura-iot--telemetria-mqtt)
8. [Licenza e Autori](#licenza-e-autori)

---

## Obiettivi Ingegneristici & Approccio Closed-Loop

Il basilico presenta requisiti agronomici stringenti: è sensibile allo stress idrico ma teme i ristagni d'acqua e l'asfissia radicale, terreno fertile per patogeni fungini.

### Principi cardine di progetto:
- **Controllo Retroazionato Negativo**: Il controller calcola l'errore $e(t) = r(t) - y(t)$ e comanda gli attuatori finché il valore misurato $y(t)$ non rientra nella finestra tollerata.
- **Isteresi Anti-Chattering**: Previene commutazioni ad alta frequenza del relè meccanico provocate dal rumore di campionamento dell'ADC.
- **Safety Interlock su Tre Livelli**:
  1. *Livello Serbatoio*: Arresto istantaneo della pompa in caso di assenza d'acqua via galleggiante fail-safe.
  2. *Timeout Pompa*: Durata massima di azionamento continuo pari a $4000\text{ ms}$ (`MAX_POMPA_RUN_MS`).
  3. *Cooldown Period*: Blocco forzato di ri-innesco della pompa per $60000\text{ ms}$ (`COOLDOWN_POMPA_MS`) per consentire la percolazione capillare.
- **Resilienza di Rete**: Continuità operativa al 100% in modalità offline in caso di disconnessione Wi-Fi o caduta del broker MQTT.

---

## Architettura del Sistema

![Schema Architettura](img/architettura.png)

---

## Mappa della Repository

```text
📁 SistemiEmbedded_Vaso_Smart/
├── 📄 README.md                         ──► Panoramica generale, obiettivi e quickstart
├── 📁 VasoSmart/                        ──► Codice sorgente modulare pronto per Arduino IDE
│   ├── 📄 config.h
│   ├── 📄 VasoSmart.ino
│   ├── 📄 sensors.ino
│   ├── 📄 actuators.ino
│   ├── 📄 fsm.ino
│   ├── 📄 display.ino
│   └── 📄 mqtt_wifi.ino
├── 📁 img/                              ──► Immagini, schemi elettrici e foto del prototipo
└── 📁 docs/                             ──► Documentazione tecnica di dettaglio:
    ├── 📄 Hardware_Setup.md             ──► Circuiti, piano di alimentazione, BOM e calibrazione
    ├── 📄 FSM_Isteresi.md               ──► Macchina a stati, isteresi circadiana e safety timers
    ├── 📄 MQTT_Protocol.md              ──► Albero topic, payload JSON e configurazione client
    └── 📄 Testing_Demo.md               ──► Metodologia Sensor Mocking e matrice test case
```

- **[`docs/Hardware_Setup.md`](docs/Hardware_Setup.md)**: Consultare per collegamenti elettrici, scelta resistenze di pull-up, ripartizione carichi 5V/3.3V e formule di linearizzazione.
- **[`docs/FSM_Isteresi.md`](docs/FSM_Isteresi.md)**: Consultare per la logica di transizione degli stati, la matrice delle soglie giorno/notte e i meccanismi di protezione.
- **[`docs/MQTT_Protocol.md`](docs/MQTT_Protocol.md)**: Consultare per integrare dashboard esterne (Home Assistant, Node-RED, MQTT Explorer) o monitorare la telemetria.
- **[`docs/Testing_Demo.md`](docs/Testing_Demo.md)**: Consultare per collaudare il firmware a banco tramite comandi seriali.

---

## Sintesi Hardware & Vincoli di Silicio (ADC1 vs ADC2)

![Prototipo e Circuito](img/circuito.png)

| Componente | Interfaccia ESP32 | Tensione Bus | Categoria Componente |
| :--- | :--- | :--- | :--- |
| **ESP32 DevKit V1** | — | 5V (VIN) / 3.3V Logic | Elaborazione centrale |
| **SEN0193 (Umidità Terreno)** | **GPIO 34 (ADC1_CH6)** | 3.3V | Input Analogico |
| **Fotoresistenza LDR + R 10kΩ** | **GPIO 35 (ADC1_CH7)** | 3.3V | Input Analogico (Partitore) |
| **Galleggiante Serbatoio** | **GPIO 32 (Input Pull-up)**| 3.3V | Input Digitale Fail-Safe |
| **DHT11 (Temp. / Umidità Aria)**| **GPIO 4 (1-Wire)** | 3.3V | Input Digitale |
| **Display OLED SSD1306** | **GPIO 21 (SDA), 22 (SCL)** | 3.3V | Bus I2C (0x3C) |
| **Modulo 4 Relè Optoisolati** | **GPIO 26, 25, 16** | 5V VCC / 3.3V Logic | Uscite Digitali (Active-LOW) |
| **Carichi 5V (Pompa, Fan, Mist)**| Uscite contatti Relè | 5V DC Esterna | Attuatori Idraulici/Aeraulici |

> **Vincolo Hardware Critico (ADC1 vs ADC2)**:  
> Sull'architettura ESP32, il convertitore `ADC2` condivide il multiplexer e il clock interno con il sottosistema radio Wi-Fi. Quando la connessione Wi-Fi o lo stack MQTT è operativo, le letture sui pin dell'ADC2 restituiscono errori o valori nulli.  
> **Tutti i sensori analogici del sistema sono stati instradati rigorosamente su pin appartenenti ad ADC1 (GPIO 34 e GPIO 35)**, garantendo letture stabili e immuni dall'attività wireless.

---

## Guida Rapida all'Avvio (Quickstart)

### 1. Requisiti Software
- **Arduino IDE 2.x** con supporto board ESP32 (Espressif v2.0+).
- Librerie richieste: `PubSubClient`, `Adafruit SSD1306`, `Adafruit GFX Library`, `DHT sensor library`.

### 2. Configurazione & Flash
1. Apri la cartella `VasoSmart/` aprendo il file principale con Arduino IDE.
2. Configura le credenziali Wi-Fi e i parametri del broker nel file `config.h`:
   ```cpp
   const char* WIFI_SSID     = "IL_TUO_WIFI";
   const char* WIFI_PASSWORD = "LA_TUA_PASSWORD";
   const char* MQTT_BROKER   = "mqtt.atrent.it";
   const int   MQTT_PORT     = 1883;
   ```
3. Seleziona la scheda **ESP32 Dev Module** e la porta seriale corrispondente.
4. Premi **Upload** (`Ctrl+U`).
5. Apri il Serial Monitor a **`115200 baud`** per verificare l'avvio e la telemetria.

---

## Panoramica Funzionale della FSM

Il microcontrollore non blocca mai l'esecuzione con delay sincroni: ogni decisione è regolata da una Macchina a Stati Finiti cooperativa aggiornata periodicamente da un loop non bloccante.

![Diagramma Transizioni FSM](img/fsm.png)

Per l'approfondimento matematico delle soglie e delle transizioni, consulta **[`docs/FSM_Isteresi.md`](docs/FSM_Isteresi.md)**.

---

## Infrastruttura IoT & Telemetria MQTT

I dati ambientali e lo stato della FSM vengono pubblicati periodicamente con periodo nominale di 5 secondi:

- `vasosmart/terreno/umidita`: Umidità volumetrica stimata del suolo ($0.0 - 100.0\%$).
- `vasosmart/aria/temperatura`: Temperatura ambiente misurata dal DHT11 in $^{\circ}\text{C}$.
- `vasosmart/aria/umidita`: Umidità relativa ambiente in $\%$.
- `vasosmart/serbatoio/stato`: Condizione del serbatoio (`OK` oppure `VUOTO`).
- `vasosmart/fsm/stato`: Stato operativo FSM (`MONITORAGGIO`, `IRRIGAZIONE`, `ERRORE`, ...).
- `vasosmart/cmd/mode`: Canale di ricezione comandi remoti (`AUTO` o `MANUAL`).

Per la specifica esaustiva dei payload e il testing con MQTT Explorer, consulta **[`docs/MQTT_Protocol.md`](docs/MQTT_Protocol.md)**.

---

## Licenza e Autori

- **Autori**: Seddik Taha Anass & Zmarandache Daniele
- **Licenza**: Distribuito con licenza open-source [GNU General Public License v3.0 (GPLv3)](https://www.gnu.org/licenses/gpl-3.0.html).
