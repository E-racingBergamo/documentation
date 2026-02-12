# Strutture dati dinamiche

Le **Strutture Dati Dinamiche** (SDD) in C rappresentano un salto di qualità nella programmazione, permettendo di gestire in modo efficiente la memoria e adattarsi alle esigenze reali di un'applicazione. A differenza delle strutture statiche (come gli array, la cui dimensione è fissata a tempo di compilazione), le SDD possono **espandersi o ridursi** durante l'esecuzione del programma.

Questo è reso possibile dall'uso combinato di due concetti fondamentali in C:

1. **Allocazione Dinamica della Memoria:** L'uso di funzioni come `malloc()`, `calloc()`, `realloc()` e `free()` per richiedere e rilasciare memoria sull'**heap** a tempo di esecuzione.
2. **Puntatori**

## Allocazione e Deallocazione Dinamica

Per creare e gestire le SDD, se si lavora con un ambiente di sviluppo tipicamente desktop, si utilizzano le funzioni della libreria standard C (`stdlib.h`). Nei sistemi embedded spesso l'uso della libreria standard è limitato o del tutto proibito, per questo è necessario trovare delle soluzioni alternative.
### Assenza di un "Sistema Operativo" Completo

Le funzioni della CSL (specialmente I/O e gestione della memoria) assumono l'esistenza di un ambiente operativo completo, con:

- Un **file system** (per `fopen`, `fread`, `fprintf`, ecc.).
- Uno **schermo/console** ben definiti (per `printf`).
- Una **gestione della memoria** (Kernel) che gestisce l'heap in modo robusto.

Nei sistemi embedded, la maggior parte di queste funzionalità è assente o deve essere implementata "da zero" (o tramite driver minimi forniti dal produttore del chip).

### Problemi con le Funzioni di Input/Output (I/O)

#### `printf()`

La funzione `printf()` è molto complessa: deve interpretare la stringa di formato, convertire vari tipi di dati (float, interi lunghi), e infine scrivere l'output in un _stream_ standard (che sia un terminale, una console seriale, o un file).

- **Dimensione del Codice (Footprint):** L'implementazione completa di `printf` può occupare decine di kilobyte di memoria Flash (ROM), che è una quantità sproporzionata per un microcontrollore con 64KB totali.
    
- **Reindirizzamento (Retargeting):** La CSL non sa "dove" si trova la console. Per usare `printf`, lo sviluppatore deve "reindirizzare" le chiamate a una funzione a basso livello che gestisca la comunicazione hardware (es. UART o USB). Questo processo, chiamato _retargeting_, è specifico per ogni hardware e RTOS.
### Gestione Dinamica della Memoria: Il Pericolo di `malloc()`

Questo è l'aspetto più critico e il motivo principale per cui `stdlib.h` viene spesso evitato.

#### Frammentazione dell'Heap e Determinismo

- **Frammentazione:** Nei sistemi che devono funzionare per molto tempo, l'uso ripetuto di `malloc()` e `free()` può causare la **frammentazione dell'heap** (memoria libera divisa in piccoli blocchi non contigui). A lungo andare, una richiesta di memoria grande potrebbe fallire, anche se c'è abbastanza memoria totale libera.
    
- **Non Determinismo:** Le implementazioni standard di `malloc()` e `free()` non sono pensate per l'ambiente real-time. Il tempo necessario per allocare un blocco di memoria può variare drasticamente (dipende dalla ricerca di un blocco libero), violando i requisiti di **determinismo temporale** di un RTOS.

#### Soluzione di FreeRTOS: Heap Management Personalizzato

FreeRTOS non usa l'implementazione standard di `malloc()` ma offre diversi schemi di gestione dell'heap (es. Heap_1, Heap_2, Heap_3, ecc.), progettati per:

- Essere **più semplici e veloci** delle implementazioni standard.
    
- Offrire un **compromesso** tra consumo di memoria e prevenzione della frammentazione, spesso a costo di non supportare tutte le operazioni standard.

### Sicurezza e Thread Safety (Task Safety)

Molte funzioni della CSL (es. `strtok()`, le funzioni di gestione del tempo) non sono **thread-safe**.

- **Variabili Statiche:** Alcune funzioni C utilizzano variabili statiche interne o globali. Se due Task provano a chiamare la stessa funzione contemporaneamente, l'accesso concorrente a queste variabili può portare a **condizioni di gara (race conditions)** e a risultati errati o crash del sistema.

Nei sistemi FreeRTOS, si devono usare:

1. **Versioni Thread-Safe:** Molti RTOS forniscono versioni re-entrant delle funzioni C (spesso chiamate con un suffisso come `_r` o richiedendo l'uso di un mutex).
2. **Primitive RTOS:** Per tutte le operazioni critiche (I/O, allocazione, comunicazione), è obbligatorio usare le API di FreeRTOS (code, semafori, ecc.) per garantire la sincronizzazione e l'integrità dei dati.