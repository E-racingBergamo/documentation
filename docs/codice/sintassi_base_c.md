
# Sintassi base del C

Il linguaggio **C** è alla base dello sviluppo embedded: è semplice, efficiente e permette un controllo diretto sull’hardware.  
In questa sezione vediamo le fondamenta che servono per scrivere e capire il codice della centralina.

---

## Variabili

Una **variabile** è uno spazio di memoria a cui diamo un nome e che contiene un valore.

> ⚠️ Nota per l’embedded:  
> È sempre consigliato evitare l’allocazione dinamica con funzioni come `malloc()` o `alloca()`.  
> Negli ambienti a risorse limitate (come i microcontrollori) queste chiamate possono causare **memory leak**, frammentazione o corruzione della memoria.  
> Meglio usare variabili globali, statiche o allocate nello stack, che garantiscono maggiore stabilità e prevedibilità.


### Dichiarazione
```c
int numero;        // variabile intera
float temperatura; // variabile con virgola
char lettera;      // variabile carattere
````

### Inizializzazione

```c
int numero = 10;
float temperatura = 36.5;
char lettera = 'A';
```

### Regole sui nomi

- Devono iniziare con lettera o `_`
- Possono contenere lettere, numeri e `_`
- Sono **case-sensitive** (`variabile` ≠ `Variabile`)

---

## Tipi di dato principali

### Interi

- `int` → intero base (dimensione dipende dal compilatore, spesso 32 bit)
- `short` → intero corto (16 bit)
- `long` → intero lungo (32 o 64 bit)
- `unsigned` → versione senza segno (solo positivi)

Esempio:

```c
int a = -10;
unsigned int b = 20;
```

### Numeri decimali

- `float` → virgola mobile a 32 bit (circa 6-7 cifre decimali)
- `double` → virgola mobile a 64 bit (circa 15-16 cifre decimali)

Esempio:

```c
float x = 3.14f;
double y = 2.718281828;
```

### Caratteri

- `char` → memorizza un singolo carattere (es. 'A', 'b', '1')
- In realtà è un numero intero (ASCII)

```c
char c = 'A';  // ASCII 65
```

### Booleani

In C puro non esisteva, ma da C99 si può usare, bisogna includere la libreria `stdbool.h`:

```c
#include <stdbool.h>

bool flag = true;
```

> Reminder:
> C $\ne$ C++, la libreria standard C è definita da file chiamati `<libreria>.h` e non `<libreria>` e basta, bisogna mettere il .h 

---

## Costanti

Una costante è un valore che non cambia.

```c
const float PI = 3.14159;
#define MAX_VALORE 100
```

- `const` → variabile costante
- `#define` → macro gestita dal preprocessore

---

## Operatori

### Aritmetici

```c
+   // addizione
-   // sottrazione
*   // moltiplicazione
/   // divisione
%   // resto (solo interi)
```

### Relazionali

```c
==  // uguale
!=  // diverso
>   // maggiore
<   // minore
>=  // maggiore o uguale
<=  // minore o uguale
```

### Logici

```c
&&  // AND logico
||  // OR logico
!   // NOT logico
```

---

### Operatori bitwise

```c
<<  // left shift
>>  // right shift
```

## Puntatori

Un **puntatore** è una variabile che contiene l’indirizzo di memoria di un’altra variabile.  
Sono fondamentali in C perché permettono di accedere direttamente alla memoria, alle periferiche e ai registri.

### Dichiarazione

```c
int x = 10;
int *p = &x;   // p contiene l’indirizzo di x
```

### Utilizzo

```c
printf("Valore di x: %d\n", x);   // stampa 10
printf("Indirizzo di x: %p\n", &x);
printf("Valore tramite puntatore: %d\n", *p); // stampa 10
```

- `&x` → indirizzo della variabile `x`
- `*p` → valore contenuto all’indirizzo puntato da `p`

### Puntatori e array

Un array è strettamente legato ai puntatori.

```c
int numeri[3] = {1, 2, 3};
int *ptr = numeri;

printf("%d\n", *ptr);       // 1
printf("%d\n", *(ptr + 1)); // 2
```

### Puntatori a `char` e stringhe

In C una stringa è un array di `char` terminato da `\0`.

```c
char saluto[] = "Ciao";
char *p = saluto;

printf("%s\n", p);   // stampa "Ciao"
```

---

## Tipi definiti dall’utente

### `struct`

Serve per raggruppare variabili diverse in un’unica entità.

```c
struct Sensore {
    int id;
    float valore;
};

struct Sensore s1 = {1, 23.5};
```

### `typedef`

Permette di creare alias per tipi più leggibili.

```c
typedef unsigned int uint32_t;

uint32_t counter = 100;
```

---

## Riassunto

- Le variabili sono spazi di memoria con un nome e un tipo.
- I tipi principali: `int`, `float`, `double`, `char`, `bool`.
- Gli operatori permettono di fare calcoli e confronti.
- I puntatori gestiscono indirizzi di memoria e sono fondamentali nell’embedded.
- `struct` e `typedef` aiutano a organizzare meglio i dati.