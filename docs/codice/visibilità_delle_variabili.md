
# Visibilità e organizzazione del codice in C

Oltre a sapere cos’è una variabile, è fondamentale capire **dove è visibile** e **come organizzare i file** in un progetto embedded.

---

## File `.c` e `.h`

In un progetto in C i file sono generalmente divisi in due categorie:

- **File `.c`** → contengono il codice implementativo (funzioni, logica).  
- **File `.h`** → contengono le dichiarazioni (prototipi di funzioni, definizioni di strutture, costanti, variabili globali).  

### Esempio

**sensore.h**
```c
#ifndef SENSORE_H
#define SENSORE_H

void sensore_init(void);
int sensore_leggi(void);

#endif
````

**sensore.c**

```c
#include "sensore.h"

void sensore_init(void) {
    // inizializzazione hardware
}

int sensore_leggi(void) {
    return 42; // valore di esempio
}
```

Così il modulo `sensore` può essere usato anche da altri file senza duplicare il codice.

---

## Visibilità delle variabili

La **visibilità** definisce in quali file o funzioni una variabile è accessibile.

### Variabili locali

Dichiarate **dentro una funzione** → visibili solo lì, allocate nello **stack**.

```c
void funzione() {
    int x = 10; // visibile solo dentro questa funzione
}
```

### Variabili globali

Dichiarate **fuori da ogni funzione** → visibili in tutto il file `.c`.

```c
int contatore = 0; // visibile in tutto il file
```

Se vogliamo usarle anche da altri file → serve la parola chiave `extern` nell’header.

```c
// in main.c
int contatore = 0;

// in sensore.h
extern int contatore;
```

---

## Il ruolo di `static`

La parola chiave `static` ha due significati diversi a seconda del contesto.

### 1. Variabili locali statiche

Mantengono il loro valore tra più chiamate della funzione.

```c
void funzione() {
    static int chiamate = 0;
    chiamate++;
    printf("%d\n", chiamate);
}
```

Ogni volta che chiamo `funzione()`, la variabile non si azzera ma “ricorda” il valore precedente.

### 2. Variabili e funzioni a livello di file

Limitano la **visibilità al solo file `.c`** (non esportate).

```c
static int interno = 5;

static void helper() {
    // usata solo in questo file
}
```

Questo è molto utile per l’**incapsulamento**: evitiamo che variabili o funzioni “interne” inquinino lo spazio globale del progetto.

---

## Buone pratiche

- Le variabili devono essere **il più locali possibile**.
- Le globali vanno usate solo se davvero necessarie.
- Usare `static` per tutto ciò che è interno a un modulo (`.c`).
- Negli header `.h` mettere solo:
    - prototipi delle funzioni pubbliche,
    - definizioni di costanti/struct/enum,
    - dichiarazioni `extern` se serve condividere una variabile globale.

Così il progetto rimane ordinato, modulare e più facile da manutenere.