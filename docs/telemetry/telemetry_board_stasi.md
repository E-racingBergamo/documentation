# Telemetria 2026 - 2027

**Stasi Samuele - 25/09/2026**

---
## Obiettivo del progetto

L'obiettivo è progettare produrre e programmare una scheda di telemetria che supporti sia acquisizione dati in locale che la telemetria wireless. La motivazioni che ci hanno spinto ad iniziare il presente progetto è la necessità di avere un supporto tecnologico più robusto, sia dal punto di vista elettrico che dal punto di vista della trasmissione dati.

La scheda utilizzerà una trasmissione LoRa a 868MHz ad alta potenza per garantire un'adeguata portata del segnale e una relativamente alta sopportazione dei disturbi elettromagnetici. A supporto di future necessità verrà scelto un modulo di trasmissione che sia in grado di trasmettere anche segnali in modalità FSC, che gli sbloccano una capacità di canale molto più ampia se ci si trova entro una data distanza. maggiori informazioni verranno fornite nel prossimo capitolo.

Per la persistenza in locale si è optato per una semplice scheda sd.

I dati verranno ricevuti dalla scheda attraverso una linea CAN 2.0A/2.0B

## Componenti di base

??? Info "Riguardi"
	 Questo progetto tiene conto di possibili sviluppi futuri, avere un occhio di riguardo per queste cose permette di avere un progetto molto solido e di risparmiare tempo e soldi per eventuali modifiche

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

Le immagini sopra mostrano il layout fisico del modulo ESP32-S3 e la corrente configurazione di pinout scelta per ottimizzare lo sbroglio in fase di progettazione della PCB.

**Controller CAN nativo (TWAI)**

Il microcontrollore integra nativamente un controller TWAI (Two-Wire Automotive Interface), pienamente compatibile con le specifiche CAN 2.0A e 2.0B. Grazie a questo hardware interno, la scheda non necessita di controller CAN esterni, ma richiede unicamente un transceiver fisico che è stato identificato essere il componente L9616 per interfacciarsi ai livelli di tensione del bus della vettura, riducendo i punti di fallimento, la latenza di lettura e l'ingombro sul PCB.

Molto probabilmente su questa linea verrà utilizzato il CAN 2.0B per garantire il massimo flusso di dati con una minima quantità di bit di servizio come i bit di id o terminazione. Non necessario sulle altre linee perché non c'è la necessità di avere tanti dati in sequenza quanto avere il dato giusto al momento giusto.

**Consumi**

A livello energetico, l'ESP32-S3 richiede una gestione termica adeguata implementata tramite pad termico e vie di dissipazione sul PCB. I consumi variano in base allo stato:

- **Deep Sleep:** ~7 µA
- **Elaborazione attiva (Radio Off):** ~20-30 mA
- **Picco di trasmissione Wi-Fi:** Fino a ~350 mA

### L9616 CAN Transceiver

Per il funzionamento della scheda si è resa necessaria la scelta di un transceiver CAN che fosse affidabile e che permettesse di gestire il data rate in modo dinamico, in questo caso il modello L9616 prodotto da ST Microelectronics.

I consumi di corrente sono ridotti, con picchi di massimo 80mA durante la fase di trasmissione di un bit dominante. La sua tolleranza elettrica lo rende anche ottimo per future implementazioni su soluzioni che utilizzano il CAN a 5V, grazie a un'ampia flessibilità dei rail di alimentazione che può sopportare transitori dai -0.3V ai +7V (come da specifiche assolute).

A protezione del transceiver contro i forti disturbi elettromagnetici dell'inverter, la linea fisica è stata equipaggiata con un array di diodi TVS e un Common Mode Choke con footprint standardizzato, permettendo di testare dinamicamente induttanze diverse.

### ADP3339 LDO Regulator

Il regolatore di tensione è una parte importantissima del progetto, perché mette insieme considerazioni riguardanti l'ambiente di funzionamento e le necessità degli altri componenti sulla scheda. In primo luogo bisogna considerare le emissioni EMI della scheda che monta non una ma ben due antenne ad alte prestazioni su frequenze diverse: l'ESP32 genera un segnale wifi e un segnale bluetooth che hanno il potenziale per generare dei disturbi indotti nelle piste del resto della scheda, aggiungendosi anche al disturbo emesso dall'inverter e dal chip LoRa.
La scelta è ricaduta su un regolatore lineare LDO perché non aggiunge ulteriori disturbi alle alimentazioni rispetto ad un regolatore DC-DC switching e, con qualche accortezza, può non essere un problema dal punto di vista termico.

La corrente di picco del ADP3339 raggiunge 1.5A, il che lascia un margine comodo per l'alimentazione di ESP32, transceiver CAN e MicroSD. Inoltre l'architettura Low Dropout dell'ADP3339 garantisce la stabilizzazione a 3.3V anche se la tensione di ingresso a 5V dovesse subire fluttuazioni o cali dovuti a picchi di assorbimento su altre centraline collegate allo stesso cablaggio della vettura, assicurando la continuità della telemetria in ogni condizione di gara.

Una scelta importante è stata quella di dividere l'alimentazione del modulo LoRa per attingere direttamente dall'alimentazione principale a 5V della scheda che attiva dall'esterno. Questo evita all'LDO di farsi carico dei 650mA di picco che viene consumato in trasmissione, lasciando un'alimentazione pulita e senza sbalzi di tensione al microcontrollore.

## Schematico e PCB

In un ambiente duro come quello della Formula Student ogni componente deve essere scelto e posizionato con cura per evitare una serie di problematiche note che possono essere:

- Disturbi elettromagnetici
- Vibrazioni
- Temperature

Nei prossimi capitoli verranno analizzati in dettaglio i blocchi logici che compongono lo schematico e la PCB, motivando le scelte di design.

### Power stage

![[power_stage_schematic.png]]

Il power stage è diviso in due blocchi concettuali, le protezioni per sovratensioni e inversioni di polarità e il regolatore LDO.

Analizzando il primo blocco si può notare che l'inversione di polarità è stata gestita attraverso il diodo schottky D5 che è stato dimensionato per sopportare la somma delle correnti dei singoli moduli, lasciando ovviamente un ampio margine. Non serve essere precisi con le correnti perché si rischierebbe di bruciare il diodo se si eccedono le sue specifiche, inoltre serve solo come protezione verso le inversioni di polarità, non è fatto per diventare un e-fuse.

Il secondo componente importante è il diodo TVS (Transient Voltage Suppressor) bidirezionale D6, che serve per sopprimere i transienti(picchi momentanei) di tensione indotti nell'alimentazione dai disturbi e a garantire una tensione massima di 5V ai suoi capi. In questo caso è necessario essere precisi perché una sovratensione potrebbe friggere il regolatore e danneggiare il modulo LoRa (max 5.5V continuativi). 