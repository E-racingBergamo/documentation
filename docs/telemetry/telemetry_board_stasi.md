# Telemetria 2026 - 2027

**Stasi Samuele - 25/09/2026**
**Nominativo progetto - Julia**

---
## Obiettivo del progetto

L'obiettivo è progettare produrre e programmare una scheda di telemetria che supporti sia l'acquisizione dati in locale che la telemetria wireless. La motivazione che ci ha spinto ad iniziare il presente progetto è la necessità di avere un supporto tecnologico più robusto, sia dal punto di vista elettrico che dal punto di vista della trasmissione dati.

La scheda utilizzerà una trasmissione LoRa a 868MHz ad alta potenza per garantire un'adeguata portata del segnale e una relativamente alta sopportazione dei disturbi elettromagnetici. A supporto di future necessità verrà scelto un modulo di trasmissione che sia in grado di trasmettere anche segnali in modalità FSK, che gli sbloccano una capacità di canale molto più ampia se ci si trova entro una data distanza. Maggiori informazioni verranno fornite nel prossimo capitolo.

Per la persistenza in locale si è optato per una semplice scheda MicroSD.

I dati verranno ricevuti dalla scheda attraverso una linea CAN 2.0A/2.0B

## Gestione della Banda

Questa sezione del documento punta a verificare che il carico di dati sia in linea con la capacità di canale di LoRa. Qualora non fosse sufficiente si esploreranno alcuni possibili algoritmi per introdurre un datarate variabile basato sulle misurazione in tempo reale della potenza del segnale da parte del modulo di trasmissione integrato.

Il firmware dell'ECU gestisce l'acquisizione dei dati serializzando due pacchetti strutturati in C privi di padding grazie all'istruzione `__attribute__((packed))`, entrambi validati da un codice a ridondanza ciclica CRC16 per identificare e scartare i pacchetti corrotti:

- Il primo pacchetto `TelemetryPacket_t` arriva dalla centralina e contiene i dati dei sensori, i parametri dell'inverter, le macchine a stati e le elaborazioni dinamiche che fa la ECU. Il pacchetto pesa `202byte` e viene trasmesso a 200Hz
```cpp
typedef struct __attribute__((packed)) {
// --- Telemetry packet id and metadata ---
uint8_t startByte1; // 0xCC
uint8_t startByte2; // 0x11
uint32_t timestamp; // Incremental
// --- Wheel velocities ---
float front_left_velocity; // km/h
float front_right_velocity; // km/h
float rear_left_velocity; // m/s
float rear_right_velocity; // m/s
// --- Suspension length ---
// The sensor is implemented but the measurement unit is at this time unknown
// 0<->4096
float front_left_suspension; // To be defined
float front_right_suspension; // To be defined
// --- APPS ---
uint16_t accelerator1; // 0<->4096
uint16_t accelerator2; // 0<->4096
float accelerator_mapped; // 0<->1
// --- Brake ---
uint16_t brake1; // 0<->4096
uint16_t brake2; // 0<->4096
// --- Steer ---
float steer; // Degrees
// --- Digital in ---
uint8_t sdc; // Digital 0/1
uint8_t ready_to_drive_button; // Digital 0/1
uint8_t ecu_reset_button; // Digital 0/1
uint8_t tractive_system_on_button; // Digital 0/1
// --- Battery (EMMA) ---
float emma_current; // A
float emma_voltage; // V
float emma_yaw; // deg/s
uint16_t emma_error; // Emma error struct
// --- Vehicle Dynamics & Powertrain Targets ---
float mean_velocity; // m/s
float real_yaw_rate; // deg/s
float total_torque_request;
float torque_tv_L; // nm
float torque_tv_R; // nm
float slip_L; // Adimensional raw
float slip_R; // Adimensional raw
float torque_reduction_L;
float torque_reduction_R;
float final_torque_target_L; // nm
float final_torque_target_R; // nm
// --- Inverter Internal Data ---
int16_t inv_L_torqueCurrent; // Raw data A?
int16_t inv_L_magnetizingCurrent; // Raw data A?
int16_t inv_L_tempMotor; // °C
int16_t inv_R_torqueCurrent; // Raw data A?
int16_t inv_R_magnetizingCurrent; // Raw data A?
int16_t inv_R_tempMotor; // °C
// --- External Thermal Data ---
float left_coolant_temp; // °C
float right_coolant_temp; // °C
// --- State machines ---
uint8_t left_inverter_fsm; // FSM struct
uint8_t right_inverter_fsm; // FSM struct
uint8_t tractive_system_fsm; // FSM struct
// --- ECU mode ---
uint8_t ECU_Mode; // ECU mode struct
uint16_t IMDInsulationValue; // KOhm
// --- Inverter diagnostics (AMK diagnostic number, cfr. INVERTER_Errors.h) ---
// 0 = nessuna diagnostica attiva. E' lo stesso numero mostrato sul display
// dell'inverter: si decodifica a terra con la tabella di INVERTER_Errors.h
uint16_t inv_L_errorInfo;
uint16_t inv_R_errorInfo;

int16_t inv_L_torque_feedback; // 0.1 nm
int16_t inv_R_torque_feedback; // 0.1 nm
uint32_t inv_L_magnetising_currentFW;
uint32_t inv_R_magnetising_currentFW;
uint16_t inv_L_service_bits;
uint16_t inv_R_service_bits;

uint16_t inv_L_dc_bus_voltage; // V
uint16_t inv_R_dc_bus_voltage; // V
uint16_t inv_L_dc_bus_voltage_monitoring; // jkv?>
uint16_t inv_L_controller_enable;
uint16_t inv_R_controller_enable;
int16_t inv_L_temperature_internal; // °C
int16_t inv_R_temperature_internal; // °C
uint32_t inv_L_actual_power_value; // W?
uint32_t inv_R_actual_power_value; // W?
int16_t inv_L_temperature_external; // °C
int16_t inv_R_temperature_external; // °C

// Inverter internal temperatures
uint16_t inv_L_temp_igbt; // °C
uint16_t inv_R_temp_igbt; // °C
uint16_t inv_L_temp; // °C
uint16_t inv_R_temp; // °C

// IMU Data
uint16_t IMU_gyro_x; // 0.1 mdps (millidegree per second)
uint16_t IMU_gyro_y; // 0.1 mdps (millidegree per second)
uint16_t IMU_gyro_z; // 0.1 mdps (millidegree per second)
uint16_t IMU_acc_x; // 0.1 mg
uint16_t IMU_acc_y; // 0.1 mg
uint16_t IMU_acc_z; // 0.1 mg

// --- CRC for corrupted packages ---
uint16_t crc16; // CRC16

} TelemetryPacket_t;
```

