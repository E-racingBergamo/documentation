# Approfondimento sui puntatori

I puntatori sono uno degli strumenti più potenti e importanti del linguaggio C. Permettono di lavorare direttamente con **indirizzi di memoria**, rendendo possibile l’accesso efficiente a variabili, array, strutture e periferiche hardware. Anche se li abbiamo introdotti nelle sezioni su variabili e funzioni, vale la pena dedicare un capitolo a parte per chiarire tutti i dettagli e i casi particolari.

### Cos’è un puntatore

Quando dichiariamo una variabile, ad esempio `int numero = 10;`, il processore riserva una cella di memoria (o più di una, a seconda del tipo di dato) per memorizzare il valore `10` e associa a questa casella il nome `numero`. L'indirizzo di questa casella è la sua posizione fisica nella memoria.

Un **puntatore** non è altro che una variabile speciale il cui valore non è un dato come un numero o un carattere, ma l'**indirizzo di memoria** di un'altra variabile. Invece di contenere il dato stesso, il puntatore "punta" alla casella di memoria che contiene quel dato.
Dichiarazione:

```c
int x = 10;
int *p = &x; // p contiene l’indirizzo di x

int *puntatore_a_intero;
char *puntatore_a_carattere;
float *puntatore_a_float;
```

- `&x` → (`&` operatore di indirizzo) indirizzo della variabile `x`
- `*p` → (`*` operatore di dereferenziazione o indirezione) valore contenuto all’indirizzo puntato da `p`

È importante sottolineare il fatto che in queste variabili sono a tutti gli effetti salvati degli indirizzi di memoria, quindi il tipo di dato contenuto al loro interno è un `unsigned int` che ha la dimensione del bus del processore. Sulle architetture a 32bit sarà un `uint32` mentre in quelle a 64bit sarà un `uint64`. Dichiarare un tipo diverso serve al processore per capire quanto è grande il dato contenuto alla cella di memoria a cui si sta puntando e come interpretarlo.

> Attenzione: 
> 
> Quando si usano i puntatori, a meno che non si sia assolutamente sicuri che abbiano un valore valido, bisogna controllare che non sia nullo e che sia del tipo corretto rispetto al dato a cui si sta puntando. È anche possibile assegnare `NULL` ad un puntatore, ma bisogna stare attenti a come lo si usa.
### Puntatori e costanti

Esistono varie combinazioni di `const` con i puntatori:

1. **Puntatore a costante**: il contenuto non può essere modificato

```c
const int *p = &x;
*p = 5; // ERRORE
p = &y; // OK
```

2. **Puntatore costante**: il puntatore non può cambiare, ma il contenuto può

```c
int *const p = &x;
*p = 5; // OK
p = &y; // ERRORE
```

3. **Puntatore costante a costante**: niente può essere modificato

```c
const int *const p = &x;
*p = 5; // ERRORE
p = &y; // ERRORE
```

Questa distinzione è fondamentale in embedded per proteggere buffer o registri di periferica.

### Aritmetica dei puntatori

Quando incrementiamo (`++`) un puntatore, non stiamo aggiungendo 1 all'indirizzo di memoria. Stiamo invece spostando il puntatore in avanti della dimensione del tipo di dato a cui punta. Se `p_voti` è un puntatore a `int` e un `int` occupa 4 byte, `p_voti++` aumenterà l'indirizzo di 4, puntando così all'intero successivo.

Le operazioni consentite sono:

- **Incremento/Decremento:** `p++`, `p--`
- **Somma/Sottrazione di un intero:** `p + n`, `p - n`
- **Differenza tra due puntatori:** Se `p1` e `p2` puntano a elementi dello stesso array, `p2 - p1` restituisce il numero di elementi tra di loro.

```c
int a[3] = {10, 20, 30};
int *p = a;

p++;        // ora punta a a[1]
printf("%d\n", *p); // stampa 20
```

L’operatore `+` o `-` muove il puntatore di un numero di elementi, non di byte.  
La sottrazione tra puntatori restituisce il numero di elementi tra loro.

---

### Puntatori e array

In C, c'è una stretta relazione tra puntatori e array. **Il nome di un array, usato in un'espressione, viene convertito in un puntatore al suo primo elemento.**

Questo significa che possiamo usare l'aritmetica dei puntatori per scorrere gli elementi di un array.
Passare un array a una funzione significa passare un puntatore:

```c
void stampa(int arr[], int n) {
    for(int i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
}
```

È equivalente a:

```c
void stampa(int *arr, int n) { ... }
```

### Puntatori a struct

Quando si ha una struttura, è spesso più efficiente passare un **puntatore**:

```c
struct Punto {
    int x;
    int y;
};

void sposta(struct Punto *p, int dx, int dy) {
    p->x += dx;
    p->y += dy;
}
```

