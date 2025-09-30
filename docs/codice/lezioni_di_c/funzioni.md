# Funzioni e modularità

Le funzioni sono blocchi di codice riutilizzabili che eseguono un compito specifico. La loro corretta progettazione è fondamentale per la modularità di un progetto, soprattutto in ambienti embedded dove il firmware viene suddiviso in moduli separati e ben organizzati.
### Dichiarazione e definizione

Una funzione si dichiara indicando il **tipo di ritorno**, il **nome** e i **parametri**:

```c
int somma(int a, int b) {
    return a + b;
}
```

Qui `somma` prende due interi e restituisce un intero.  
Il tipo di ritorno può essere qualsiasi tipo valido: `int`, `float`, `char`, `struct`, oppure `void` se non deve restituire nulla.
### Parametri per valore e per riferimento

- **Per valore**: viene passata una copia della variabile, modifiche dentro la funzione non influenzano l’originale.
- **Per riferimento**: si passa l’indirizzo della variabile tramite un puntatore, e quindi la funzione può modificare l’originale.

```c
void incrementa_valore(int x) {
    x++;
}

void incrementa_riferimento(int *x) {
    (*x)++;
}
```

In questo caso il simbolo `*` viene chiamato operatore di dereference, che consente di modificare direttamente il valore della variabile a partire dal suo puntatore. Se nel secondo caso si facesse `x++` si andrebbe ad incrementare il puntatore alla variabile, rischiando di andare oltre la sua area di memoria con operazioni successive e corrompere altre variabili o funzioni.
### Tipi di funzioni

1. **Funzioni con ritorno e parametri**
    ```c
    int moltiplica(int a, int b) {
        return a * b;
    }
    ```
2. **Funzioni senza parametri**
    ```c
    int leggi_sensore(void) {
        return 42;
    }
    ```
3. **Funzioni senza ritorno (`void`)**
    ```c
    void stampa_valore(int v) {
        printf("%d\n", v);
    }
    ```
4. **Funzioni inline**  
    Suggeriscono al compilatore di espandere la funzione al posto della chiamata (ottimizzazione). In poche parole la funzione viene copiata al posto della chiamata. E' consigliabile usarlo solo con funzioni piccole, cioè che fanno una semplice operazione aritmetica o chiamata a funzione.
    ```c
    inline int quadrato(int x) {
        return x * x;
    }
    ```
5. **Funzioni static**  
    Se dichiarate `static` a livello di file `.c`, la loro visibilità è limitata solo a quel file, impedendo che siano richiamabili da altri moduli.
    ```c
    static void reset_interno(void) {
        // codice interno
    }
    ```

E utile conoscere anche la combinazione di static e inline, che permette al compilatore di poter scegliere quando utilizzare inline o quando invece serve avere una definizione della funzione vera e propria.
### Modularità con file `.c` e `.h`

Per mantenere il codice ordinato:

- nel file `.c` si mettono le **definizioni** delle funzioni,
- nel file `.h` si mettono i **prototipi** (cioè la firma senza corpo).

Esempio:

**math_utils.h**

```c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

int somma(int a, int b);
int moltiplica(int a, int b);

#endif
```

**math_utils.c**

```c
#include "math_utils.h"

int somma(int a, int b) {
    return a + b;
}

int moltiplica(int a, int b) {
    return a * b;
}
```

Così gli altri moduli possono includere `math_utils.h` e usare le funzioni senza conoscere l’implementazione.

---

### Puntatori a funzioni

In C è possibile memorizzare l’indirizzo di una funzione in una variabile, e poi richiamarla attraverso questa.  
Questo meccanismo è molto utile per callback e per realizzare codice flessibile.

```c
int somma(int a, int b) { return a + b; }
int moltiplica(int a, int b) { return a * b; }

int calcola(int (*operazione)(int, int), int x, int y) {
    return operazione(x, y);
}

int main(void) {
    int r1 = calcola(somma, 2, 3);
    int r2 = calcola(moltiplica, 2, 3);
}
```

---

### Typedef per puntatori a funzioni

Scrivere i prototipi di puntatori a funzioni può diventare complicato, per questo si usa `typedef`.