- `TelemetryPacket_EMMA_t` pesa 81 Byte e contiene lo Stato di Carica, le matrici di tensione e temperatura delle singole celle, e il registro dei moduli attivi per la diagnostica del pacco batteria. La sua frequenza di trasmissione è di 5Hz
```cpp
typedef struct __attribute((packed)){
// --- Telemetry packet id and metadata ---
uint8_t startByte1; // 0xEE
uint8_t startByte2; // 0x11
uint32_t timestamp;

// --- Module states ---
uint8_t state_of_charge; // percentage
uint8_t module_voltage[14]; // V
uint16_t module_temperatures[28]; // degrees 0.1 °C
uint16_t onlineModules;

// --- CRC for corrupted packages ---
uint16_t crc16;

} TelemetryPacket_EMMA_t;
```

Per calcolare la banda:

- `TelemetryPacket_t`: $202 b * 200 = 40.400 B/s = 323.2kbps$.
- `TelemetryPacket_EMMA_t`: $79 b * 5 = 395 B/s = 3,16 kbps$.
- **Totale flusso dati utile:** 326,36 kbps.

### Implicazioni per comunicazione LoRa

La comunicazione LoRa può essere effettuata su tre distinte frequenze portanti (433Mhz, 868Mhz e 2.4Ghz).
La frequenza dei 433Mhz non è praticabile per via del suo ridotto bitrate, quindi la scelta deve ricadere su una delle altre due frequenze. Le bande sono regolamentate dall'unione europea secondo i seguenti parametri:

| Sub-banda          | Potenza max | Duty cycle | Note                                               |
| ------------------ | ----------- | ---------- | -------------------------------------------------- |
| 863.0 – 868.0 MHz  | 25 mW ERP   | 0.1% o 1%  | Varia per sotto-banda. La 868.0-868.6 MHz ha DC 1% |
| 868.0 – 868.6 MHz  | 25 mW ERP   | 1%         | Uso generale LoRa/ LoRaWAN                         |
| 868.7 – 869.2 MHz  | 25 mW ERP   | 0.1%       | Uso generale, duty cycle molto restrittivo         |
| 869.4 – 869.65 MHz | 500 mW ERP  | 10%        | **Sub-banda usata da MeshCore e Meshtastic**       |
| 869.7 – 870.0 MHz  | 25 mW ERP   | 1%         | Uso generale                                       |
| 4.4GHz             | 200 mW ERP  | 100%       | Uso generale                                       |

#### Analisi Bottleneck
Sommando i flussi dei due pacchetti di telemetria si ottiene un totale di flusso di 326,36kbps, che pur essendo ridotto, è comunque eccessivo per il modulo LoRa che nella sua modalità normale ha una capacità di canale massima teorica di 62,5kbps. Risulta quindi fisicamente impossibile trasmettere l'intero flusso dati in tempo reale via radio senza incorrere in saturazione della banda, collisioni o svuotamento incontrollato del FIFO di trasmissione.

Inoltre le leggi europee vietano di sfruttare tutta la banda teorica della frequenza 868MHz con la clausola del duty cycle, che abbatte drasticamente la capacità di canale e rende l'utilizzo di questa portante pressoché impossibile.
Per risolvere questa criticità, il progetto adotta una strategia ibrida basata su una mitigazione software e un cambio di portante.

Gestione del Dato (Software)
Per risolvere la situazione si utilizzano due instradamenti diversi per i dati: appena i pacchetti arrivano vengono instradati direttamente nella memoria della scheda MicroSD con lo scopo di di salvare tutti i dati senza perdere di risoluzione nel post processing. Nell'instradamento del canale viene utilizzato un algoritmo che ridimensiona la frequenza del primo pacchetto di telemetria in termini di frequenza a 10Hz-20Hz, rendendo la capacità totale richiesta a circa 15-25 kbps.

Per massimizzare l'efficienza di trasmissione, si prevede l'implementazione di un sistema di **Adaptive Data Rate**.

**1. Il Feedback Loop (Rilevamento della Qualità del Link)** Affinché il microcontrollore possa adattare il traffico dati, necessita di una metrica sulla qualità del canale. Sfruttando i registri interni del ricetrasmettitore, è possibile leggere in hardware il valore **RSSI** (Received Signal Strength Indicator) e l'**SNR** (Signal-to-Noise Ratio). L'architettura software prevederà l'invio di un pacchetto di ACK periodico dalla Ground Station verso la vettura; analizzando l'RSSI di questo pacchetto in ingresso, l'ESP32 avrà una stima istantanea del Path Loss del canale radio.

**2. Strategie di Adattamento Dinamico** Una volta nota la qualità del segnale, il firmware interverrà dinamicamente:

- **Transizione di Modulazione e SF Scaling:** Il sistema modificherà lo stato in base all'SNR. In rettilineo (SNR pulito), utilizzerà la modulazione FLRC (fino a 1.3 Mbps). Se il rumore dell'inverter degrada il segnale, scalerà alla modulazione LoRa 2.4 GHz, incrementando lo Spreading Factor (da SF5 a SF12). Questo abbasserà la soglia di sensibilità del ricevitore (fino a $-22.5$ dB), garantendo la connessione al costo di un calo del throughput.
    
- **Larghezza di Banda (BW):** Il punto di lavoro base sarà fissato a $BW = 812$ kHz. Questo offre un equilibrio perfetto per mantenere un basso _Time-On-Air_ (limitando l'esposizione ai disturbi dell'inverter) e un data rate di $44.4$ kbps (a SF7).
    
- **Priority Payload:** In condizioni critiche di segnale o forte attenuazione per ostacoli (shadowing), l'ESP32 sospenderà l'invio del pacchetto dinamico da 140 byte, commutando su un pacchetto "Emergency" ultra-ridotto contenente esclusivamente informazioni di sicurezza vitali.

#### Scelta Architetturale Hardware (Modulo E28 a 2.4 GHz)

Per superare i limiti di legge legati al Duty Cycle degli 868 MHz e avere un margine di banda adeguato, la scelta hardware è ricaduta sul modulo **Ebyte E28-2G4M27SX**. Questo componente è basato sul transceiver Semtech SX1280, che trasporta la tecnologia LoRa sulla banda dei 2.4 GHz.

Questa scelta sblocca vantaggi operativi fondamentali per la telemetria:

- **Duty Cycle al 100%:** Operando a 2.4 GHz, la vettura può trasmettere dati in modo continuo senza pause forzate, garantendo fluidità alla dashboard dei box.
- **Triplice Modulazione:** Il chip supporta la modalità LoRa classica (fino a 253 kbps per la massima resistenza ai disturbi dell'inverter), la modalità FSK (fino a 2.0 Mbps per alta velocità) e la rivoluzionaria modalità **FLRC** (Fast Long Range Communication).
- **Throughput Elevato in FLRC:** Questa modulazione proprietaria garantisce velocità fino a **1.3 Mbps**, tollerando al contempo un moderato rumore condotto con SNR leggermente negativi (fino a -5 dB).
- **Potenza e Sensibilità:** Il modulo integra un amplificatore che può spingere fino a 27 dBm (limitabile via software a 20 dBm per compliance ETSI in Europa) e riceve segnali fino a -132 dBm.
- **Globalmente accettata:** La banda 2.4 GHz ISM è libera e utilizzabile in tutto il mondo, requisito cruciale per le competizioni estere del team.

L'interfacciamento del modulo E28 avviene tramite bus SPI dedicato, pilotato nativamente dai task FreeRTOS del microcontrollore ESP32, permettendo un controllo a bassissima latenza dei pin per gli switch d'antenna in trasmissione e ricezione.

## Componenti di base

??? Info "Riguardi"
	 Questo progetto tiene conto di possibili sviluppi futuri; avere un occhio di riguardo per queste cose permette di avere un progetto molto solido e di risparmiare tempo e soldi per eventuali modifiche

### E28-2G4M27SX - SX1280 2.4GHz Wireless Module

Questo modulo contiene un chip avanzato per la comunicazione LoRa. Le principali caratteristiche sono le seguenti:

- **Triplice Modulazione:** Supporto nativo per protocolli LoRa, FLRC (Fast Long Range Communication) e (G)FSK.
- **Throughput Elevato:** Fino a 253 kbps in modulazione LoRa, fino a 1.3 Mbps in FLRC e 2.0 Mbps in FSK.
- **Potenza di Trasmissione:** Amplificatore di potenza (PA) integrato fino a 27 dBm (500 mW), scalabile via software per rispettare il limite europeo ETSI di 20 dBm (100 mW) EIRP.
- **Sensibilità in Ricezione:** Fino a -132 dBm in modalità LoRa, garantendo un eccellente link budget.
- **Frequenza Operativa:** Banda ISM 2.4 GHz, utilizzabile legalmente e senza limitazioni di Duty Cycle.
- **Alimentazione:** Tolleranza da 2.5V a 5.5V.
- **Interfaccia di Controllo:** Bus SPI ad alta velocità compatibile con i livelli logici a 3.3V del microcontrollore.

Da questi dati risulta palese il motivo architetturale della scelta: il modulo risolve definitivamente il collo di bottiglia della banda passante. In particolare, la modulazione proprietaria **FLRC** offre il compromesso perfetto per il motorsport, garantendo un flusso dati massiccio unito a un forte Forward Error Correction capace di tollerare i disturbi elettromagnetici condotti generati dall'inverter (resistendo a SNR leggermente negativi).

L'interfaccia hardware con l'ESP32-S3 avviene tramite un bus SPI dedicato (SPI 3). Per la gestione ottimale dell'energia e del flusso radio, il microcontrollore pilota direttamente i pin `RXEN` e `TXEN` del modulo, commutando l'antenna e l'amplificatore interno con tempistiche precise dettate dai task del sistema operativo.

**Consumi**

| Stadio                | Consumo | Note                           |
| --------------------- | ------- | ------------------------------ |
| Emission current (mA) | 580     | Consumo di corrente istantaneo |
| Receive current (mA)  | 14.5    |                                |
| Sleep current (μA)    | 2.0     |                                |


![[lora_footprint.png]]

### **ESP32-S3-WROOM-1U** (N8R2)

Come microcontrollore è stato scelto un ESP32-S3-WROOM-1U nella sua versione da 8MB di flash e 2MB di PSRAM. La dicitura 1U sta ad intendere che non è presente l'antenna sulla PCB del modulo, ma un connettore per antenna esterna. Sia questo modulo che il modulo LoRa presentano la stessa soluzione per antenna esterna per il semplice fatto che un body costruito in materiale composito, soprattutto fibra di carbonio, essendo conduttivo potrebbe totalmente o parzialmente bloccare le emissioni elettromagnetiche. L'antenna esterna permette un superiore guadagno essendo in linea d'aria con il ricevitore.

Sebbene la telemetria a lungo raggio sia affidata al modulo LoRa, l'antenna dell'ESP32-S3 sblocca le funzionalità radio native del chip con Wi-Fi 4 a 2.4 GHz e Bluetooth 5.0 Low Energy. Questa scelta architetturale apre a due scenari fondamentali per l'operatività in pista:

- **Scaricamento dati e OTA:** Il Wi-Fi permette di scaricare i log pesanti ad alta velocità quando la vettura rientra ai box, o di eseguire aggiornamenti firmware OTA senza dover smontare la scocca per accedere fisicamente alla porta USB della centralina.

- **Diagnostica rapida:** Il Bluetooth LE può essere utilizzato per connettere rapidamente un tablet o uno smartphone per visualizzare lo stato della vettura, gli errori o i valori dei sensori durante le fasi di ispezione o di setup pre gara.

**Matrice GPIO e Flessibilità di Routing**

Un fattore decisivo nella scelta della famiglia ESP32-S3 è l'avanzata matrice GPIO. A differenza dei microcontrollori tradizionali dove le periferiche hardware sono legate a pin fissi, l'ESP32-S3 permette di instradare quasi ogni segnale digitale (SPI, UART, I2C, PWM) su qualsiasi pin libero. Questa caratteristica ha semplificato drasticamente lo sbroglio (routing) del PCB: ha permesso di sbrogliare le piste del bus SPI verso la MicroSD e le linee di comunicazione verso il modulo LoRa in modo diretto e parallelo, riducendo al minimo l'uso di via passanti e migliorando l'integrità dei segnali.

![[ESP32_pin_layout.png|300]]![[ESP_pinout.png|350]]

Le immagini sopra mostrano il layout fisico del modulo ESP32-S3 e la corrente configurazione di pinout scelta per ottimizzare lo sbroglio in fase di progettazione della PCB.

**Consumi**

A livello energetico, l'ESP32-S3 richiede una gestione termica adeguata implementata tramite pad termico e vie di dissipazione sul PCB. I consumi variano in base allo stato:

- **Deep Sleep:** ~7 µA
- **Elaborazione attiva (Radio Off):** ~20-30 mA
- **Picco di trasmissione Wi-Fi:** Fino a ~350 mA

### Linee di comunicazione

Per mantenere le comunicazioni pulite tra il microcontrollore e le periferiche evitando di aggiungere complessità software per l'arbitraggio è stato deciso di utilizzare una linea diversa per ognuno:

- il modulo di comunicazione LoRa sulla SPI3
- il modulo SD può essere cablato sulla periferica **SDMMC**, specificatamente progettata per comunicare con le MicroSD
- il controller CAN va sulla SPI3

## Calcolo di congestione della rete CAN

Il modulo di telemetria, oltre ad occuparsi della telemetria, si occupa anche di mantenere in memoria alcuni dei parametri modificabili a bordo vettura, come ad esempio i limiti di coppia. Per motivi di sincronizzazione con il resto del sistema della vettura sono necessari dei messaggi di servizio che si andranno a sommare a quelli della telemetria.

Per capire se questo risulta un problema bisogna calcolare la banda necessaria per l'invio periodico di tutti i pacchetti di telemetria e aggiungere in un periodo anche l'eventuale passaggio dei pacchetti di servizio.

Per comporre il messaggio principale di telemetria a 200hz sono necessari i seguenti pacchetti CAN:

| **ID (Dec)** | **Nome Messaggio**           | **Nodo**  | **Segnali Inclusi**                                                                                                                                                                                                                                                                                                             |
| ------------ | ---------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 288          | ECU_Time                     | ECU       | timestamp                                                                                                                                                                                                                                                                                                                       |
| 289          | ECU_WheelVel                 | ECU       | front_left_velocity, front_right_velocity, rear_left_velocity, rear_right_velocity                                                                                                                                                                                                                                              |
| 290          | ECU_Pedals                   | ECU       | accelerator1, accelerator2, brake1, brake2                                                                                                                                                                                                                                                                                      |
| 291          | ECU_SuspSteer                | ECU       | front_left_suspension, front_right_suspension, steer, accelerator_mapped                                                                                                                                                                                                                                                        |
| 292          | ECU_States                   | ECU       | SDC, ready_to_drive_button, ecu_reset_button, tractive_system_on_button, left_inverter_fsm, right_inverter_fsm, tractive_system_fsm, ECU_Mode, IMDInsulationValue                                                                                                                                                               |
| 293          | ECU_Emma                     | ECU       | emma_voltage, emma_current, emma_yaw, emma_error                                                                                                                                                                                                                                                                                |
| 294          | ECU_Dynamics1                | ECU       | mean_velocity, real_yaw_rate, slip_L, slip_R                                                                                                                                                                                                                                                                                    |
| 295          | ECU_Torque1                  | ECU       | total_torque_request, torque_tv_L, torque_tv_R, torque_reduction_L                                                                                                                                                                                                                                                              |
| 296          | ECU_Torque2                  | ECU       | torque_reduction_R, final_torque_target_L, final_torque_target_R                                                                                                                                                                                                                                                                |
| 297          | ECU_InvLeft                  | ECU       | inv_L_torqueCurrent, inv_L_magnetizingCurrent, inv_L_tempMotor, inv_L_errorInfo                                                                                                                                                                                                                                                 |
| 298          | ECU_InvRight                 | ECU       | inv_R_torqueCurrent, inv_R_magnetizingCurrent, inv_R_tempMotor, inv_R_errorInfo                                                                                                                                                                                                                                                 |
| 299          | ECU_Thermal                  | ECU       | left_coolant_temp, right_coolant_temp                                                                                                                                                                                                                                                                                           |
| 257          | EMMA_StatusMessage           | EMMA      | EMMA_STATUS_OK, EMMA_ERR_UNDERVOLTAGE, EMMA_ERR_OVERVOLTAGE, EMMA_ERR_OVERTEMP_CELL, EMMA_ERR_UNDERTEMP_CELL, EMMA_ERR_OVERTEMP_PCB, EMMA_ERR_UNDERTEMP_PCB, EMMA_ERR_CELL_MISMATCH, EMMA_ERR_BSPD, EMMA_ERR_DAISY_CHAIN, EMMA_IMD_LATCH, EMMA_AMS_LATCH, EMMA_FINISH_PRECHARGE, EMMA_ERR_CRITICAL_FAULT, voltage, current, yaw |
| 264          | EMMA_DisplayData             | EMMA      | state_of_charge, voltage, max_temperature                                                                                                                                                                                                                                                                                       |
| 263          | EMMA_BMSDiagnostics          | EMMA      | devState, ioState, protections                                                                                                                                                                                                                                                                                                  |
| 267          | EMMA_Gyro                    | EMMA      | x, y, z                                                                                                                                                                                                                                                                                                                         |
| 268          | EMMA_Acc                     | EMMA      | x, y, z                                                                                                                                                                                                                                                                                                                         |
| **392**      | **InverterLeftSendMessage**  | **ECU**   | bInverterOn, bDcOn, bEnable, bErrorReset, torqueSetpoint, positiveTorqueLimit, negativeTorqueLimit                                                                                                                                                                                                                              |
| **393**      | **InverterRightSendMessage** | **ECU**   | bInverterOn, bDcOn, bEnable, bErrorReset, torqueSetpoint, positiveTorqueLimit, negativeTorqueLimit                                                                                                                                                                                                                              |
| **647**      | **InverterLeftValues1**      | **INV_L** | RL_ActualVelocity, RL_ActualTorque, RL_MotorTemp, AMK_bSystemReady, AMK_bError, AMK_bWarn, AMK_bQuitDcOn, AMK_bDcOn, AMK_bQuitInverterOn, AMK_bInverterOn, AMK_bDerating                                                                                                                                                        |
| **648**      | **InverterRightValues1**     | **INV_R** | RR_ActualVelocity, RR_ActualTorque, RR_MotorTemp, AMK_bSystemReady, AMK_bError, AMK_bWarn, AMK_bQuitDcOn, AMK_bDcOn, AMK_bQuitInverterOn, AMK_bInverterOn, AMK_bDerating                                                                                                                                                        |
| **649**      | **InverterLeftValues2**      | **INV_L** | TempMotor, TempInverter, ErrorInfo, TempIGBT                                                                                                                                                                                                                                                                                    |
| **656**      | **InverterRightValues2**     | **INV_R** | TempMotor, TempInverter, ErrorInfo, TempIGBT                                                                                                                                                                                                                                                                                    |
| **657**      | **InverterLeftValues3**      | **INV_L** | torqueFeedback, magnetisingCurrentFW, serviceBits                                                                                                                                                                                                                                                                               |
| **658**      | **InverterRightValues3**     | **INV_R** | torqueFeedback, magnetisingCurrentFW, serviceBits                                                                                                                                                                                                                                                                               |
| **659**      | **InverterLeftValues4**      | **INV_L** | DCBusVoltage, DCBusVoltageMonitoring, controllerEnable, temperatureInternal                                                                                                                                                                                                                                                     |
| **660**      | **InverterRightValues4**     | **INV_R** | DCBusVoltage, DCBusVoltageMonitoring, controllerEnable, temperatureInternal                                                                                                                                                                                                                                                     |
| **661**      | **InverterLeftValues5**      | **INV_L** | actualPowerValue, temperatureExternal                                                                                                                                                                                                                                                                                           |
| **662**      | **InverterRightValues5**     | **INV_R** | actualPowerValue, temperatureExternal                                                                                                                                                                                                                                                                                           |

| **ID (Dec)** | **Nome Messaggio**           | **Dimensione Payload (Byte)** |
| ------------ | ---------------------------- | ----------------------------- |
| 288          | ECU_Time                     | 4                             |
| 289          | ECU_WheelVel                 | 8                             |
| 290          | ECU_Pedals                   | 8                             |
| 291          | ECU_SuspSteer                | 8                             |
| 292          | ECU_States                   | 7                             |
| 293          | ECU_Emma                     | 8                             |
| 294          | ECU_Dynamics1                | 8                             |
| 295          | ECU_Torque1                  | 8                             |
| 296          | ECU_Torque2                  | 6                             |
| 297          | ECU_InvLeft                  | 8                             |
| 298          | ECU_InvRight                 | 8                             |
| 299          | ECU_Thermal                  | 4                             |
| 257          | EMMA_StatusMessage           | 8                             |
| 264          | EMMA_DisplayData             | 4                             |
| 263          | EMMA_BMSDiagnostics          | 8                             |
| 267          | EMMA_Gyro                    | 6                             |
| 268          | EMMA_Acc                     | 6                             |
| **392**      | **InverterLeftSendMessage**  | **8**                         |
| **393**      | **InverterRightSendMessage** | **8**                         |
| **647**      | **InverterLeftValues1**      | **8**                         |
| **648**      | **InverterRightValues1**     | **8**                         |
| **649**      | **InverterLeftValues2**      | **8**                         |
| **656**      | **InverterRightValues2**     | **8**                         |
| **657**      | **InverterLeftValues3**      | **8**                         |
| **658**      | **InverterRightValues3**     | **8**                         |
| **659**      | **InverterLeftValues4**      | **8**                         |
| **660**      | **InverterRightValues4**     | **8**                         |
| **661**      | **InverterLeftValues5**      | **6**                         |
| **662**      | **InverterRightValues5**     | **6**                         |
Mentre per comporre il messaggio a 5Hz servono i seguenti:

| **ID (Dec)** | **Nome Messaggio** | **Nodo** | **Segnali Inclusi**                                      |
| ------------ | ------------------ | -------- | -------------------------------------------------------- |
| 258          | EMMA_Voltages1     | EMMA     | module1, module2, module3, module4                       |
| 259          | EMMA_Voltages2     | EMMA     | module5, module6, module7, module8                       |
| 260          | EMMA_Voltages3     | EMMA     | module9, module10, module11, module12                    |
| 261          | EMMA_Voltages4     | EMMA     | module13, module14, connectedModules, IMDInsulationValue |
| 262          | EMMA_Temp          | EMMA     | pcbTemp_0..13, extTemp1_0..13, extTemp2_0..13, id M      |

| **ID (Dec)** | **Nome Messaggio** | **Dimensione Payload (Byte)** |
| ------------ | ------------------ | ----------------------------- |
| 258          | EMMA_Voltages1     | 8                             |
| 259          | EMMA_Voltages2     | 8                             |
| 260          | EMMA_Voltages3     | 8                             |
| 261          | EMMA_Voltages4     | 8                             |
| 262          | EMMA_Temp          | 7                             |

**Banda richiesta totale**

| **Gruppo Messaggi**                          | **Frequenza Rete** | **N° Frame/sec Totali** | **DLC** | **Tempo MAX Singolo Frame** | **Occupazione Banda (Bus Load)** |
| -------------------------------------------- | ------------------ | ----------------------- | ------- | --------------------------- | -------------------------------- |
| **BMS Diagnostics** (ID 263)                 | 5 Hz               | 5                       | 8 Byte  | 270 µs                      | **0.13 %**                       |
| **EMMA Voltages** (ID 258 $\rightarrow$ 261) | 5 Hz               | 20                      | 8 Byte  | 270 µs                      | **0.54 %**                       |
| **EMMA Temp (Multiplex 14x)** (ID 262)       | 5 Hz (70 Hz reali) | 70                      | 7 Byte  | 250 µs                      | **1.75 %**                       |
| **Telemetria 4-Byte** (3 ID)                 | 200 Hz             | 600                     | 4 Byte  | 190 µs                      | **11.40 %**                      |
| **Telemetria 6-Byte** (5 ID)                 | 200 Hz             | 1.000                   | 6 Byte  | 230 µs                      | **23.00 %**                      |
| **Telemetria 7-Byte** (1 ID)                 | 200 Hz             | 200                     | 7 Byte  | 250 µs                      | **5.00 %**                       |
| **Telemetria 8-Byte** (19 ID)                | 200 Hz             | 3.800                   | 8 Byte  | 270 µs                      | **102.60 %**                     |
| **TOTALE COMPLESSIVO**                       |                    | **5.695 frame/s**       |         |                             | **144.42 %**                     |

Come denotato dalla tabella, la linea CAN a 500kbps non può di certo sopportare un carico così grande di dati, per questo ci sono due soluzioni che possono dare una mano:

- **Framerate adattivo** dei vari messaggi basati sulla fisica del sensore oppure della logica che li crea. Ad esempio i controlli di coppia hanno senso se i calcoli vengono fatti a 5 ms, mentre i dati dell'acceleratore già ogni 20ms andrebbero più che bene.
- **CAN FD**: il CAN FD è una speciale architettura della comunicazione CAN che permette di negoziare i bit iniziali a 500kbaud e poi di salire a frequenze anche di 2-5Mbaud, aumentando drasticamente la capacità del canale. In aggiunta possiede la possibilità di inviare payload estesi fino a 64byte, riducendo l'overhead di comunicazione di header e trailer

Implementando queste due soluzioni riusciremo ad aumentare la capacità del canale e contemporaneamente ridurre il carico abbassando la frequenza di log dei dati.

### TCAN4550-Q CAN Controller

Il microcontrollore integrato sul microcontrollore non supporta nativamente il CAN FD, per questo motivo non sara necessario solo un transceiver, ma proprio un controller CAN esterno. La scelta è ricaduta su un **TCAN4550-Q** della Texas Instruments per via la sua elevata affidabilità e perché può essere utilizzato sia con oscillatore dedicato che con clock fornito dal microcontrollore (punto a favore che è certificato automotive). Ad ogni modo si è deciso di optare per un oscillatore esterno da 40Mhz perché è consigliato da datasheet per raggiungere la massima velocità.

L'utilizzo di questo componente implica una linea SPI dedicata.

### ADP3339 LDO Regulator

Il regolatore di tensione è una parte importantissima del progetto, perché mette insieme considerazioni riguardanti l'ambiente di funzionamento e le necessità degli altri componenti sulla scheda. In primo luogo bisogna considerare le emissioni EMI della scheda che monta non una ma ben due antenne ad alte prestazioni su frequenze diverse: l'ESP32 genera un segnale wifi e un segnale bluetooth che hanno il potenziale per generare dei disturbi indotti nelle piste del resto della scheda, aggiungendosi anche al disturbo emesso dall'inverter e dal chip LoRa.
La scelta è ricaduta su un regolatore lineare LDO perché non aggiunge ulteriori disturbi alle alimentazioni rispetto ad un regolatore DC-DC switching e, con qualche accortezza, può non essere un problema dal punto di vista termico.

La corrente di picco del ADP3339 raggiunge 1.5A, il che lascia un margine comodo per l'alimentazione di ESP32, transceiver CAN e MicroSD. Inoltre l'architettura Low Dropout dell'ADP3339 garantisce la stabilizzazione a 3.3V anche se la tensione di ingresso a 5V dovesse subire fluttuazioni o cali dovuti a picchi di assorbimento su altre schede collegate allo stesso cablaggio della vettura, assicurando la continuità della telemetria in ogni condizione di gara.

Una scelta importante è stata quella di dividere l'alimentazione del modulo LoRa per attingere direttamente dall'alimentazione principale a 5V della scheda che arriva dall'esterno. Questo evita all'LDO di farsi carico dei 650mA di picco che viene consumato in trasmissione, lasciando un'alimentazione pulita e senza sbalzi di tensione al microcontrollore.

## Schematico e PCB

In un ambiente duro come quello della Formula Student ogni componente deve essere scelto e posizionato con cura per evitare una serie di problematiche note che possono essere:

- Disturbi elettromagnetici
- Vibrazioni
- Temperature

Nei prossimi capitoli verranno analizzati in dettaglio i blocchi logici che compongono lo schematico e la PCB, motivando le scelte di design.

### Stack-Up e piani

Usando una struttura a 6 layer, il progetto fa un salto qualitativo necessario per supportare la presenza di un trasmettitore LoRa da 1W, un microcontrollore ad alta frequenza e un bus differenziale CAN. Lo stack-up scelto è stato pensato per massimizzare l'integrità dei segnali e abbattere le emissioni irradiate.

| No. | Name     | Type       |
| --- | -------- | ---------- |
| 1   | Top      | Signal     |
| 2   | GND      | Plane Zone |
| 3   | Inner 3  | Signal     |
| 4   | Inner 4  | Signal     |
| 5   | 3V3_MAIN | Plane Zone |
| 6   | Bottom   | Signal     |


La scheda è stratificata secondo la seguente logica:

- **Layer 1 - Top (Signal):** Dedicato ai componenti principali e al routing esclusivo dei segnali differenziali ad alta criticità. Instradare le piste `CANH` e `CANL` rigorosamente su questo strato, in modo diretto e senza mai attraversare la scheda con dei via, evita discontinuità di impedenza e previene alterazioni di fase, garantendo la massima immunità ai disturbi esterni dell'inverter e delle antenne.
    
- **Layer 2 - Inner 1 (GND Plane):** Configurato come Plane Zone dedicata interamente alla massa. Avere un piano di massa continuo subito sotto i segnali ad alta velocità è fondamentale perché fornisce il percorso di ritorno più breve possibile per le correnti ad alta frequenza, abbassando le emissioni elettromagnetiche e proteggendo la scheda dai disturbi esterni dell'inverter. Inoltre agisce come un massiccio dissipatore termico per l'intera scheda.

- **Layer 3-4 - Inner 2-3**: Entrambi i layer servono per semplificare lo sbroglio delle net e garantiscono anche una maggiore schermatura alle linee ad alta velocità che ci passano per via del fatto che sono imbottigliate in due piani continui di rame.
    
- **Layer 5 - Inner 3 (3V3_MAIN Plane):** Questo layer interno è dedicato alla distribuzione della potenza a 3.3V. Utilizzare un intero piano per la potenza (invece di semplici piste) garantisce un'impedenza bassissima, permettendo all'ESP32 e alla MicroSD di assorbire picchi di corrente istantanei senza generare cali di tensione. Inoltre, l'accoppiamento capacitivo naturale che si crea tra questo piano e il piano di massa adiacente funge da enorme condensatore di disaccoppiamento distribuito, filtrando ulteriormente il rumore ad alta frequenza.
    
- **Layer 6 - Bottom (Signal):** Utilizzato per i segnali secondari a bassa velocità e per completare il posizionamento dei componenti ingombranti o critici termicamente come il regolatore LDO. Instradare le piste trasversali su questo layer permette di evitare tagli sul layer Top, preservando la continuità del piano di massa interno.
### Power stage

![[power_stage_schematic.png]]

Il power stage è diviso in due blocchi concettuali, le protezioni per sovratensioni e inversioni di polarità e il regolatore LDO.

Analizzando il primo blocco si può notare che l'inversione di polarità è stata gestita attraverso il diodo schottky D5 che è stato dimensionato per sopportare la somma delle correnti dei singoli moduli, lasciando ovviamente un ampio margine. Non serve essere precisi con le correnti perché si rischierebbe di bruciare il diodo se si eccedono le sue specifiche, inoltre serve solo come protezione verso le inversioni di polarità, non è fatto per diventare un e-fuse.

Il secondo componente importante è il diodo TVS (Transient Voltage Suppressor) bidirezionale D6, che serve per sopprimere i transienti(picchi momentanei) di tensione indotti nell'alimentazione dai disturbi. Il TVS è dimensionato per rimanere inattivo alla tensione nominale, ma interviene rapidamente scaricando a massa i picchi prima che possano friggere il regolatore o, peggio, danneggiare il modulo LoRa (che ha un limite operativo massimo di 5.5V continuativi).

Nel secondo blocco è presente il regolatore che abbassa la tensione dai 5V ai 3V3. La peculiarità di questo regolatore è che la tensione fissa di uscita evita l'aggiunta del partitore di tensione tra l'uscita e il pin di regolazione. Il datasheet indica che per mantenere la tensione in uscita stabile è necessario mettere due condensatori da 1$\micro F$ in ingresso e in uscita. In questo caso nono stati messi da 10 perché in un ambiente rumoroso come il nostro è meglio non rischiare.

Gli ultimi componenti sono i LED di stato, che indicano l'accensione del power stage e dell'alimentazione principale.

![[power_stage_layout.png|400]]

L'immagine sopra mostra il layout implementato per il regolatore. Tutto il power stage è stato posizionato sul layer inferiore della scheda per risparmiare spazio e per ottimizzare la dissipazione termica.

L'alimentazione a 5V arriva dal connettore nella parte inferiore e viene instradata attraverso piste da 0.8mm, passando prima attraverso i componenti D5 e D6 per le protezioni, per poi immettersi nei pin di ingresso del regolatore.

**Gestione Termica e Piani di Rame**

Poiché l'ADP3339 è un regolatore lineare, l'abbassamento di tensione da 5V a 3.3V comporta una dissipazione di energia sotto forma di calore, proporzionale alla corrente assorbita. Per smaltire questo carico termico senza l'ausilio di dissipatori meccanici esterni, il layout sfrutta attivamente i piani di rame del PCB. I pin di massa del regolatore e dei condensatori di filtro sono ancorati a un poligono di rame dedicato. Questo piano superficiale agisce da primo dissipatore, allontanando rapidamente il calore dal package del componente.

**Via Termici e Transizioni di Potenza**

Per massimizzare l'efficienza termica e garantire percorsi a bassissima impedenza per le correnti elevate, il layout fa un uso estensivo di via:

- **Via Termici:** All'interno e in prossimità del piano di rame dell'LDO sono stati piazzati multipli via passanti collegati al piano di massa interno continuo. Questi via fungono da tubi termici, trasferendo il calore dal layer inferiore ai layer interni, permettendo all'intera massa della scheda di funzionare come un grande radiatore.
    
- **Via per Alte Correnti:** Nelle zone in cui l'alimentazione deve necessariamente cambiare layer (ad esempio per portare i 3.3V all'ESP32), è stato evitato l'uso di un singolo via. Per gestire i picchi transitori, le transizioni avvengono tramite cluster di via multipli. Questa ridondanza abbassa la resistenza e l'induttanza parassita del collegamento, prevenendo surriscaldamenti localizzati e cadute di tensione sui rail di alimentazione che causerebbero riavvii improvvisi del microcontrollore.

