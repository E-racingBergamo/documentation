# Linked list (Lista concatenata)
La **Lista Concatenata** è la SDD più fondamentale. È una sequenza di elementi chiamati **nodi**, dove ogni nodo contiene il dato e un **puntatore** al nodo successivo.

**Struttura di un nodo**
```c
typedef struct Nodo {
    int dato;             // Il dato che il nodo deve memorizzare
    struct Nodo* prossimo; // Puntatore al nodo successivo nella sequenza
} Nodo;

// Il puntatore alla testa della lista
typedef Nodo* Lista;
```

L'elemento `struct Nodo* prossimo;` rende la struttura **autoreferenziale**.
Per gestire la lista, usiamo un puntatore principale, spesso chiamato **testa** (`Lista testa = NULL;`), che punta al primo nodo. Se la lista è vuota, `testa` è `NULL`.

**Inserimento in testa**
Questa è l'operazione di inserimento più semplice e veloce, con complessità temporale costante (O(1)).

```c
Lista inserisciInTesta(Lista testa, int nuovo_dato) {
    // 1. Alloca il nuovo nodo
    Nodo* nuovo_nodo = (Nodo*)malloc(sizeof(Nodo));

    // Controllo allocazione
    if (nuovo_nodo == NULL) {
        // Gestione errore
        return testa;
    }

    // 2. Inserisce il dato
    nuovo_nodo->dato = nuovo_dato;

    // 3. Collega il nuovo nodo alla vecchia testa
    nuovo_nodo->prossimo = testa;

    // 4. Aggiorna la testa al nuovo nodo
    return nuovo_nodo;
}
```

**Eliminazione dalla testa**
Operazione semplice come la recedente, fondamentale per deallocare la memoria.

```c
// Funzione per eliminare il nodo in testa
Lista eliminaTesta(Lista testa) {
    if (testa == NULL) {
        return NULL; // La lista è vuota
    }

    // 1. Salva il puntatore alla testa corrente
    Nodo* temp = testa;

    // 2. Sposta la testa al prossimo nodo
    testa = testa->prossimo;

    // 3. Dealloca il nodo precedente
    free(temp);

    // 4. Restituisce la nuova testa
    return testa;
}
```

**Attraversamento e Stampa (Traversal)**
Per scorrere la lista, si parte dalla testa e si segue il puntatore `prossimo` con un ciclo finché non si arriva a `NULL`, ultimo elemento della lista.

```c
void stampaLista(Lista testa) {
    Nodo* corrente = testa; // Puntatore ausiliario per lo scorrimento
    while (corrente != NULL) {
        printf("%d -> ", corrente->dato);
        corrente = corrente->prossimo; // Passa al nodo successivo
    }
    printf("NULL\n");
}
```

## Derivazioni
Oltre alla lista **semplicemente concatenata** che abbiamo visto ci sono principalmente altre due strutture dati che vengono derivate da questa e sono:

- **Lista doppiamente concatenata**: ogni nodo ha un puntatore al successivo e uno al **precedente**. Permette di attraversare la lista in entrambe le direzioni e di eliminare un nodo in O(1) (una volta trovato) ma occupa più memoria per il doppio puntatore.
- **Lista circolare**: L'ultimo nodo punta al primo, creando un ciclo. Utile per l'implementazione di buffer circolari o round-robin ma bisogna stare attenti a non ciclare all'infinito.

