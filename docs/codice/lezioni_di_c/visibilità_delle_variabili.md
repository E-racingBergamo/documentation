# Visibilità e organizzazione del codice in C

Oltre a sapere cos’è una variabile, è fondamentale capire **dove è visibile** e **come organizzare i file** in un progetto embedded.
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

## Cos’è extern

La keyword `extern` in C serve a **dichiarare** una variabile o funzione definita in un altro file, così che il compilatore sappia che **esiste** e che verrà risolta in fase di linking.

In pratica:

- `dichiarazione` = dire “questa variabile/funzione esiste da qualche parte”.
- `definizione` = riservare davvero memoria o scrivere il codice.

`extern` riguarda solo le variabili globali (anche se si può usare pure con funzioni, ma lì è implicito).
## Senza `extern` (definizione)

Se scrivi:

```c
int counter = 0;
```

questa è una **definizione**: il compilatore alloca spazio in memoria per `counter`.
## Con `extern` (dichiarazione)

Se scrivi:

```c
extern int counter;
```

questa è solo una **dichiarazione**: non alloca memoria, dice solo “da qualche parte esiste un `int counter`”.  
In questo modo puoi usare `counter` anche in un file diverso da quello in cui è stato definito.

## Esempio pratico con più file

**file1.c**

```c
#include <stdio.h>

int counter = 42;   // definizione (memoria allocata qui)

void printCounter(void) {
    printf("counter = %d\n", counter);
}
```

**file2.c**

```c
#include <stdio.h>

extern int counter;  // dichiarazione (nessuna memoria allocata)

void increment(void) {
    counter++;
}
```

**main.c**

```c
void printCounter(void);
void increment(void);

int main(void) {
    printCounter();  // counter = 42
    increment();
    printCounter();  // counter = 43
    return 0;
}
```

Compilando tutti e tre insieme funziona perché il linker trova `counter` definito in `file1.c`:

```bash
gcc file1.c file2.c main.c -o program
```

## Casi tipici in Embedded

Noi `extern` lo usiamo per:

- **Variabili globali condivise** tra più moduli (es. uno stato della macchina, un buffer CAN, una coda FreeRTOS).
- **Funzioni scritte in un file ma richiamate da altri**, anche se lì l’`extern` è implicito.
- **Header file**: metti `extern` lì, la definizione vera sta nel `.c`.

Esempio:

```c
// uart.h
#ifndef UART_H
#define UART_H

extern QueueHandle_t uartQueue;   // dichiarazione, visibile a tutti i .c

void UART_Init(void);

#endif
```

```c
// uart.c
#include "uart.h"

QueueHandle_t uartQueue;          // definizione vera (memoria allocata qui)

void UART_Init(void) {
    uartQueue = xQueueCreate(10, sizeof(uint8_t));
}
```

Così ogni modulo che include `uart.h` sa che esiste `uartQueue`, ma la memoria viene allocata una volta sola in `uart.c`.

## Il ruolo di `static`

La parola chiave `static` ha due significati diversi a seconda del contesto.
Il primo caso sono le variabili locali statiche, che mantengono il loro valore tra più chiamate della funzione.

```c
void funzione() {
    static int chiamate = 0;
    chiamate++;
    printf("%d\n", chiamate);
}
```

Ogni volta che chiamo `funzione()`, la variabile non si azzera ma “ricorda” il valore precedente.
il secondo caso sono le variabili e funzioni a livello di file, che limitano la **visibilità al solo file `.c`** (non esportate).

```c
static int interno = 5;

static void helper() {
    // usata solo in questo file
}
```

Questo è molto utile per l’**incapsulamento**, evitando che variabili o funzioni “interne” inquinino lo spazio globale del progetto.

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