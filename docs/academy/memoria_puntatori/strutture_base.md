
# Strutture dati minime

Nel linguaggio C le strutture dati fondamentali che si usano quasi ovunque sono gli **array**, le **struct** e le **enum**. Conoscerle a fondo è essenziale per scrivere codice chiaro, efficiente e comprensibile.

---

### Array

Un array è una sequenza di elementi dello stesso tipo, memorizzati in celle di memoria contigue.  
Quando si dichiara un array, si deve indicare il tipo degli elementi e la dimensione:

```c
int valori[10]; // array di 10 interi
````

Questo significa che la variabile `valori` contiene 10 interi, indicizzati da `0` a `9`. L’indice deve sempre essere compreso tra 0 e dimensione-1. Accedere a un indice fuori dai limiti produce un comportamento indefinito, cioè errori difficili da individuare.

Gli array possono essere **inizializzati**:

```c
int a[5] = {1, 2, 3, 4, 5};
int b[5] = {0}; // tutti inizializzati a zero
```

Se si forniscono meno valori rispetto alla dimensione, i restanti vengono messi a zero. È possibile anche lasciare vuota la dimensione quando i valori sono noti a compilazione:

```c
int c[] = {10, 20, 30}; // dimensione 3 dedotta dal compilatore
```

Gli array di caratteri sono spesso usati per rappresentare stringhe terminate dal carattere nullo `'\0'`:

```c
char nome[10] = "ciao"; // occuperà 5 celle: 'c','i','a','o','\0'
```

Negli ambienti embedded gli array vengono utilizzati anche come **buffer** per la comunicazione (es. pacchetti CAN, UART). In questi casi è fondamentale rispettare sempre la dimensione massima per evitare corruzione di memoria.

Un array in C si comporta quasi sempre come un **puntatore al suo primo elemento**. Questo significa che se passo un array a una funzione, in realtà passo l’indirizzo della prima cella, non una copia dell’intero array:

```c
void stampa(int arr[], int n) {
    for(int i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
}
```

---

### Struct

Una `struct` è una struttura che permette di raggruppare variabili di tipi diversi sotto un unico nome. È molto utile per rappresentare entità complesse:

```c
struct Punto {
    int x;
    int y;
};
```

Ora è possibile dichiarare variabili di questo tipo:

```c
struct Punto p1 = {10, 20};
```

Ogni campo si accede con l’operatore `.`:

```c
printf("%d", p1.x);
```

Le struct possono contenere anche array o altre struct:

```c
struct Rettangolo {
    struct Punto vertici[4];
};
```

Quando si lavora con puntatori a struct, si usa l’operatore `->`:

```c
struct Punto *pp = &p1;
pp->x = 5; // equivalente a (*pp).x
```

È possibile usare `typedef` per evitare di scrivere ogni volta `struct`:

```c
typedef struct {
    int x;
    int y;
} Punto;

Punto p2 = {3, 4};
```

In embedded le struct sono spesso usate per definire i registri delle periferiche. In questo caso è importante considerare il **padding**: il compilatore può inserire byte di allineamento tra i campi per rispettare i vincoli dell’architettura. Questo significa che due struct con gli stessi campi ma ordine diverso possono avere dimensioni diverse. Per controllare queste situazioni si usano attributi come `__attribute__((packed))` o le opzioni del compilatore.

Le struct possono anche essere annidate e usate come contenitori per dati condivisi tra task in un sistema operativo real-time. È una buona pratica abbinarle a mutex o code quando più parti del programma devono accedere alle stesse informazioni.

---

### Enum

Una `enum` definisce un insieme di valori simbolici associati a numeri interi. È utile per rendere più leggibile il codice e per rappresentare stati, modalità o codici di errore.

```c
enum Stato {
    IDLE,
    RUN,
    ERROR
};
```

Di default il primo valore parte da 0 e gli altri crescono di 1. Nell’esempio, `IDLE = 0`, `RUN = 1`, `ERROR = 2`. È possibile assegnare valori arbitrari:

```c
enum Comando {
    START = 10,
    STOP = 20,
    RESET = 30
};
```

È consigliato usare `typedef` per semplificare:

```c
typedef enum {
    LED_OFF = 0,
    LED_ON = 1
} LedState;
```

Le enum migliorano la leggibilità:

```c
LedState stato = LED_OFF;

if (stato == LED_ON) {
    // accendi il LED
}
```

In embedded le enum sono molto utili per definire le **macchine a stati finiti**. Ogni stato della FSM corrisponde a un valore della enum, rendendo il codice più chiaro e meno soggetto a errori rispetto all’uso di costanti numeriche.


### Union

Una `union` è simile a una `struct`, ma con una differenza fondamentale: **tutti i campi condividono lo stesso spazio di memoria**.  
Questo significa che una union occupa tanta memoria quanta ne richiede il campo più grande, e tutti i membri si sovrappongono.

Esempio di dichiarazione:

```c
union Valore {
    uint32_t intero;
    float reale;
    uint8_t byte[4];
};
````

In questo caso `union Valore` occupa 4 byte, perché il campo più grande (`uint32_t` e `float`) è di 4 byte.  
Se assegniamo un valore a `intero`, possiamo leggere la sua rappresentazione binaria attraverso l’array `byte`:

```c
union Valore v;
v.intero = 0x12345678;

printf("%02X %02X %02X %02X\n", v.byte[0], v.byte[1], v.byte[2], v.byte[3]);
```

Il risultato dipenderà dall’**endianness** del processore (little endian o big endian), perché determina l’ordine dei byte in memoria.

Le union sono molto potenti, ma bisogna usarle con attenzione: scrivere in un campo e leggere da un altro non sempre è portabile al 100% secondo lo standard C, anche se in pratica nei sistemi embedded è una tecnica comune.

**Esempi di utilizzo tipico nelle applicazioni embedded:**

- Interpretare un pacchetto ricevuto su bus di comunicazione sia come array di byte che come valori numerici.
- Definire registri hardware in modo che un singolo registro possa essere letto intero oppure per campi più piccoli.
- Effettuare “type punning” (riutilizzare lo stesso blocco di memoria come tipi diversi) per ridurre l’uso di memoria.

**Differenze rispetto a struct:**

- In una `struct` ogni campo ha il suo spazio separato, e la dimensione totale è almeno la somma delle dimensioni dei campi (più eventuale padding).
- In una `union` tutti i campi condividono lo stesso spazio, e la dimensione totale è uguale a quella del campo più grande.