# Telemetry Node (ESP32)
Il nodo della telemetria, pur essendo un nodo secondario di tutta l'architettura, rappresenta il punto in cui i dati provenienti da tutta la macchina vengono aggregati per essere utilizzati in un secondo momento per simulazioni o analisi di guasti di sistema. È importante perché contiene anche i file di configurazione di alcune funzionalità della centralina che verranno recuperati all'accensione.

## Specifiche hardware

- **Scheda:** Arduino Nano ESP32.
- **Periferiche collegate:**
    - Modulo MicroSD SPI.
    - Connessione UART verso la VCU (TX, RX, GND).
- **Obiettivo:** Ascoltare la seriale, salvare i dati su file CSV e gestire l'invio dei file di setup alla centralina all'accensione.

## Architettura del software

Il software non deve fare calcoli, è puramente un gestore di dati che deve essere ottimizzato per essere velocissimo e un gestore di file. Deve implementare una macchina a stati solida per non bloccarsi se la SD va in errore o se vede dei dati corrotti sulla linea UART.

### Macchina a stati

La macchina a stati viene implementata usando uno switch (vedi codice centralina per gli inverter).
#### Stato 1: INIT
In questo stato bisogna inizializzare tutte le connessioni:

- La centralina necessita di una UART configurata per girare a 500 KBaud/s per non rappresentare un collo di bottiglia né per l'ECU né per la stessa telemetria
- Inizializzazione della linea SPI per la connessione dati con la scheda SD

#### Stato 2: SYNC_SETUP
In questo stato bisogna leggere il file di configurazione `setup.csv` per ricavare i dati da mandare alla centralina che poi si sincronizzerà con lo schermino. I dati avranno la seguente struttura:

| Type     | Name                 | Description                                                                                                                                                                     |
| -------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| uint8_t  | enables              | Bit field che contiene gli enable dei componenti della vettura, come ad esempio il traction control e il torque vectoring. In caso siano tutti abilitati il dato sarà: `000011` |
| uint8_t  | endurance_max_torque | Massima coppia erogabile durante l'endurance, in Nm (max 21)                                                                                                                    |
| uint16_t | torque_vectoring_Kp  | Valore della costante di proporzionalità del controllore PID del torque vectoring                                                                                               |
| uint16_t | torque_vectoring_Ki  | Valore della costante di integrazione del controllore PID del torque vectoring                                                                                                  |
| uint16_t | torque_vectoring_Kd  | Valore della costante di derivazione del controllore PID del torque vectoring                                                                                                   |
| uint16_t | traction_control_Kp  | Valore della costante di proporzionalità del controllore PID del traction control                                                                                               |
| uint16_t | traction_control_Ki  | Valore della costante di integrazione del controllore PID del traction control                                                                                                  |
| uint16_t | traction_control_Kd  | Valore della costante di derivazione del controllore PID del traction control                                                                                                   |

_I k dei PID sono in millesimi_

| bit0                         |
| ---------------------------- |
| ECU_TRACTION_CONTROL_ENABLED |
| **bit1**                     |
| ECU_TORQUE_VECTORING_ENABLED |
| **bit2-bit7**                |
| reserved                     |

I file CSV avranno il delimitatore che ti sembra più appropriato. Ogni riga rappresenta una modifica di uno dei parametri e deve essere riempita con il resto dei dati che non sono cambiati.

```c
typedef enum {
  ECU_TRACTION_CONTROL_ENABLED = 0x01, // 0b00000001
  ECU_TORQUE_VECTORING_ENABLED = 0x02  // 0b00000010
} ECU_FunctionEnables_mask_t;

// Base PID structure definition
typedef struct __attribute__((packed)) {
  uint16_t kp;
  uint16_t ki;
  uint16_t kd;
} PID_Parameters_t;

// Main payload structure matching the dashboard table
typedef struct __attribute__((packed)) {
  uint8_t enables;              // Bit field using ECU_FunctionEnables_mask_t
  uint8_t endurance_max_torque; // Max torque in Nm (max 21)
  PID_Parameters_t torque_vectoring;
  PID_Parameters_t traction_control;
} ECU_SetupData_t;
```

