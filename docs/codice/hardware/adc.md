# ADC (Analog to Digital Converter)
Un convertitore analogico-digitale (ADC) è un dispositivo elettronico la cui funzione è trasformare un segnale analogico, continuo nel tempo e nell'ampiezza, in un segnale digitale, discreto sia nel tempo che nell'ampiezza. Questo processo di "traduzione" è fondamentale per l'interfacciamento tra i sensori del mondo reale e i sistemi di elaborazione numerica (come microcontrollori o computer).

L'intero processo si fonda su due operazioni concettuali distinte e sequenziali: il **campionamento** (discretizzazione nel tempo) e la **quantizzazione** (discretizzazione nell'ampiezza). Queste operazioni teoriche sono implementate da specifiche architetture hardware.
### Il Campionamento e il Circuito di Sample and Hold (S/H)

Il primo passo è la discretizzazione temporale. Un segnale analogico $v_{in}(t)$ ha un valore definito per ogni istante di tempo $t$. Un sistema digitale non può acquisire un'infinità di valori in un dato intervallo; deve "fotografare" il segnale a intervalli di tempo specifici. Questo processo è il **campionamento**.

#### Teoria: Il Teorema di Nyquist-Shannon

La teoria fondamentale del campionamento stabilisce che, per poter ricostruire fedelmente un segnale analogico a partire dai suoi campioni, la frequenza di campionamento $f_s$ deve essere rigorosamente maggiore del doppio della massima frequenza contenuta nel segnale stesso, $f_{max}$.

$$f_s > 2 \cdot f_{max}$$

Questa condizione è nota come **criterio di Nyquist**. Se il criterio non è rispettato, si verifica un fenomeno distorsivo chiamato **aliasing**, in cui le componenti spettrali del segnale a frequenze superiori a $f_s/2$ (la frequenza di Nyquist) si "ripiegano" nella banda di interesse, apparendo come segnali a frequenza più bassa e corrompendo irrimediabilmente l'informazione. Per questo motivo, un ADC è quasi sempre preceduto da un **filtro anti-aliasing**, un filtro passa-basso analogico che elimina le componenti spettrali oltre $f_{max}$ prima che il segnale raggiunga il campionatore.

#### Hardware: Il Circuito di Sample and Hold (S/H)

A livello hardware, il campionamento non è un'operazione istantanea. Il processo di quantizzazione richiede un certo tempo per essere completato, e durante questo tempo il segnale d'ingresso non deve variare. Se il segnale variasse durante la quantizzazione, il risultato digitale sarebbe indefinito e affetto da errore.

Per risolvere questo problema, si utilizza un circuito fondamentale chiamato **Sample and Hold (S/H)**, o talvolta _Track and Hold_.

Questo circuito è concettualmente semplice:

1. Un **interruttore analogico** (tipicamente un MOSFET).
2. Un **condensatore di hold** (o di campionamento), $C_H$.
3. Un **buffer** (amplificatore operazionale in configurazione _voltage follower_) per isolare il condensatore dal resto del circuito.

Il suo funzionamento avviene in due fasi:

- **Fase di Sample (o Track):** L'interruttore è chiuso. Il condensatore $C_H$ si carica (o scarica) per seguire la tensione di ingresso $v_{in}(t)$. Il tempo necessario affinché la tensione ai capi del condensatore, $v_C$, raggiunga un valore sufficientemente prossimo a $v_{in}$ (entro una frazione di LSB) è chiamato **Tempo di Acquisizione** ($t_{acq}$). Questo tempo dipende dalla resistenza dell'interruttore ($R_{on}$) e dalla capacità $C_H$ (costante di tempo $\tau = R_{on} \cdot C_H$).
    
- **Fase di Hold:** L'interruttore viene aperto istantaneamente (in teoria). La carica accumulata su $C_H$ non ha più un percorso a bassa impedenza per scaricarsi. Il condensatore "mantiene" quindi la tensione $v_C$ costante al valore che $v_{in}(t)$ aveva nell'istante esatto dell'apertura. Questa tensione stabile è ora disponibile per il blocco di quantizzazione.
    

