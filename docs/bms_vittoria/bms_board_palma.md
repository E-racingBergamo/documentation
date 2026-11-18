# BMS Vittoria 2026/2027

**Fabrizio Palma - 03/11/2026**
**Nominativo progetto - Vittoria**

---
## Obiettivo del progetto

In questi anni di E racing Bergamo, abbiamo adottato dei BMS già prodotti, i Tiny AFE v1.1 di ENEPAQ. Durante i test pre-gara svolti nell'estate 2026, abbiamo notato una grande sofferenza ai disturbi emessi dall'inverter, in fase di alimentazione dei motori. In particolare, non riuscivamo a leggere i dati dei BMS quando la macchina è in funzione. Dopo tentativi di migliorare la schermatura, utilizzo di ferriti, non siamo riusciti a risolvere il problema. 

L'obiettivo postoci per la stagione 2026/2027 è quella di trovare una soluzione a questo problema, anche cambiando tecnologia dei BMS. Dopo esserci consultati anche con altri team di Formula student, abbiamo notato che l'utilizzo di un componente di Analog Device con comunicazione isoSPI ha risolto il problema. 
Dopo aver consultato datasheet di vari componenti della stessa famiglia di ADI, abbiamo deciso di adottare l'ADBMS6815.
### ==🟡Informazioni principali TSAC ERB03==

| Parametro                   | Valore  |
| --------------------------- | ------- |
| Tensione max modulo         | 42V     |
| Celle monitorate per modulo | 10      |
| Temperature monitorate      | 5       |
| Modello celle               | VTC6 4p |
| Totale moduli               | 14      |
| Tensione max TSAC           | 588V    |
*Caratteristiche celle:*
![[Caratteristiche Celle.png]]
### ==🟡Informazioni principali ADBMS6815==

L'ADBMS6815 è un componente che permette 
1. monitoraggio di 12 celle 
2. Comunicazione isoSPI a bassa suscettibilità ai disturbi EMI
3. bilanciamento interno passivo delle celle fino a 300mA (possibilità di aggiungere un circuito esterno per bilanciamento oltre i 300mA) con PWM programmabile
4. 7GPIO o input analogici (utilizzati per misurare le temperature sulle celle)
5. Sleep state che consuma 5uA

Tra le applicazioni in cui viene utilizzato, quella che interessa a noi è "Electric and hybrid electric vehicles".

L'idea per la nostra applicazione è quella di misurare 10 celle, seguendo la configurazione dei moduli attuali, con una tensione massima di 42V (4,2Vx10) e 5 sensori di temperatura (da regolamento bisogna misurare almeno il 30% delle temperature delle celle. Con 5 siamo al 50%, e usiamo due GPIO in meno di quelli disponibili, scelta che verrà motivata successivamente)

Altre features utili:
1. È possibile collegare in serie più dispositivi **ADBMS6815**, consentendo il monitoraggio simultaneo delle celle in lunghe stringhe di batterie ad alta tensione
2. Più dispositivi vengono collegati in una configurazione a cascata (_daisy chain_) con una singola connessione al processore host
3. La comunicazione può avvenire in entrambe le direzioni, garantendo l'integrità dei dati anche in caso di guasto lungo il percorso di comunicazione.
![[ADBMS6815 schema generale.png|300]](schema generale)


### ==🟡Il componente==