### Linea CAN

![[can_schematic.png]]

A protezione del controller contro i forti disturbi elettromagnetici dell'inverter, la linea fisica è stata equipaggiata con un array di diodi TVS U3 e un Common Mode Choke L3 con footprint standardizzato, permettendo di testare dinamicamente induttanze diverse. 

Sulla linea 3V3 è stato anche inserito in condensatore di decoupling, che permette di assorbire sbalzi di corrente e alcuni disturbi. L'ultimo elemento importante per far funzionare la linea CAN sono le resistenze di terminazione da 120 ohm, che vengono inserite sulla linea differenziale.

Il componente più sensibile in questo design è indubbiamente il quarzo U7 che genera il clock a 40Mhz richiesto dal CAN FD. Ci sono due parametri che bisogna controllare per scegliere un quarzo ideale:

- La deriva di frequenza deve essere inferiore a 20ppm considerando anche la variazione di deriva nell'arco del tempo
- La capacità dei condensatori deve essere il doppio della capacità di carico contando anche una capacità parassita che va dai 2pf ai 4pf

$$C_{17} /C_{18} = 2C_{l} - C_{parassita}$$

![[can_layout.png|450]]

I segnali CAN_H e CAN_L sono segnali differenziali e, in quanto tali, sono state instradate in modo parallelo e accoppiato per tutta la lunghezza della pista. Il segnale segue precisamente un ordine fisico che è il seguente: Connettore $\rightarrow$ Diodo TVS $\rightarrow$ Common Mode Choke $\rightarrow$ Terminazione Split $\rightarrow$ Transceiver.