Nella realtà, l'apertura non è istantanea. Il breve intervallo in cui l'interruttore passa da chiuso ad aperto è il **tempo di apertura** (_aperture time_). L'incertezza statistica su _quando_ esattamente avviene questa transizione è il **jitter di apertura**, una fonte critica di rumore e errore nelle conversioni ad alta frequenza. Durante la fase di Hold, inoltre, la tensione non rimane perfettamente costante a causa di correnti di perdita (correnti di _leakage_ dell'interruttore e correnti di _bias_ del buffer), provocando un leggero decadimento della tensione (_droop rate_).

### La Quantizzazione e l'Errore di Quantizzazione

Una volta che il circuito S/H ha "congelato" un valore di tensione analogica $v_C$, inizia il secondo processo: la **quantizzazione**. Questa è la discretizzazione dell'ampiezza.
La quantizzazione consiste nell'associare il valore analogico $v_C$ (che appartiene a un intervallo continuo) a uno tra un numero finito di livelli digitali. Il numero di questi livelli è determinato dalla **risoluzione** dell'ADC, espressa in bit ($N$).

Un ADC a $N$ bit può produrre $2^N$ codici digitali distinti. Ad esempio, un ADC a 8 bit ha $2^8 = 256$ livelli; un ADC a 12 bit ne ha $2^{12} = 4096$.

#### Hardware: Il Riferimento di Tensione ($V_{ref}$)

L'hardware che definisce l'intervallo di tensioni che l'ADC può misurare è il **riferimento di tensione** ($V_{ref}$). Questo è uno dei componenti più critici per l'accuratezza dell'ADC. L'ADC mappa l'intervallo di tensioni d'ingresso, definito come $V_{FS}$ (_Full-Scale Range_), che solitamente va da una $V_{ref-}$ (spesso 0V, o GND) a una $V_{ref+}$ (spesso $V_{ref}$ stessa), sull'intero insieme dei $2^N$ codici digitali.

L'ampiezza di un singolo livello discreto, ovvero la minima variazione di tensione che l'ADC è teoricamente in grado di distinguere, è chiamata **quantum** o **LSB** (Least Significant Bit). Il suo valore è:

$$q = \text{LSB} = \frac{V_{FS}}{2^N} = \frac{V_{ref+} - V_{ref-}}{2^N}$$

Ad esempio, un ADC a 10 bit ($2^{10} = 1024$ livelli) con un riferimento $V_{ref} = 5.0\text{V}$ (e $V_{ref-} = 0\text{V}$) avrà un LSB di $q = \frac{5.0\text{V}}{1024} \approx 4.88\text{mV}$.

#### Teoria: L'Errore di Quantizzazione

Il processo di quantizzazione è intrinsecamente un'approssimazione. Qualsiasi tensione analogica $v_C$ che cade tra due livelli di decisione $V_k$ e $V_{k+1}$ viene mappata a un singolo codice digitale (ad esempio, il codice $k$).

Questa differenza tra il valore analogico reale campionato $v_C$ e il valore digitale $V_{digitale}$ che lo rappresenta è un errore ineluttabile chiamato **errore di quantizzazione** ($e_q$).

$$e_q = v_C - V_{digitale}$$

Nel caso ideale, il circuito di quantizzazione assegna al campione analogico il codice digitale più vicino. In questa situazione, l'errore di quantizzazione è sempre limitato a metà del gradino minimo.

$$|e_q| \leq \frac{q}{2} \text{ (ovvero } \pm \frac{1}{2} \text{LSB)}$$

Questo errore è inevitabile e viene spesso modellato come una fonte di rumore (il **rumore di quantizzazione**), la cui potenza è direttamente legata alla risoluzione $N$. Aumentare la risoluzione (più bit) riduce il $q$ e, di conseguenza, riduce il rumore di quantizzazione, aumentando il **rapporto segnale/rumore** (SNR) dell'ADC.

### Il Ciclo di Conversione e i Tempi Operativi

L'intero processo non è istantaneo. L'operazione completa di un ADC si svolge in un ciclo che somma i tempi delle fasi viste finora.

Il tempo totale per ottenere un singolo campione digitale è il **tempo di ciclo** ($T_{cycle}$), dato dalla somma del tempo di acquisizione e del tempo di conversione:

$$T_{cycle} = t_{acq} + t_{conv}$$

1. **Tempo di Acquisizione ($t_{acq}$):** È il tempo, discusso prima, in cui il circuito S/H è in fase "Sample" e il condensatore $C_H$ si carica alla tensione di ingresso.
    