![[ADBMS6815 componente.png|300]]
I pin:
1) **V+** è il pin a cui si collega il positivo della serie di celle (dove avremo al max 42V)  
2) Tutti i pin **Cx** sono i pin a cui si collega il positivo delle celle, in mezzo c'è il circuito di filtro
3) Tutti i pin **Sx** sono i pin a cui si collega la resistenza di scarica per il bilanciamento delle celle, al suo interno c'è un mosfet che collega il pin $S_X$ e $S_{x-1}$  per la scarica delle celle
4) I **GPIO(1-7)** possono essere usati come input analogici/digitali dove collegheremo i sensori di temperatura
5) **V-** sono due pin dove si collega il negativo di tutta la serie di celle, fa da GND
6) **Drive** è il pin che fornisce 5.7V per pilotare un transistor che regoli la tensione per alimentare la Vreg
7) **ISOMD** è un digital input. Mandandolo alto configuriamo il BMS per la comunicazione isoSPI (2 cavi), mandandolo basso invece lo configura per la comunicazione SPI (4 cavi)
8) $V_{REF1}$ e $V_{REF2}$ forniscono una tensione di riferimento. Entrambi hanno bisogno del condensatore di bypass. Solo la $V_{REF2}$ possiamo usarla per fornire tensione di riferimento ai sensori di temperatura
9) **ISOTHR** 
### ==🟡Alimentazione del componente==

Il pin primario per fornire alimentazione al componente è il Vreg (input pin a 5V).

|       | min  | typ | max |     |
| ----- | ---- | --- | --- | --- |
| Vreg  | 4.5  | 5   | 5.5 | V   |
Il datasheet dice che ci sono 2/3 modi di fornire alimentazione al pin Vreg:
1) **Simple linear regulator** : con un tansistor BJT NPN si sfrutta l'uscita del pin DRIVE per generare i 5V sull'emettitore a cui si collega il pin di Vreg
2) **External regulator for Improved efficiency** : utilizzare un regolatore esterno per aumentare l'efficienza, utilizzando un regolatore step down

La nostra scelta è ricaduta su un semplice regolatore lineare, per non aumentare rischi di errori e analizzando le foto dei test termici condivisi da Analog, si nota che non c'è una dissipazione critica.

Il **regolatore lineare discreto** con un transistor NPN in configurazione **emitter follower** (collettore comune) è pilotato sulla base dal pin Drive che fornisce:

|       | min  | typ | max |     |
| ----- | ---- | --- | --- | --- |
| Drive | 5.6  | 5.7 | 5.8 | V   |
|       | -0.2 |     | 1   | mA  |
Il nostro circuito:
![[Vreg.png|300]]
Seguendo il circuito suggerito dal datasheet:
1) R36 è una resistenza di disaccoppiamento delle RF  
   $P_{R_{36}}​=I^2R_{36}=0.03^2⋅100=0.09W$  (scelta una 0603 1/10W)
2) R37 è una resistenza che protegge il Q1 da rumore e ESD (ElectroStatic Discharge). Si collega direttamente al positivo della serie di celle
3) Il pin Drive fornisce 5.7V, quindi per la configurazione del BJT abbiamo: 
   $V_{out}=V_{DRIVE}-V_{be}=5.7V-0.7V=5V$ che è la tensione all'uscita del transistor (tra emettitore e GND)
4) Il BJT scelto è un BCP56-16 che supporta 80V 1A con $h_{fe}=100$ tipica
5) C42 e C41 sono condensatori di disaccoppiamento delle RF in ingresso (valori suggeriti da datasheet)

Il datasheet suggerisce di aggiungere anche un L1 che è una **ferrite bead**. Il BMS misura tensioni molto piccole e i distrubi sono un problema. I rumori che possono finire sulla Vreg possono provenire da:
- isoSPI,
- ADC interni,
- commutazioni digitali,
- cavi lunghi,

L'**utilità della ferrite bead** è di mantenere:
- VREG più pulita;
- minore rumore sull'ADC;
- migliore precisione di misura;

La ferrite insieme al condensatore da C43=1 µF crea un filtro passa-basso. 

| A bassa frequenza (corrente continua):                                               | Ad alta frequenza:                                                                                                                                                     |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| - la bead ha una resistenza piccolissima;<br>- il circuito riceve quasi tutti i 5 V. | - la bead presenta un'impedenza elevata;<br>- i disturbi provenienti dal transistor o dall'esterno vengono attenuati;<br>- il condensatore da 1 µF li scarica a massa; |
Quindi alla fine la progettazione di questa parte di circuito viene guidata dal datasheet del componente.