Il filtro di modo comune è l'elemento più critico per sopravvivere al rumore dell'inverter. Spesso è impossibile calcolare l'induttanza perfetta, poiché la capacità parassita del cablaggio e dei diodi TVS può innescare circuiti LC risonanti che distorcono il segnale. Per mitigare questo rischio si è optato per una strategia flessibile: è stato previsto un footprint SMD standard 1210 per il Common Mode Choke. Questo permette di testare diversi valori di induttanza (es. 11 µH, 22 µH, 51 µH) o di sostituire il filtro con resistenze da 0 Ohm o jumper qualora si rilevasse una distorsione del segnale dovuta a un'induttanza eccessiva (come già successo su altre schede).

Dallo schematico si può anche notare l'utilizzo dello strato interno Inner 3 per lo sbroglio della linea SPI.

### LoRa

![[lora_schematic.png|300]]![[lora_layout.png|350]]

L'integrazione di un trasmettitore da 500mW (E28-2G4M27SX) direttamente su un circuito compatto pone sfide severe in termini di integrità del segnale e immunità ai disturbi. Un segnale RF ad alta potenza può facilmente accoppiarsi con le piste vicine, corrompendo i dati del bus SPI o inducendo reset anomali nel microcontrollore. Per evitare questo scenario, il layout del modulo LoRa è stato progettato attorno a tre strategie chiave:

- **Schermatura RF e Stitching**: Il modulo integrato che contiene l'antenna LoRa ha al di sopra una schermatura metallica che impedisce l'uscita di radiofrequenze se non dall'antenna. Per riprodurre un effetto simile, sotto il modulo è stata introdotto un piano continuo di rame sul primo layer, collegato al piano di massa da un array di via. Questa struttura crea una vera e propria gabbia di faraday che intrappola al suo interno tutti i segnale che vengono Generati dal modulo.
- **Keepout zone**: Direttamente sotto il connettore dell'antenna è stata instaurata una keepout zone, che impedisce il piazzamento di aree di rame, componenti e piste direttamente sotto la stessa. Se si inserisce un piano di rame, le onde elettromagnetiche si ripiegano direttamente verso GND, il che oltre a generare riflessioni rischia di indebolire il segnale.
- **Power Delivery**: il modulo LoRa è alimentato direttamente dal piano di rame a 3.3V. Tuttavia, trasmettere pacchetti a 500mW richiede picchi di assorbimento transitori molto rapidi, fino a 350mA. Per evitare che l'induttanza parassita delle piste rallentasse l'erogazione di corrente causando cali di tensione, il condensatore di bulk primario C2 è stato posizionato fisicamente a ridosso del pin VCC del modulo. Questo condensatore funge da riserva di energia locale, garantendo la massima potenza senza stressare il cablaggio generale.