2. **Tempo di Conversione ($t_{conv}$):** È il tempo richiesto dall'hardware _interno_ dell'ADC (che può essere un'architettura SAR, Flash, Delta-Sigma, ecc.) per eseguire l'algoritmo di quantizzazione e codifica. È il tempo che intercorre tra il comando di "Hold" e l'istante in cui il dato digitale $N$-bit è valido e stabile sull'uscita.
    

La massima frequenza di campionamento $f_s$ che l'ADC può sostenere è l'inverso del tempo di ciclo totale:

$$f_s = \frac{1}{T_{cycle}} = \frac{1}{t_{acq} + t_{conv}}$$

Questi parametri ($t_{acq}$ e $t_{conv}$) sono fondamentali nei datasheet di un ADC e determinano il _throughput_ (la velocità) massimo del convertitore. Per esempio, un ADC "pipeline" può iniziare l'acquisizione di un nuovo campione _mentre_ sta ancora convertendo quello precedente, ottimizzando la velocità complessiva e disaccoppiando $f_s$ dalla somma semplice dei due tempi. Tuttavia, per le architetture più comuni come il SAR (Successive Approximation Register), questo ciclo $t_{acq} + t_{conv}$ definisce rigidamente la massima velocità operativa.

## ADC sulla centralina FAE

![Centralina fae](adcfae.png)

Nell'immagine in alto è descritto il **circuito di condizionamento del segnale** che si trova _prima_ dell'ADC nella centralina FAE che al momento utilizziamo.

Il suo compito è prendere un segnale industriale "grezzo" (come un 0-5V o un 4-20mA), proteggere l'elettronica, filtrarlo e adattarlo affinché possa essere letto correttamente e in sicurezza da un pin di ingresso dell'ADC (presumibilmente un microcontrollore).

Analizziamo il percorso del segnale, passo dopo passo, per un singolo canale (ad esempio, quello di sinistra, che è identico a quello di destra).

---

### 1. Stadio di Ingresso e Conversione Corrente/Tensione (I/V)

Il segnale entra dal connettore **J38**. Lo schema indica che questo ingresso accetta due tipi di segnali standard industriali: 0-5V (in tensione) o 4-20mA (in corrente).

- **R471 (200 $\Omega$)**: Questo è il componente chiave per l'ingresso in corrente. È un **resistore di shunt** (o _burden resistor_). Per la legge di Ohm ($V = I \cdot R$), questo resistore converte la corrente che lo attraversa in una tensione.
    - Se l'ingresso è **4mA** (minimo segnale): $V = 0.004 \text{ A} \times 200 \text{ } \Omega = \mathbf{0.8 \text{ V}}$
    - Se l'ingresso è **20mA** (massimo segnale): $V = 0.020 \text{ A} \times 200 \text{ } \Omega = \mathbf{4.0 \text{ V}}$
- Se l'ingresso è un segnale 0-5V, questo viene semplicemente applicato ai capi di questo resistore (e dei componenti a valle).

Quindi, questo stadio unifica entrambi i tipi di segnale in un segnale di tensione.

### 2. Stadio di Protezione (Clamping)

Il segnale in tensione (che, come abbiamo visto, può arrivare fino a 4.0V o 5.0V) incontra subito il diodo **D5 (BAT54SW)**.

- Questo è un package contenente **due diodi Schottky** connessi in modo da realizzare un **circuito di clamping**.
- Il catodo del diodo superiore è connesso all'alimentazione (`3V3_MAIN`, cioè 3.3 Volt).
- L'anodo del diodo inferiore è connesso a massa (`GND_DIGITAL`).
- Il segnale di ingresso è connesso al punto centrale.

**Funzione:** Questo circuito "blocca" (delimita) la tensione per proteggere i componenti successivi.

- Se la tensione di ingresso tenta di salire _sopra_ i 3.3V, il diodo superiore entra in conduzione e "scarica" la tensione in eccesso verso l'alimentazione (limitando la tensione a circa $3.3\text{V} + V_f \approx 3.6\text{V}$).
    
- Se la tensione di ingresso tenta di scendere _sotto_ lo 0V (ad esempio a causa di un disturbo negativo), il diodo inferiore entra in conduzione e la "ancora" a massa (limitandola a circa $-V_f \approx -0.3\text{V}$).
    

**Nota importante:** Poiché l'alimentazione è a 3.3V, questo circuito _clippa_ attivamente i segnali 4-20mA (che arrivano a 4.0V) e 0-5V. Ciò significa che l'intervallo di misura _effettivo_ del circuito è limitato dalla tensione di alimentazione. Qualsiasi segnale che produce una tensione superiore a $\approx 3.3\text{V}$ verrà letto come "fondo scala" dall'ADC.

### 3. Stadio di Filtraggio Anti-Aliasing (Passa-Basso)

Il segnale, ora protetto, passa attraverso un **filtro RC passa-basso** passivo.
- **R305 ($10\text{k}\Omega$)** e **C231 (1nF)** formano questo filtro.

**Funzione:** Questo è il **filtro anti-aliasing** di cui abbiamo parlato nella teoria. Il suo compito è eliminare le componenti di rumore ad alta frequenza dal segnale _prima_ che questo venga campionato dall'ADC. Questo previene l'errore di _aliasing_. La sua frequenza di taglio è:

$$f_c = \frac{1}{2\pi \cdot R \cdot C} = \frac{1}{2\pi \cdot 10 \cdot 10^3 \text{ } \Omega \cdot 1 \cdot 10^{-9} \text{ F}} \approx 15.9 \text{ kHz}$$

Qualsiasi rumore o segnale spuria con frequenza superiore a $\approx 16\text{kHz}$ viene significativamente attenuato.

### 4. Stadio di Buffering (Inseguitore di Tensione)

Il segnale filtrato entra nel componente **U30 (AD8542)**. Questo è un amplificatore operazionale (in package doppio).

- Il segnale entra nell'ingresso non-invertente (pin 3, `+INA`).
- L'uscita (pin 1, `OUTA`) è collegata direttamente all'ingresso invertente (pin 2, `-INA`).

Questa configurazione è chiamata **voltage follower** (o _inseguitore di tensione_).

**Funzione:** Questo stadio è fondamentale e svolge un doppio ruolo:

1. **Adattamento di Impedenza:** L'op-amp ha un'impedenza di ingresso altissima. Ciò significa che "guarda" il filtro RC (R305/C231) senza "caricarlo", ovvero senza assorbire corrente da esso, garantendo che il filtro funzioni come calcolato.
    
2. **ADC Driver:** L'op-amp ha un'impedenza di uscita bassissima. Questo è cruciale. L'ingresso dell'ADC (che non vediamo, ma che è collegato a `ADC1_IN`) contiene il circuito di **Sample and Hold (S/H)**. Quando l'S/H commuta per "campionare", il suo condensatore interno ($C_H$) deve caricarsi istantaneamente. L'op-amp, con la sua bassa impedenza di uscita, è in grado di fornire la "botta" di corrente necessaria per caricare $C_H$ rapidamente e senza far "cadere" la tensione del segnale.

### 5. Stadio di Uscita e Serbatoio di Carica

L'uscita dall'op-amp (pin 1) non va _direttamente_ all'ADC, ma passa attraverso un ultimo piccolo filtro.
- **R311 ($10\Omega$)** e **C232 (1nF)**.

**Funzione:** Anche questo stadio ha un doppio ruolo critico per l'integrità del segnale:

1. **R311 (Resistore di Isolamento):** Gli op-amp possono diventare instabili (oscillare) se pilotano un carico puramente capacitivo, come l'ingresso di un ADC. Questo piccolo resistore da 10$\Omega$ "isola" l'uscita dell'op-amp dalla capacità $C_{232}$ e da quella dell'ADC, garantendo la stabilità.
    
2. **C232 (Serbatoio di Carica):** Questo condensatore, posto proprio vicino al pin dell'ADC, agisce come un "mini-serbatoio" di carica. Quando l'interruttore S/H dell'ADC si chiude, la maggior parte della carica istantanea necessaria a $C_H$ proviene da C232 (che è fisicamente vicino), non dall'op-amp (che è "più lontano", elettricamente parlando, attraverso R311). Questo minimizza il _voltage droop_ (calo di tensione) istantaneo sul pin dell'ADC, permettendo un campionamento molto più accurato.
    

Il segnale finale, **`ADC1_IN`**, è ora pulito, filtrato, protetto, bufferizzato e pronto per essere convertito in un numero digitale. L'intero circuito è duplicato per gestire un secondo canale analogico.