### ==🟡Filtraggio e bilanciamento di carica==
![[F&S basso.png|300]]
Tranne per il pin C0, che si collega al V- della serie di celle, tutti i pin **Cx** misurano la tensione sulla cella. In mezzo c'è un filtro passa basso RC prima che filtra i rumori prima del pin collegato all'ADC interno. I valori di resistenza ($470\Omega$) e del condensatore ($2200nF$) sono consigliati dal datasheet, che suggerisce di tenere valori  $\le470\Omega$  e  $\ge2200nF$  per minimizzare l'errore di misura.
La resistenza da $200\Omega$ è la resistenza che protegge il componente dai picchi di tensione.
Tra ogni pin **Sx** si mette un condensatore da $220nF$ $16V$ per filtrare i rumori tra i pin **Sx**.
La resistenza da $15\Omega$ è la resistenza di scarica per il bilanciamento delle celle e si collega al pin **Sx**. Tale resistenza è calcolata affinché sia all'interno del valore massimo di 300mA sostenibili dal componente, e si calcola come:
$$R_{scarica}=\frac{\frac{V_{cell}} {I_{scarica}}-R_{ds,on}} {2}$$
dove: 
- $R_{ds,on}$ è la resistenza del mosfet interno del componente ($3\Omega$)
- $V_{cell}$ è la tensione delle celle
- $I_{cell}$ è la corrente di scarica (per ora 132mA, come evaluation board vogliamo testare).
Quindi la potenza :$$P_{scarica}= I^2R= (0.132mA)^2*15\Omega= 0.261W$$
motiva la scelta di una resistenza di dimensione 2512 da 1W, sfruttabile se si aumenta la corrente di scarica. 

Nell'immagine successiva, la parte di circuito in cui vengono collegate le celle 9 e 10 e come si collegano i pin del componente $C_{11}$ e $C_{12}$ inutilizzati.
![[fine bilanciamento.png|300]]

### ==🟡Temperature sensing==

#### Monitoraggio della temperatura delle celle
Il regolamento Formula Student richiede il monitoraggio di almeno il 30% delle temperature delle celle presenti nel TSAC. Considerando che ciascun modulo è composto da 10 gruppi di celle in serie, sono stati installati **5 sensori di temperatura per modulo**, consentendo il monitoraggio del 50% delle celle e aumentando la probabilità di individuare tempestivamente eventuali punti caldi (_hot-spots_) o condizioni anomale di funzionamento.

Per mantenere la compatibilità con l'architettura attualmente utilizzata dal team, è stato scelto lo stesso sensore di temperatura impiegato nei moduli esistenti. Il dispositivo è un **temperature-variable voltage shunt**, ovvero un sensore che genera una tensione dipendente dalla temperatura. A differenza dei comuni termistori NTC, il parametro misurato non è la resistenza ma direttamente la tensione ai terminali del sensore.

![[Tabella Sensing.png|350]]
#### Caratteristica tensione-temperatura

Il sensore presenta una caratteristica non lineare nel range operativo compreso tra **−40°C e +120°C**. All'aumentare della temperatura la tensione misurata diminuisce secondo la curva fornita dal costruttore.

Alcuni punti caratteristici sono riportati in Tabella:![[Tabella valori sensing.png]]
![[Grafico sensing.png|350]]
La conversione della tensione misurata in temperatura verrà eseguita dal microcontrollore del sistema utilizzando la tabella fornita dal costruttore e un'eventuale interpolazione lineare tra i punti disponibili.
#### Collegamento all'ADBMS6815