Le due piste a sinistra sono i due segnali `LORA_TXEN` e `LORA_RXEN`, che servono a settare la modalità dell'antenna in trasmissione e ricezione. Affinché questi comandi arrivino in modo perfettamente sincrono, è stata applicata la tecnica del length tuning (ricciolo a serpentina sulla pista sotto) per equalizzarne la lunghezza e annullare le differenze di tempo di propagazione. Infine, le connessioni nella parte inferiore costituiscono il bus SPI ad alta velocità, instradato in modo compatto e diretto per trasportare i comandi e i dati tra il modulo LoRa e il microcontrollore.

### Microcontrollore
![[ESP_pinout.png|320]]![[ESP_layout.png|320]]

Al centro del footprint dell'ESP32 è ben visibile il grande pad termico esposto. Invece di affidare la dissipazione alla sola superficie del layer Top, il pad è stato ancorato al piano di massa interno tramite un array di 4 via passanti. Questa soluzione abbassa la resistenza termica del componente, trasferendo il calore generato durante le trasmissioni Wi-Fi o i calcoli intensivi agli strati interni, sfruttando l'intera massa di rame del PCB come radiatore.

Un elemento critico per l'affidabilità di qualsiasi microcontrollore è il power-on reset. Come visibile dallo sbroglio evidenziato in rosso, il pin `ENABLE` dell'ESP32 è controllato da una specifica rete RC, composta dalla resistenza di pull-up R1 e da un condensatore a massa. Questo circuito hardware ritarda l'accensione del microcontrollore di qualche millisecondo rispetto all'erogazione dell'alimentazione. In questo modo si garantisce che l'ESP32 inizi l'esecuzione del firmware solo quando il piano dei 3.3V è perfettamente stabilizzato, prevenendo crash, stati logici indefiniti o scritture corrotte sulla MicroSD all'avvio. Sono stati inoltre integrati due pulsanti tattili BOOT ed ENABLE per consentire il reset manuale e l'ingresso forzato in modalità bootloader per il flashing del firmware durante lo sviluppo del software.

