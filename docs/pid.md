# Progettazione e Implementazione in C di un Controllore PID con Gain Scheduling e Gestione Transitori per ECU Automotive

---
Samuele Stasi - E-Racing Bergamo

---

## Obiettivo

Il problema da risolvere è implementare un modulo che espanda la libreria attuale del PID per integrare **gain scheduling** e la **gestione dei transitori**. In seconda battuta bisogna svolgere una simulazione per verificare il comportamento dell'uscita ad alcune variazioni della variabile di controllo.

## Architettura software

L'originale è una libreria single header scritta in C-98 (standard automotive senza allocazioni di memoria) che permette di inizializzare un controllore PID e svolgere calcoli con le seguenti feature:
- Anti wind-up attraverso il clipping
- Cambiamento delle costanti
- Filtraggio della parte derivativa
- Modalità manuale/automatica

La struttura originale del controllore è definita come:
```c
typedef struct
{
 float kp; /**< Proportional gain */
 float ki; /**< Integral gain */
 float kd; /**< Derivative gain */
 float setpoint; /**< Target value */
 float dt; /**< Sampling time in seconds (e.g., 0.01f for 100Hz) */

 float integral; /**< Integral accumulator */
 float previous_error; /**< Error from last cycle */

 float output_min; /**< Minimum controller output limit */
 float output_max; /**< Maximum controller output limit */

 float max_integral; /**< Anti-windup upper limit */
 float min_integral; /**< Anti-windup lower limit */
 bool anti_windup_en; /**< Anti-windup enablement flag */

 float d_alpha; /**< Alpha for derivative EMA filter [0.0 - 1.0] */
 float prev_d_filtered; /**< State for the EMA filter */
 bool d_filter_en; /**< Derivative filter enablement flag */

 bool is_auto;
 float manual_output;
} PID_Controller_t;
```

Ho deciso di seguire il principio della single responsability, quindi l'header che computa l'uscita dal controllore resta completamente isolato da quello che si occupa di effettuare i cambi di parametri al raggiungimento dei setpoint.

**Gain scheduler**
Il gain scheduler è banalmente una lookup table 1D in cui si vanno a recuperare i valori delle costanti per ogni setpoint. Per semplicità implementativa non si considera il valore di inizio del primo setpoint per essere in grado di gestire ingressi della variabile con cui si confrontano i setpoint che vanno da $-\infty \to +\infty$.

```c
typedef struct {
 float kp;
 float ki;
 float kd;
} PID_Gains_t;

typedef struct {
 PID_Gains_t gainSet;
 float startsAtTarget;
} GainScheduleEntry_t;

typedef struct {
 GainScheduleEntry_t *entries;
 uint16_t numEntries;
 float histeresys;
 float lastMeasurement;
 uint16_t current_index;
} GainSchedule_t;
```

## Gestione delle criticità e transitori

La transizione dal sistema ad anello aperto (modalità `auto` è disattivata) ad anello chiuso è un punto critico per la dinamica del controllore. Se $t^*$ è l'istante della commutazione, si ha che il passaggio all'anello chiuso rappresenta una discontinuità nell'output in quanto le variabili di stato interne al controllore sono nulle.

Per consentire una transizione senza discontinuità (Bumpless transition) bisogna precaricare l'integrale con il valore che avrebbe avuto se fosse stato attivo fin dal principio. Lo stato integrale equivalente $\bar{e}(t^*)$ deve essere forzato a rispettare la seguente equazione:

$$\bar{e}(t^*) = \left[ u(t^*) - K_p e(t^*) \right] \frac{1}{K_I}$$

Nella transizione all'implementazione digitale per la ECU, questa formulazione è stata ottimizzata. All'interno della struttura dati `PID_Controller_t`, la variabile di stato `pid->integral` memorizza direttamente l'intera componente integrale già scalata per il guadagno (ovvero $K_I \int e(\tau)d\tau$), rendendo superflua la divisione per $K_I$ a runtime e risparmiando cicli di clock della CPU.
Estendendo il concetto al controllore PID completo (includendo il termine derivativo per evitare picchi causati dal filtro passa-basso sulla derivata), la libreria traccia dello stato in tempo reale. Finché il sistema opera in modalità manuale, la funzione campiona costantemente la velocità reale per calcolare l'errore $e(t^*)$ ed esegue il pre-carico dell'integrale invertendo l'equazione di controllo:

```c
// If we are in manual mode, we track for a bumpless transfer
  if (!pid->is_auto)
  {
    float error = pid->setpoint - measurement;
    float P = pid->kp * error;
    
    // ... [Derivative calculation omitted] ...

    // Exact state formulation matching the theoretical requirements
    pid->integral = output - P - D;

    // Apply anti-windup to the preloaded integral
    if (pid->anti_windup_en)
    {
      if (pid->integral > pid->max_integral)
        pid->integral = pid->max_integral;
      else if (pid->integral < pid->min_integral)
        pid->integral = pid->min_integral;
    }
    
    pid->previous_error = error;
  }
```

Questo approccio assicura che, nel ciclo di clock immediatamente successivo all'attivazione dell'algoritmo automatico, la somma $P + I + D$ generi un valore matematicamente identico all'ultima uscita manuale richiesta, garantendo una transizione fisicamente e meccanicamente trasparente.

## Mitigazione del proportional kick

Per adattare la legge di controllo alla dinamica non lineare del veicolo, l'architettura implementata sfrutta una tecnica di gain scheduling. Per soddisfare i rigidi vincoli computazionali imposti dal microcontrollore, si è scelto di scartare approcci ad interpolazione lineare continua, preferendo una commutazione a stati discreti (lookup table a gradino).

Il passaggio istantaneo tra due zone operative al tempo $t^*$ comporta un cambiamento brusco dei parametri, in particolare del guadagno proporzionale, che commuta da $K_p^1$ a $K_p^2$. Essendo l'azione proporzionale una componente algebrica istantanea definita come $P(t) = K_p \cdot e(t)$, una variazione a gradino del parametro genera un transitorio fittizio, il proportional kick.
La discontinuità introdotta nell'azione di controllo è quantificabile come:

$$\Delta P = (K_p^2 - K_p^1) \cdot e(t^*)$$

Per eliminare il problema e preservare la continuità del segnale di controllo $u(t^-) = u(t^+)$, la libreria implementa una compensazione attiva che sfrutta il termine integrale come ammortizzatore algebrico. Nel momento esatto della commutazione, l'algoritmo inietta un offset nell'accumulatore integrale esattamente pari e opposto alla variazione subita dall'azione proporzionale.
L'aggiornamento dello stato integrale obbedisce quindi alla legge:

$$I(t^+) = I(t^-) + (K_p^1 - K_p^2) \cdot e(t^*)$$

A livello software, questa compensazione deve essere calcolata ed applicata strettamente prima che il modulo PID aggiorni la propria struttura dati interna con i nuovi guadagni. Di seguito l'implementazione nella funzione `GainSchedule_Update`:

```c
  if (target_index != schedule->current_index) {
    float new_kp = schedule->entries[target_index].gainSet.kp;
    float new_ki = schedule->entries[target_index].gainSet.ki;
    float new_kd = schedule->entries[target_index].gainSet.kd;

    // Anti-kick compensation
    float error = pid->setpoint - measurement;
    pid->integral += (pid->kp - new_kp) * error;
    
    // Anti windup
    if (pid->anti_windup_en) {
      if (pid->integral > pid->max_integral) 
        pid->integral = pid->max_integral;
      else if (pid->integral < pid->min_integral) 
        pid->integral = pid->min_integral;
    }

    // Parameter commutation
    PID_SetGains(pid, new_kp, new_ki, new_kd);
    schedule->current_index = target_index;
  }
```

Questa manipolazione dello stato assicura che, nell'esatto istante $t^*$, la somma delle componenti $P(t) + I(t)$ resti invariata, chiedendo al nuovo guadagno integrale $K_2$ e alla nuova dinamica del sistema l'onere di smaltire gradualmente il valore di pre-carico nei cicli successivi, garantendo una fluidità non indifferente.

## Isteresi a stati discreti

In sistemi fisici reali le letture dei sensori sono intrinsecamente affette da rumori di lettura ad alta frequenza. Una continua variazione dei valori attorno al punto di cambiamento dei parametri comporterebbe un sovraccarico sia dal punto di vista del microcontrollore che deve continuare a compensare l'azione derivativa, sia per il gruppo meccanico/inverter soggetto a sbalzi di coppia e quindi a surriscaldamento.

Si è optato per un'architettura basata sulla memoria di stato, implementando un'isteresi asimmetrica. L'algoritmo non valuta la pendenza istantanea del segnale, ma adatta dinamicamente la soglia di scatto in base alla zona operativa in cui il controllore si trova in quel preciso momento.

Definita una soglia base $Th$ e un valore di isteresi $h$, la soglia effettiva $Th_{eff}$ viene calcolata come segue:

- In accelerazione (transizione verso una zona superiore): Il sistema richiede di superare la soglia maggiorata, ovvero $Th_{eff} = Th + h$.
- In decelerazione (transizione verso una zona inferiore): Il sistema richiede di scendere al di sotto della soglia minorata, ovvero $Th_{eff} = Th - h$.

A livello firmware, la logica è integrata direttamente nel ciclo di scansione della lookup table delle zone operative all'interno di `GainSchedule_Update`:
```c
for (uint16_t i = 1; i < schedule->numEntries; i++) {
    float threshold = schedule->entries[i].startsAtTarget;
    float effective_threshold = threshold;

    // Hysteresis logic
    if (schedule->current_index < i) {
      // Acceleration transition
      effective_threshold += schedule->histeresys;
    } else {
      // Deceleration transition
      effective_threshold -= schedule->histeresys;
    }

    // Threshold verification
    if (measurement < effective_threshold) {
      target_index = i - 1;
      break;
    }
  }
```

Questa implementazione crea una banda bidirezionale di ampiezza $2h$ attorno al punto di commutazione ideale. Se il veicolo viaggia in prossimità del limite, l'eventuale rumore di lettura non è sufficiente a superare la barriera dell'isteresi, bloccando il sistema nello stato attuale e garantendo l'immunità totale al rumore di lettura.

## Simulazioni
Per validare la robustezza e la correttezza logica della libreria in assenza di un banco prova hardware, è stato sviluppato un ambiente di simulazione in C. L'obiettivo dell'esperimento è simulare la logica di una ECU preposta al controllo longitudinale, la quale deve adattare i parametri di erogazione in tempo reale in base allo stato del fondo stradale.

Il controllore PID è stato configurato con limiti di saturazione asimmetrici pari a $[-200, 400]$ Nm, vincolando la coppia massima erogabile in trazione e la coppia minima richiesta in fase di frenata rigenerativa. La variabile di scheduling passata al modulo decisionale non è la velocità, bensì una stima del coefficiente di attrito dell'asfalto ($\mu$).

I guadagni sono stati partizionati in due stati:

- Zona 0 (Bagnato / Bassa Aderenza): Guadagni conservativi ($K_p=1.0, K_i=0.1$) per prevenire lo slittamento.
- Zona 1 (Asciutto / Alta Aderenza): Guadagni aggressivi ($K_p=3.0, K_i=0.5$) per massimizzare la reattività del powertrain.

Il punto di commutazione ideale è stato fissato a $\mu = 0.5$, con un'isteresi di sicurezza $h = 0.05$.

### Sollecitazioni

Il sistema è stato sottoposto al tracking di un setpoint di velocità sinusoidale. Contemporaneamente, il fondo stradale è stato modellato con una forzante tempo-variante:

$$\mu(t) = 0.5 + 0.3 \sin(2\pi f t)$$

Questa condizione forza il veicolo a transitare ripetutamente tra pozzanghere (asfalto bagnato, $\mu \approx 0.2$) e tratti asciutti ($\mu \approx 0.8$), obbligando la ECU a commutare ciclicamente la mappatura dei guadagni per mantenere la stabilità.

L'esportazione dei log di esecuzione conferma la robustezza dell'implementazione:

1. Isteresi su stima di attrito: Il ritardo di fase indotto dalla logica di stato si evince chiaramente. Quando l'asfalto si sta asciugando ($\mu$ in aumento), il controllore mantiene la mappa da bagnato ignorando l'incrocio della soglia teorica ($\mu=0.5$), rilasciando la coppia aggressiva solo al raggiungimento di $\mu = 0.55$. Viceversa, quando il manto stradale peggiora, la ECU attende che $\mu$ scenda a $0.45$ prima di tagliare i guadagni. Questo isola l'inverter dal rumore tipico degli stimatori di attrito.
2. Continuità dell'azione di controllo: Nonostante la centralina scali il termine proporzionale da 1.0 a 3.0 in un singolo ciclo di clock, l'uscita in coppia verso il motore non subisce alcuno scalino. L'assorbimento dell'urto tramite l'accumulatore integrale garantisce transizioni meccanicamente trasparenti al variare del grip, requisito fondamentale in ambito automotive per non destabilizzare l'assetto del veicolo.

![[Screenshot 2026-04-29 alle 15.44.33.png]]
## Sviluppi futuri
Probabilmente in macchina si utilizzerà un EKF per fondere i dati e calcolare il $\mu$ dato che l'interazione tra pneumatico e asfalto non è lineare.