L'ADBMS6815 dispone di sette GPIO configurabili come ingressi analogici. Per il monitoraggio delle temperature vengono utilizzati cinque canali:
- GPIO1 → Temp1
- GPIO2 → Temp2
- GPIO3 → Temp3
- GPIO4 → Temp4
- GPIO5 → Temp5

Ogni sensore è collegato all'uscita di riferimento **VREF2** attraverso una resistenza da **1.8 kΩ**. Tale resistenza determina la corrente che attraversa il sensore e ne consente il corretto funzionamento. 
Il pin di $V_{ref2}$ può fornire al massimo 5mA, assumendo:
- **VREF2 = 3.0 V** (valore tipico dell'ADBMS6815)
- **R = 1.8 kΩ**
- il sensore si comporta come uno shunt di tensione e genera le tensioni riportate nella sua tabella
la corrente che attraversa il sensore vale:
$$I_{sens}=\frac{V_{REF2}-V_{sensore}}{1.8k\Omega}​$$
Calcolando i valori:

| Temperatura [°C] | $V_{sensore}$ | Corrente [mA] |
| ---------------- | ------------- | ------------- |
| -40              | 2.44          | 0.311         |
| 0                | 2.17          | 0.461         |
| 25               | 1.86          | 0.667         |
| 60               | 1.51          | 0.828         |
| 120              | 1.3           | 0.944         |
Il valore scelto verrà validato durante la fase di testing del sistema.
Con **5 sensori** sul modulo:
$I_{tot,max}=5\cdot 0.944 = 4.72mA$
Rimaniamo ancora sotto di circa 0.28mA ossia circa il **5,6% di margine**.

Tra il sensore e l'ingresso GPIO è inoltre presente una resistenza serie da **1 kΩ**, mentre verso massa è collegato un condensatore da **100 nF**. Questi componenti realizzano un filtro passa-basso che riduce il rumore ad alta frequenza acquisito dai cavi dei sensori e migliora la stabilità della misura effettuata dall'ADC interno dell'ADBMS6815.
#### Validazione sperimentale
Poiché il sistema di acquisizione non è ancora stato testato con il componente definitivo, la correttezza delle misure verrà verificata durante la fase di validazione della scheda.
Le prove previste comprendono:
1. riscaldamento controllato delle celle;
2. misura simultanea della temperatura mediante termocoppia di riferimento e sistema BMS;
3. confronto tra le misure ottenute;
4. valutazione dell'errore massimo nel range operativo;
5. verifica della robustezza delle misure in presenza dei disturbi elettromagnetici generati dall'inverter.

L'obiettivo è verificare che la catena di misura composta da sensore, circuito di condizionamento e ADC interno dell'ADBMS6815 fornisca una precisione adeguata ai requisiti di monitoraggio e sicurezza del pacco batterie.

### ==🟡Comunicazione isoSPI==
#### Architettura di comunicazione

Per la comunicazione tra i moduli di monitoraggio e il controllore principale è stata adottata l'interfaccia **isoSPI** integrata nell'ADBMS6815. Questa tecnologia è stata sviluppata da Analog Devices per applicazioni Battery Management System ad alta tensione e consente di realizzare collegamenti robusti in ambienti caratterizzati da elevato rumore elettromagnetico.
Ogni ADBMS6815 dispone di due porte isoSPI indipendenti:
- **Port A**
- **Port B**
Le due porte consentono la realizzazione di una catena di comunicazione (_daisy chain_) tra i diversi moduli del pacco batteria. In questa configurazione la Port B di un modulo è collegata alla Port A del modulo successivo fino a raggiungere il monitor di batteria ADBMS6822, responsabile dell'interfacciamento con il microcontrollore principale.

#### Principio di funzionamento

L'interfaccia isoSPI utilizza una trasmissione differenziale a due fili. Le informazioni vengono trasmesse mediante impulsi differenziali che risultano intrinsecamente meno sensibili ai disturbi elettromagnetici rispetto a collegamenti single-ended tradizionali.
Poiché i diversi moduli del pacco batteria operano a potenziali elettrici differenti, il collegamento deve inoltre garantire un'elevata immunità alle tensioni di modo comune (_common-mode voltage_) generate dalle rapide commutazioni dell'inverter.
Per questo motivo Analog Devices prevede l'utilizzo di un accoppiamento magnetico tra i nodi della catena, ottenuto mediante trasformatori dedicati. Il trasformatore trasferisce esclusivamente le variazioni del segnale differenziale e contribuisce ad attenuare le componenti di rumore presenti simultaneamente sui due conduttori.
#### Trasformatore di accoppiamento

Per la fase di prototipazione è stato previsto il footprint del trasformatore **Pulse HM2100NL**, componente raccomandato per applicazioni di trasmissione dati differenziale ad elevata immunità ai disturbi.
Le principali caratteristiche del dispositivo sono:
- rapporto di trasformazione 1:1;
- isolamento ingresso-uscita pari a 4300 VDC;
- tensione di lavoro pari a 1000 VDC;
- leakage inductance massima di 0.5 µH;
- Common Mode Rejection Ratio (CMRR) minimo di 35 dB tra 1 MHz e 10 MHz.
L'elevato valore di CMRR consente di ridurre significativamente i disturbi di modo comune indotti dai cablaggi ad alta tensione e dall'inverter del veicolo.
#### Soluzioni implementate sulla scheda
Poiché il comportamento del collegamento dipende fortemente dall'ambiente elettromagnetico reale del veicolo, il PCB è stato progettato prevedendo due possibili configurazioni selezionabili durante l'assemblaggio:
##### Accoppiamento capacitivo
Configurazione raccomandata da Analog Devices per collegamenti punto-punto di lunghezza ridotta.
I segnali isoSPI vengono accoppiati tramite condensatori e resistenze secondo il circuito applicativo riportato nel datasheet dell'ADBMS6815.
![[isoSPI capacitivo.png]]
##### Accoppiamento tramite trasformatore
Configurazione alternativa utilizzabile durante le attività di validazione.
In questo caso il collegamento tra i nodi isoSPI viene realizzato tramite il trasformatore HM2100NL, consentendo di valutare sperimentalmente l'efficacia dell'accoppiamento magnetico nel ridurre gli errori di comunicazione in presenza dei disturbi generati dall'inverter.
La selezione della configurazione avviene mediante la popolazione alternativa dei componenti sul PCB, senza richiedere modifiche hardware alla scheda.
![[isoSPI trasformatore.png]]
##### Attività di validazione
Durante la campagna di test verranno confrontate entrambe le configurazioni al fine di determinare la soluzione più robusta per l'applicazione Formula Student.
Le prove previste comprendono:
- verifica della comunicazione con inverter spento;
- verifica della comunicazione con inverter alimentato;
- verifica della comunicazione durante il funzionamento del motore;
- monitoraggio di eventuali errori di comunicazione o perdita di pacchetti;
- confronto delle prestazioni tra accoppiamento capacitivo e accoppiamento magnetico.

L'obiettivo finale è individuare la configurazione che garantisca la massima affidabilità della comunicazione BMS nelle condizioni operative reali del veicolo.

### ==🟠Fase di testing==
Dopo aver revisionato la scheda, partirà la fase di prototipazione con 5 schede da testare. Per la comunicazione l'idea iniziale è quella di acquistare l'evaluation board di ADI, la DC2792B che monta il componente LTC6820 che converte la comunicazione isoSPI a SPI collegabile al microcontrollore della scheda di controllo del pacco batterie. Un primo test sarà attraverso una board di ST, affinché il codice software possa essere già validato per la scheda presente nel pacco batterie.
In fase di testing schede bisognerà verificare:
- immunità EMI
- l'accuratezza delle misure temperatura.
- il balancing a corrente elevata.
- la comunicazione isoSPI con inverter acceso.