Sui pin di alimentazione sul lato destro del chip, si notano i condensatori di decoupling. La loro  vicinanza ai pin VCC non è casuale: serve a minimizzare l'induttanza parassita delle piste. Questi condensatori agiscono come serbatoi che forniscono energia istantanea alle porte logiche interne dell'ESP32 durante le rapide commutazioni, sopprimendo al contempo il rumore ad alta frequenza prima che possa inquinare il piano di alimentazione 3V3_MAIN.

### MicroSD

![[microsd_layout.png|400]]

A livello meccanico, il socket per la MicroSD è stato collocato strategicamente lungo il perimetro esterno del PCB. Questa scelta non risponde solo a criteri di sbroglio, ma è una precisa esigenza di usabilità in pista: permette l'estrazione rapida della scheda senza dover smontare la centralina. In questo modo è possibile scaricare fisicamente i log della telemetria sul PC qualora lo scaricamento OTA tramite Wi-Fi non fosse utilizzabile.

### RTC

![[rtc_schematic.png|350]]![[rtc_layout.png|300]]

È stato deciso di inserire anche un real time clock per essere in grado di assegnare un timestamp ai pacchetti di telemetria. La particolarità è che si rende necessaria una batteria a bottone aggiuntiva per alimentarlo a macchina spenta

## Algoritmo ADR

L'adozione del modulo E28-2G4M27SX permette di superare il limite della larghezza di banda. Per fare in modo di avere sempre il maggior data rate possibile in qualsiasi condizione è possibile operare su tre parametri fondamentali della rete LoRa:

- **Spreading Factor (SF)**
- **Bandwidth (BW)**
- **Potenza di Trasmissione (TX Power)**
- **Modulazione**

### Parametri fondamentali

È necessario acquisire due metriche fondamentali: il **Margine di Link (LM)** e il **Packet Error Rate (PER)**.

Il mattone base per la scelta del modem e per la transizione tra le diverse modulazioni è la comprensione della distanza tra le prestazioni hardware e il limite teorico della fisica delle telecomunicazioni. In particolare il teorema di Shannon descrive la capacità teorica di canale mettendo in relazione SNR e larghezza di banda.

Guardando il datasheet del chip all'interno del modulo si ottiene questa relazione: 

![[modem_selection.png|300]]

Nel grafico, la linea blu rappresenta il limite di Shannon, ovvero la sensibilità teorica minima assoluta per una comunicazione priva di errori a un dato bit rate. Le altre curve rappresentano le performance hardware reali dei modem integrati. Come si evince, la modulazione LoRa grazie alle tecniche Spread Spectrum si avvicina maggiormente al limite teorico rispetto al GFSK convenzionale, garantendo una superiore tolleranza al rumore e permettendo la demodulazione di segnali con SNR fortemente negativi fino a -22.5 dB.

tuttavia, nella vita reale, il rumore non è costante. I transienti elettromagnetici ad alta frequenza generati dall'inverter di trazione e lo shadowing causato dagli ostacoli abbattono repentinamente l'SNR. Per non rischiare disconnessioni, l'algoritmo ADR non si spinge mai fino al limite teorico consentito dalla modulazione, ma calcola un margine di link (es. 6 dB). Questo cuscinetto di sicurezza garantisce che, anche in caso di un forte picco di rumore EMI condotto, il collegamento rimanga robusto e il packet error rate contenuto.

### Cambi di modulazione e spreading factor

I cambi che l'ADR deve effettuare non sono mai repentini, ma dei gradini più morbidi, che permettano di non portare troppo in basso il margine di link. Per ricavale quale siano questi salti dobbiamo partire dalla modulazione più robusta e basilare, LoRa.
Partiamo da LoRa perché possiede la maggiore immunità al rumore e permette la ricezione corretta dei pacchetti anche con un SNR fortemente negativo.

All'interno della modulazione si può agire sulla regolazione dello spreading factor, che rappresenta il numero di bit di informazione codificati all'interno di un singolo chip. Uno spreading factor più basso permette di modulare il messaggio più rapidamente, quindi sblocca velocità di trasmissione superiori. Normalmente LoRa possiede solo 6 SF da SF7 a SF14, ma noi con questo modulo partiamo da SF5.

![[lora_sf.jpeg]]

La seguente tabella mostra la sensitività in relazione a larghezza di banda e spreading factor

| **Bandwidth [kHz]** | **SF5 [dBm]** | **SF6 [dBm]** | **SF7 [dBm]** | **SF8 [dBm]** | **SF9 [dBm]** | **SF10 [dBm]** | **SF11 [dBm]** | **SF12 [dBm]** |
| ------------------- | ------------- | ------------- | ------------- | ------------- | ------------- | -------------- | -------------- | -------------- |
| **203**             | -109          | -111          | -115          | -118          | -121          | -124           | -127           | -130           |
| **406**             | -107          | -110          | -113          | -116          | -119          | -122           | -125           | ==🟢-128==     |
| **812**             | -105          | -108          | -112          | -115          | -117          | -120           | ==🟢-123==     | -126           |
| **1625**            | ==🟢-99==     | -103          | ==🟢-106==    | -109          | ==🟢-111==    | -114           | -117           | -120           |

Mentre questa mostra il datarate

| **Bandwidth [kHz]** | **SF5 [kb/s]** | **SF6 [kb/s]** | **SF7 [kb/s]** | **SF8 [kb/s]** | **SF9 [kb/s]** | **SF10 [kb/s]** | **SF11 [kb/s]** | **SF12 [kb/s]** |
| ------------------- | -------------- | -------------- | -------------- | -------------- | -------------- | --------------- | --------------- | --------------- |
| **203**             | 31.72          | 19.03          | 11.10          | 6.34           | 3.57           | 1.98            | 1.09            | 0.595           |
| **406**             | 63.44          | 38.06          | 22.20          | 12.69          | 7.14           | 3.96            | 2.18            | ==🟢1.190==     |
| **812**             | 126.88         | 76.13          | 44.41          | 25.38          | 14.27          | 7.93            | ==🟢4.36==      | 2.380           |
| **1625**            | ==🟢253.91==   | 152.34         | ==🟢88.87==    | 50.78          | ==🟢28.56==    | 15.87           | 8.73            | 4.760           |
Le sezioni evidenziate in verde