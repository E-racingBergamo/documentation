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

I puntatori possono essere incrementati o decrementati, ma l’unità di incremento dipende dal tipo a cui puntano:

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

Un array si comporta come un puntatore al primo elemento.  
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

---

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
    

---

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

---

### Puntatori a funzioni

I puntatori possono memorizzare indirizzi di funzioni, permettendo di chiamarle in modo dinamico.  
Abbiamo già visto esempi nel capitolo su funzioni.

Esempio riassuntivo:

```c
typedef int (*Operazione)(int, int);

int somma(int a, int b) { return a + b; }

Operazione op = somma;
int risultato = op(3, 4); // richiama somma(3,4)
```

---

### Considerazioni finali

- I puntatori sono essenziali in C embedded per accesso diretto alla memoria, gestione di buffer e interfacciamento con driver.
    
- Comprendere `const`, `void *`, aritmetica dei puntatori e puntatori a struct/funzioni è fondamentale per evitare bug difficili da trovare.
    
- Una volta chiari questi concetti, il codice diventa più modulare, efficiente e sicuro.