Se per qualche arcano motivo non riesci a usare gli struct dei pid puoi anche usare uno struct piatto contenente tutto:

```c
// Flat payload structure alternative
typedef struct __attribute__((packed)) {
  uint8_t enables;              // Bit field using ECU_FunctionEnables_mask_t
  uint8_t endurance_max_torque; // Max torque in Nm (max 21)
  uint16_t torque_vectoring_Kp;
  uint16_t torque_vectoring_Ki;
  uint16_t torque_vectoring_Kd;
  uint16_t traction_control_Kp;
  uint16_t traction_control_Ki;
  uint16_t traction_control_Kd;
} ECU_SetupDataFlat_t;
```

Una volta ottenuti i dati dalla SD bisogna mandarli in loop alla centralina ogni 10 ms finchè non viene ricevuto un ack, che è semplicemente un messaggio sulla UART contenente i byte `0xFF 0xAC 0xC0`.

Il file contenente di dati di telemetria deve essere creato in questa fase, identificandolo con un numero incrementale.
#### Stato 3: LOGGING_MODE
In questo stato bisogna ascoltare all'infinito, se arriva un pacchetto di telemetria bisogna metterlo direttamente dentro al file che viene creato al punto precedente in formato binario. NON CONVERTIRE LA TELEMETRIA IN UN CSV OPPURE ESPLODE TUTTO. Se invece arriva un pacchetto di configurazione va salvato nel file di configurazione convertendolo in csv.

```c
typedef struct __attribute__((packed)) {
// --- Telemetry packet id and metadata ---
uint8_t startByte1; // 0xCC
uint8_t startByte2; // 0x11
uint32_t timestamp;
// --- Wheel velocities ---
float front_left_velocity;
float front_right_velocity;
float rear_left_velocity;
float rear_right_velocity;
// --- Suspension length ---
float front_left_suspension;
float front_right_suspension;
// --- APPS ---
uint16_t accelerator1;
uint16_t accelerator2;
float accelerator_mapped;
// --- Brake ---
uint16_t brake1;
uint16_t brake2;
// --- Steer ---
float steer;
// --- Digital in ---
uint8_t sdc;
uint8_t ready_to_drive_button;
uint8_t ecu_reset_button;
uint8_t tractive_system_on_button;
// --- Battery (EMMA) ---
float emma_current;
float emma_voltage;
float emma_yaw;
uint16_t emma_error;
// --- Vehicle Dynamics & Powertrain Targets ---
float mean_velocity;
float real_yaw_rate;
float total_torque_request;
float torque_tv_L;
float torque_tv_R;
float slip_L;
float slip_R;
float torque_reduction_L;
float torque_reduction_R;
float final_torque_target_L;
float final_torque_target_R;
// --- Inverter Internal Data ---
int16_t inv_L_torqueCurrent;
int16_t inv_L_magnetizingCurrent;
int16_t inv_L_tempMotor;
int16_t inv_R_torqueCurrent;
int16_t inv_R_magnetizingCurrent;
int16_t inv_R_tempMotor;
// --- External Thermal Data ---
float left_motor_temp;
float right_motor_temp;
float left_coolant_temp;
float right_coolant_temp;
// --- State machines ---
uint8_t left_inverter_fsm;
uint8_t right_inverter_fsm;
uint8_t tractive_system_fsm;
// --- ECU mode ---
uint8_t ECU_Mode;
// --- CRC for corrupted packages ---
uint16_t crc16;
} TelemetryPacket_t;

typedef struct __attribute((packed)){
// --- Telemetry packet id and metadata ---
uint8_t startByte1; // 0xEE
uint8_t startByte2; // 0x11
uint32_t timestamp;
// --- Inverter states ---
uint8_t state_of_charge;
uint8_t module_voltage[14];
uint16_t module_temperatures[28];
// --- CRC for corrupted packages ---
uint16_t crc16;
} TelemetryPacket_EMMA_t;
```

### ATTENZIONE
Letture e scritture su sd sono imprevedibili, possono metterci pochi millisecondi come 200ms, il mio consiglio è quello di aumentare il buffer standard della UART e di creare dei buffer che aggregano i dati per un certo periodo di tempo.