- `p->x` è equivalente a `(*p).x`
- I puntatori a struct permettono di risparmiare memoria e tempo di copia.

### Puntatori void

`void *` è un puntatore generico, senza tipo.  
Serve quando vogliamo passare un indirizzo di memoria senza sapere il tipo, ad esempio in buffer generici o callback.

```c
void stampa_generico(void *ptr, char tipo) {
    if(tipo == 'i') printf("%d\n", *(int *)ptr);
    if(tipo == 'f') printf("%f\n", *(float *)ptr);
}
```

---

### Array di puntatori e puntatori a puntatori

È possibile avere array di puntatori:

```c
char *nomi[] = {"Mario", "Luigi", "Peach"};
```

O puntatori a puntatori, utili ad esempio per gestire array dinamici di stringhe:

```c
char **strs;
```

### Passaggio per Riferimento (Simulato)

Di default, il C passa gli argomenti alle funzioni "per valore", creando una copia della variabile. Qualsiasi modifica all'interno della funzione non influisce sulla variabile originale. Passando un puntatore a una funzione, possiamo simulare un "passaggio per riferimento", permettendo alla funzione di modificare la variabile originale.

Esempio riassuntivo:

```c
void incrementa(int *valore) {
    (*valore)++; // Incrementa il valore a cui punta 'valore'
}

int main() {
    int a = 10;
    incrementa(&a); // Passiamo l'indirizzo di 'a'
    printf("Valore di a dopo l'incremento: %d\n", a); // Stampa 11
    return 0;
}
```

### Allocazione Dinamica della Memoria

Fino ad ora, la memoria per le nostre variabili veniva allocata staticamente dal compilatore. I puntatori ci permettono di gestire la memoria **dinamicamente** a runtime, ovvero di richiederne e liberarne blocchi quando ne abbiamo bisogno. Questo è fondamentale per creare strutture dati la cui dimensione non è nota a priori. Le funzioni si trovano nella libreria `<stdlib.h>`.

- **`malloc(size_t size)`**: Alloca un blocco di memoria della dimensione specificata in byte. Restituisce un puntatore `void*` all'inizio del blocco, o `NULL` se fallisce.
    
- **`calloc(size_t num, size_t size)`**: Alloca memoria per un array di `num` elementi, ciascuno di dimensione `size`. La memoria viene inizializzata a zero.
    
- **`realloc(void *ptr, size_t new_size)`**: Ridimensiona un blocco di memoria precedentemente allocato.
    
- **`free(void *ptr)`**: Dealloca (libera) un blocco di memoria precedentemente allocato, rendendolo di nuovo disponibile al sistema.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *array_dinamico;
    int n = 5;

    // Alloca memoria per 5 interi
    array_dinamico = (int*) malloc(n * sizeof(int));

    if (array_dinamico == NULL) {
        printf("Allocazione di memoria fallita!\n");
        return 1;
    }

    // Usa l'array come un normale array
    for (int i = 0; i < n; i++) {
        array_dinamico[i] = i * 10;
        printf("%d ", array_dinamico[i]);
    }
    printf("\n");

    // È FONDAMENTALE liberare la memoria quando non serve più
    free(array_dinamico);
    array_dinamico = NULL; // Buona pratica per evitare "dangling pointers"

    return 0;
}
```

> **Attenzione:** Dimenticare di usare `free()` causa **memory leak** (perdita di memoria), un bug grave in cui il programma consuma memoria senza mai rilasciarla.


### Puntatori a Puntatori

Un puntatore a puntatore è una variabile che contiene l'indirizzo di un altro puntatore. Si dichiara con un doppio asterisco (`**`).
```c
int x = 10;
int *p = &x;
int **pp = &p;

printf("Valore di x: %d\n", x);
printf("Valore tramite p: %d\n", *p);
printf("Valore tramite pp: %d\n", **pp);
```

Sono utili, ad esempio, per creare array di stringhe o per modificare un puntatore all'interno di una funzione.

### Puntatori a Funzioni

Così come le variabili, anche le funzioni risiedono in memoria e hanno un indirizzo. Un puntatore a funzione può memorizzare questo indirizzo, permettendoci di trattare le funzioni come dati: passarle ad altre funzioni, inserirle in array, ecc.

La sintassi è un po' ostica: `tipo_ritorno (*nome_puntatore)(lista_parametri);`

```c
int somma(int a, int b) {
    return a + b;
}

int main() {
    int (*operazione)(int, int); // Dichiara un puntatore a funzione
    operazione = &somma;

    int risultato = operazione(5, 3); // Chiama la funzione 'somma' tramite il puntatore
    printf("Risultato: %d\n", risultato); // Stampa 8
}
```