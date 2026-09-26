# Telemetria 2026 - 2027

**Stasi Samuele - 25/09/2026**

---
## Obiettivo del progetto

L'obiettivo è progettare produrre e programmare una scheda di telemetria che supporti sia acquisizione dati in locale che la telemetria wireless. La motivazioni che ci hanno spinto ad iniziare il presente progetto è la necessità di avere un supporto tecnologico più robusto, sia dal punto di vista elettrico che dal punto di vista della trasmissione dati.

La scheda utilizzerà una trasmissione LoRa a 868MHz ad alta potenza per garantire un'adeguata portata del segnale e una relativamente alta sopportazione dei disturbi elettromagnetici. A supporto di future necessità verrà scelto un modulo di trasmissione che sia in grado di trasmettere anche segnali in modalità FSC, che gli sbloccano una capacità di canale molto più ampia se ci si trova entro una data distanza. maggiori informazioni verranno fornite nel prossimo capitolo.

Per la persistenza in locale si è optato per una semplice scheda sd.

I dati verranno ricevuti dalla scheda attraverso una linea CAN 2.0A/2.0B

## Componenti di base

### **E22-900M30S - SX1262 868/915MHz Wireless Module**

Questo modulo è un'evoluzione dei prodotti precedenti e che contiene un modulo avanzato per la comunicazione LoRa, le principali caratteristiche sono le seguenti:

- La distanza di comunicazione arriva fino a 12Km
- Potenza di trasmissione massima 1W, modificabile via software
- Supporto per la banda globale ISM 868/915MHz
- Supporto per un data-rate che varia da 0.018-62.5kbps in modalità LoRa
- Supporto fino a 300kpbs in FSK mode
- FIFO di 256Byte per la cache
- Supporto per alimentazione da 2.5V~5.5V, 5V+ garantiscono la massima performance
- Supporto antenna IPEX

Da questi dati risulta palese il motivo della scelta: anche se la modalità FSK non è disponibile su distanze elevate, il chip LoRa può continuare ad operare con minimi cali di segnale per lunghi tratti e sicuramente su tutto il tracciato di gara o di test.
Inoltre la banda lo rende la scelta ideale perché è l'unica che può essere utilizzata quasi in tutto il mondo anche in vista di ripetute tappe estere.

La sua alimentazione varia dai 3.3V ai 5.5V, garantendo il funzionamento durante cali di voltaggio e in caso ci fosse la necessità di farlo lavorare con un minor consumo di energia (ovviamente al costo di restringere il range di trasmissione). Comunica con il resto della scheda attraverso una linea SPI a 3.3V, permettendo di essere controllato semplicemente da qualsiasi microcontrollore mainstream sul mercato.
Il consumo di corrente varia dai 14mA in sola ricezione ai 650mA istantanei in trasmissione.

![[E22-900M30S_footprint.png|301]]![[E22-900M30S_footprint_2.png]]

### **ESP32-S3-WROOM-1U** (N8R2)

Come microcontrollore è stato scelto un ESP32-S3-WROOM-1U nella sua versione da 8MB di flash e 2MB di PSRAM. La dicitura 1U sta ad intendere che non è presente l'antenna sulla PCB del modulo, ma un connettore per antenna esterna. Sia questo modulo che il modulo LoRa presentano la stessa soluzione per antenna esterna per il semplice fatto che un body costruito in materiale composito, soprattutto fibra di carbonio, essendo conduttivo potrebbe totalmente o parzialmente bloccare le emissioni elettromagnetiche. L'antenna esterna permette un superiore guadagno essendo in linea d'aria con il ricevitore.

Sebbene la telemetria a lungo raggio sia affidata al modulo LoRa, l'antenna dell'ESP32-S3 sblocca le funzionalità radio native del chip con Wi-Fi 4 a 2.4 GHz e Bluetooth 5.0 Low Energy. Questa scelta architetturale apre a due scenari fondamentali per l'operatività in pista:

- **Scaricamento dati e OTA:** Il Wi-Fi permette di scaricare i log pesanti ad alta velocità quando la vettura rientra ai box, o di eseguire aggiornamenti firmware OTA senza dover smontare la scocca per accedere fisicamente alla porta USB della centralina.

- **Diagnostica rapida:** Il Bluetooth LE può essere utilizzato per connettere rapidamente un tablet o uno smartphone per visualizzare lo stato della vettura, gli errori o i valori dei sensori durante le fasi di ispezione o di setup pre gara.

**Matrice GPIO e Flessibilità di Routing**

Un fattore decisivo nella scelta della famiglia ESP32-S3 è l'avanzata matrice GPIO. A differenza dei microcontrollori tradizionali dove le periferiche hardware sono legate a pin fissi, l'ESP32-S3 permette di instradare quasi ogni segnale digitale (SPI, UART, I2C, PWM) su qualsiasi pin libero. Questa caratteristica ha semplificato drasticamente lo sbroglio (routing) del PCB: ha permesso di sbrogliare le piste del bus SPI verso la MicroSD e le linee di comunicazione verso il modulo LoRa in modo diretto e parallelo, riducendo al minimo l'uso di via passanti e migliorando l'integrità dei segnali.

![[ESP32_pin_layout.png|300]]![[ESP32_pinout.png|350]]

Le immagini sopra mostrano il layout fisico del modulo ESP32-S3 e la corrente congirurazione di pinout scelta per ottimizzare lo sbroglio in fase di progettazione della PCB.