```c
typedef int (*Operazione)(int, int);

int calcola(Operazione op, int x, int y) {
    return op(x, y);
}
```

Questo approccio è molto comune in embedded, ad esempio per definire callback da registrare in un driver.

---

### Lambda in C

Il C standard **non supporta le lambda** (funzioni anonime inline) come C++ o altri linguaggi.  
Alcuni compilatori (come GCC) offrono estensioni che permettono funzioni nidificate, ma non sono portabili e non sono consigliate in un progetto embedded.

Per simulare lambde, in C si usano:

- puntatori a funzioni,
- funzioni statiche locali al file,
- closure manuali con struct che raggruppano stato e puntatore a funzione.

### Funzioni avanzate
**Ricorsione**  
Una funzione ricorsiva è una funzione che chiama se stessa per risolvere un problema più grande suddividendolo in sottoproblemi più semplici. Un esempio classico è il calcolo del fattoriale:

```c
int fattoriale(int n) {
    if (n <= 1)
        return 1;
    else
        return n * fattoriale(n - 1);
}
```

In questo caso la funzione continua a richiamarsi fino a quando raggiunge la condizione base (`n <= 1`). La ricorsione è potente, ma in ambiente embedded va usata con estrema cautela perché ogni chiamata aggiunge un nuovo frame nello stack. Poiché lo stack nei microcontrollori è tipicamente molto limitato, un uso eccessivo della ricorsione può portare a stack overflow e comportamenti imprevedibili.  
In sistemi a risorse limitate, è spesso preferibile convertire una ricorsione in un ciclo iterativo.

**Funzioni variadiche**  
Alcune funzioni in C possono accettare un numero variabile di argomenti. L’esempio più noto è `printf`. Per implementarle si utilizza la libreria `<stdarg.h>`, che fornisce i tipi e le macro necessarie per accedere agli argomenti aggiuntivi.

Esempio semplificato di funzione che calcola la media di un numero variabile di interi:

```c
#include <stdarg.h>
#include <stdio.h>

double media(int count, ...) {
    va_list args;
    va_start(args, count);
    int somma = 0;

    for (int i = 0; i < count; i++) {
        somma += va_arg(args, int);
    }

    va_end(args);
    return (double)somma / count;
}
```

L’uso di funzioni variadiche deve essere ponderato: gli argomenti non hanno un tipo esplicito e un errore nella gestione (ad esempio passare un tipo sbagliato) non viene rilevato dal compilatore.

---

**Funzioni inline vs macro**  
Le macro (`#define`) permettono di definire frammenti di codice riutilizzabili che vengono sostituiti dal preprocessore. Ad esempio:

```c
#define QUADRATO(x) ((x) * (x))
```

Il problema è che le macro non hanno controllo sui tipi e possono introdurre errori difficili da individuare (ad esempio `QUADRATO(a+b)` si espande in `((a+b)*(a+b))`, che può avere effetti collaterali inattesi se `x` è un’espressione con side-effect).

Le funzioni `inline` sono un’alternativa molto più sicura:

```c
inline int quadrato(int x) {
    return x * x;
}
```

Il compilatore può sostituire la chiamata con il corpo della funzione (senza overhead di chiamata), mantenendo però il controllo sui tipi e la leggibilità. Per questo motivo, in C moderno si preferisce sempre l’uso di funzioni `inline` rispetto alle macro per operazioni semplici.

---

**Uso di `const` nei parametri**  
La keyword `const` applicata ai parametri di funzione è fondamentale per chiarire le intenzioni e prevenire modifiche indesiderate.

Esempio senza `const`:

```c
void stampa_stringa(char *str);
```

In questo caso non è chiaro se la funzione modifichi o meno il contenuto della stringa.

Con `const`:

```c
void stampa_stringa(const char *str);
```

Ora il compilatore impedirà qualsiasi modifica al contenuto di `str` all’interno della funzione, e chi legge il codice capisce immediatamente che la funzione si limita a leggere la stringa senza alterarla.

L’uso di `const` è particolarmente importante in ambienti embedded, dove la protezione contro modifiche indesiderate alla memoria (ad esempio buffer condivisi o registri di periferiche mappati in memoria) può prevenire errori difficili da diagnosticare.