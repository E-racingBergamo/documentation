# State Of Charge (SOC)

> [!danger] High voltage
> La batteria ha una tensione che oscilla tra 504V e 588V, è una tensione letale. Osservare tutte le indicazioni di sicurezza prima di iniziare a lavorare sul pacco.

> [!note] Configurazione pacco
> Il nostro pacco batterie è divisa in 14 moduli in serie con 10 celle ciascuno. Ogni cella ha il suo BMS Enepaq tinyAFE per ottenere informazioni riguardanti tensioni e temperature e bilanciare le singole celle.
>
> Le celle sono delle sony 4p, quindi la struttura totale della batteria è 140S 4P.
> 

Dato che i TinyAFE non sono dei veri e propri BMS, non possono calcolare autonomamente lo state of charge dei moduli, quindi si rendono necessari degli elementi esterni per fare i calcoli:

- Sensore di corrente
- Microcontrollore esterno
- Linea UART isolata

Ci sono principalmente due metodi per stimare lo stato di carica delle batterie al litio, il primo è basato sulla **tensione**, ma è meno preciso e può dare degli errori anche del 20% quando la batteria è sotto carico, il secondo è il **Coulomb Counting**.

## Primo metodo: tensioni

È il metodo più facile e veloce, ma meno preciso. Lo useremo per calibrare il BMS Master all'accensione della vettura.

Le celle utilizzate all'interno dei moduli sono delle Sony/Murata VTC6-4 (chimica NMC) e hanno una curva di scarica caratteristica fornita dal costruttore e mostrata nell'immagine qua sotto.

![Curva di scarica](discharge_characteristic.png)

Bisogna mappare la tensione del pacco oppure una media delle celle lette dal TinyAFE a una percentuale, usando una tabella estratta dalla curva di scarica al più basso carico. È stato scelto il carico più basso del grafico e non 0 perché ci sono dei componenti, come ad esempio le pompe di raffreddamento, che attingono energia direttamente dal pacco e rischiano di rovinare le celle se si permette l'accensione ad un livello basso senza contare questi consumi. La tensione di fine scarica è stata alzata a 2.80V per motivi di sicurezza e per preservare l'integrità delle celle.

| **SoC (%)** | **Capacità Residua (mAh)** | **Tensione Cella (V)** | **Note Visive dal Grafico**            |
| ----------- | -------------------------- | ---------------------- | -------------------------------------- |
| **100%**    | 3000                       | **4.20 V**             | Carica completa                        |
| **90%**     | 2700                       | **4.08 V**             | Caduta iniziale rapida                 |
| **80%**     | 2400                       | **4.00 V**             | Inizio zona lineare                    |
| **70%**     | 2100                       | **3.91 V**             |                                        |
| **60%**     | 1800                       | **3.82 V**             |                                        |
| **50%**     | 1500                       | **3.72 V**             | Metà grafico (Asse X = 1500)           |
| **40%**     | 1200                       | **3.62 V**             |                                        |
| **30%**     | 900                        | **3.52 V**             | La curva si accentua                   |
| **20%**     | 600                        | **3.40 V**             |                                        |
| **10%**     | 300                        | **3.20 V**             | Ginocchio della curva (scarica rapida) |
| **5%**      | 150                        | **3.05 V**             | Zona critica                           |
| **0%**      | 0                          | **< 2.80 V**           | Cutoff di sicurezza                    |
Questo metodo fallisce quando il pacco è sotto carico per via del fatto che la tensione scende a causa della resistenza interna. Il ragionamento vale ugualmente anche quando il pacco è in carica solo che invece che scendere, la tensione sale.

## Secondo metodo: Coulomb Counting

Il Coulomb Counting è metodo semplice per stimare lo Stato di Carica (SOC) di una batteria, misurando e **integrando nel tempo la corrente** che entra o esce dalla batteria (Ampere-ora) per calcolare la carica rimanente, partendo da un SOC iniziale noto.
La formula dell'algoritmo è la seguente:

$$SoC(t) = SoC(t_0) - \frac{1}{C_{tot}} \int_{t_0}^{t} I(\tau) d\tau$$

Dove:

- $SoC(t_0)$ è lo stato di carica iniziale calcolato attraverso il primo metodo delle tensioni.
- $C_{tot}$ è la capacità totale delle celle. Le nostre sono da $3000mAh$ e ne abbiamo 4 in parallelo, risulta che la capacità vale $12000mAh$.
- $I(\tau)$ è la corrente fornita o sottratta in un lasso di tempo

Parlando concretamente dobbiamo discretizzare la formula per renderla compatibile con un sistema a tempo discreto come un microcontrollore, allo scopo di effettuare il calcolo ad una frequenza fissa ($10Hz$)

$$Q(t) = Q(t-1) + (I_{inst} \times \Delta t)$$

dove:

- $Q(t)$ è la carica attuale
- $I_{inst}$ è la corrente istantanea letta dal sensore (positiva se è in carica e negativa se è in scarica)
- $\Delta t$ è il lasso di tempo passato dall'ultima lettura, a $10Hz$ vale $\frac{1}{10}s = 100ms$ 

> [!tip] Deriva
> Il sensore di corrente avrà per forza uno scostamento rispetto al valore reale a causa di interferenze elettromagnetiche e quant'altro, falsando il calcolo a lungo andare.
> Per correggere bisogna forzare il SOC al 100% vicino al massimo e allo 0% vicino al minimo.

La frequenza di campionamento è stata impostata a 100ms perché i TinyAFE ci mettono un certo lasso di tempo a rispondere alla chiamata del master e sono 14, una frequenza superiore rischia di falsare il risultato usando valori vecchi.