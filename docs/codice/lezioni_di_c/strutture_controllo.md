## Strutture di Controllo Condizionali: Prendere Decisioni

Spesso, un programma deve comportarsi in modo diverso a seconda delle condizioni. "Se l'utente ha inserito una password corretta, allora accedi. Altrimenti, mostra un errore". Questo è il ruolo delle strutture condizionali.

### `if`, `else if`, `else`

La struttura `if-else` è il modo più basilare per eseguire un blocco di codice solo se una certa condizione è vera.

**La sintassi è:**
```c
if (condizione) {
    // Blocco di codice da eseguire se la condizione è VERA
} else {
    // Blocco di codice da eseguire se la condizione è FALSA (opzionale)
}
```

La `condizione` è un'espressione che viene valutata come "vera" (qualsiasi valore diverso da 0) o "falsa" (0).

```c
#include <stdio.h>

int main() {
    int eta = 20;

    if (eta >= 18) {
        printf("Sei maggiorenne. Puoi entrare.\n");
    } else {
        printf("Sei minorenne. Non puoi entrare.\n");
    }

    return 0;
}
```

### `switch`

Quando devi controllare una singola variabile rispetto a una serie di valori costanti, usare una lunga catena di `if-else if` può diventare goffo. In questi casi, lo `switch` è una soluzione più pulita e spesso più efficiente.

**La sintassi è:**
```c
switch (espressione) {
    case valore1:
        // Codice da eseguire se espressione == valore1
        break; // L'istruzione 'break' è cruciale!
    case valore2:
        // Codice da eseguire se espressione == valore2
        break;
    default:
        // Codice da eseguire se nessun 'case' corrisponde (opzionale)
}
```

**Punti Chiave dello `switch`:**

1. L'`espressione` deve essere di tipo intero (inclusi i `char`).
2. I `valore` dei `case` devono essere costanti.
3. **L'istruzione `break` è fondamentale.** Se omessa, l'esecuzione "cadrà" (fall-through) al `case` successivo, eseguendo anche il suo codice, fino a quando non incontrerà un `break` o la fine dello `switch`. Questo a volte è un comportamento desiderato, ma spesso è fonte di bug.
4. Il blocco `default` cattura tutti i casi non esplicitamente gestiti.

> Attenzione: 
> Lo `switch` possiede un unico contesto, nel senso che una variabile dichiarata in un case, se ridichiarata in un altro genererà un errore di compilazione. In questo caso si utilizzano le parentesi graffe per separare i contesti.

```c
case valore: {
	// codice da eseguire
	break;
}
```

**Esempio:** Un menu.
```c
#include <stdio.h>

int main() {
    char scelta = 'B';

    printf("Menu:\n");
    printf("A - Avvia\n");
    printf("B - Salva\n");
    printf("C - Esci\n");
    printf("La tua scelta: %c\n", scelta);

    switch (scelta) {
        case 'A':
            printf("Programma avviato.\n");
            break;
        case 'B':
            printf("Salvataggio in corso...\n");
            break;
        case 'C':
            printf("Uscita dal programma.\n");
            break;
        default:
            printf("Scelta non valida!\n");
    }
    return 0;
}
```

## Loop (o Cicli): L'Arte della Ripetizione

I loop ci permettono di eseguire un blocco di codice più volte, fino a quando una certa condizione di terminazione non viene soddisfatta. In C, abbiamo tre tipi principali di loop.

### `while`
Il ciclo `while` è il più semplice. Esegue un blocco di codice **fintanto che** la sua condizione rimane vera. La condizione viene controllata **prima** di ogni esecuzione del blocco.

**Sintassi:**
```c
while (condizione) {
    // Blocco di codice da ripetere
    // È importante che qui dentro qualcosa modifichi la condizione,
    // altrimenti si crea un ciclo infinito!
}
```

**Esempio:** Un conto alla rovescia.
```c
#include <stdio.h>

int main() {
    int contatore = 5;

    while (contatore > 0) {
        printf("%d...\n", contatore);
        contatore--; // Decremento il contatore per evitare un loop infinito
    }

    printf("Lancio!\n");
    return 0;
}
```

### `do-while`: Esegui Almeno una Volta

Il `do-while` è una variante del `while`. La sua caratteristica distintiva è che la condizione viene controllata **alla fine** del blocco di codice. Questo garantisce che il codice all'interno del loop venga eseguito **almeno una volta**, indipendentemente dalla condizione.

**Sintassi:**
```c
do {
    // Blocco di codice da ripetere
} while (condizione);
```

**Esempio:** Richiedere un input finché non è valido.
```c
#include <stdio.h>

int main() {
    int numero;

    do {
        printf("Inserisci un numero positivo: ");
        scanf("%d", &numero);

        if (numero <= 0) {
            printf("Errore: il numero deve essere positivo.\n");
        }
    } while (numero <= 0); // Ripeti se il numero non è valido

    printf("Hai inserito il numero valido: %d\n", numero);
    return 0;
}
```

### `for`: Il Ciclo Strutturato

Il ciclo `for` è ideale quando si conosce in anticipo il numero di iterazioni da eseguire (es. "ripeti 10 volte" o "scorri tutti gli elementi di un array"). Condensa in una sola riga l'inizializzazione, la condizione e l'incremento del contatore.

**Sintassi:**
```c
for (inizializzazione; condizione; aggiornamento) {
    // Blocco di codice da ripetere
}
```

- **Inizializzazione:** Eseguita una sola volta, all'inizio del ciclo.
- **Condizione:** Controllata prima di ogni iterazione. Se è falsa, il ciclo termina.
- **Aggiornamento:** Eseguito alla fine di ogni iterazione.

**Esempio:** Stampare i primi 10 numeri e la loro somma.
```c
#include <stdio.h>

int main() {
    int somma = 0;

    for (int i = 1; i <= 10; i++) {
        printf("Numero: %d\n", i);
        somma += i;
    }

    printf("La somma totale è: %d\n", somma);
    return 0;
}
```

Tuttavia, il linguaggio C non obbliga a specificare tutte e tre queste parti. Ognuna di esse è **opzionale**. L'unica cosa che non può mancare sono le due parentesi `()` e i due punti e virgola `;` al loro interno, che agiscono come separatori.
#### 1. Omettere l'Inizializzazione

Se la variabile contatore è già stata inizializzata prima del ciclo, o se il ciclo non dipende da un nuovo contatore, possiamo lasciare vuota la prima parte.

**Quando è utile?**

- Quando il valore iniziale della variabile di controllo dipende da calcoli precedenti.    
- Quando si vuole riutilizzare una variabile esistente.

```c
#include <stdio.h>

int main() {
    int inizio;
    printf("Da che numero vuoi iniziare il conto alla rovescia? ");
    scanf("%d", &inizio);

    // 'inizio' è già stata inizializzata dall'utente.
    // La prima sezione del 'for' è vuota.
    for ( ; inizio >= 0; inizio--) {
        printf("%d...\n", inizio);
    }

    printf("Finito!\n");
    return 0;
}
```

#### 2. Omettere l'Aggiornamento

Possiamo omettere la terza parte se l'aggiornamento della variabile di controllo avviene all'interno del corpo del ciclo.

**Quando è utile?**

- Quando l'aggiornamento non è un semplice incremento/decremento (es. `i++`, `i--`).
- Quando l'aggiornamento deve avvenire solo al verificarsi di una certa condizione all'interno del ciclo.

**Esempio:** Scorrere una lista concatenata fino alla fine. L'aggiornamento consiste nel passare al nodo successivo, un'operazione che si fa all'interno del ciclo.

```c
// Ipotizzando di avere una struttura Nodo e una lista già creata
// struct Nodo { int dato; struct Nodo* next; };
// Nodo* testa; // puntatore al primo nodo

// Il puntatore 'corrente' viene aggiornato nel corpo del ciclo
for (Nodo* corrente = testa; corrente != NULL; ) {
    printf("%d -> ", corrente->dato);
    corrente = corrente->next; // Aggiornamento manuale
}
```

#### 3. Omettere la Condizione

Questa è la situazione più interessante e potenzialmente pericolosa. Se si omette la condizione (la parte centrale), il C la considera **sempre vera**. Questo crea un **ciclo infinito**.

**Quando è utile?** Un ciclo infinito non è sempre un errore. È la base per programmi che devono rimanere in esecuzione continua fino a un intervento esterno, come sistemi operativi, server, o programmi embedded. L'uscita dal ciclo viene gestita internamente con un `break`, un `return` o una chiamata a `exit()`.

**Esempio:** Un menu interattivo che continua a funzionare finché l'utente non sceglie di uscire.

```c
#include <stdio.h>

int main() {
    char scelta;

    for ( ; ; ) { // Ciclo infinito: mancano tutte e tre le parti
        printf("\n--- MENU ---\n");
        printf("1. Stampa 'Ciao'\n");
        printf("2. Stampa 'Mondo'\n");
        printf("3. Esci\n");
        printf("Scelta: ");
        scanf(" %c", &scelta); // Nota lo spazio prima di %c per consumare newline

        if (scelta == '1') {
            printf("Ciao\n");
        } else if (scelta == '2') {
            printf("Mondo\n");
        } else if (scelta == '3') {
            printf("Uscita in corso...\n");
            break; // Uscita controllata dal ciclo infinito
        } else {
            printf("Scelta non valida.\n");
        }
    }

    printf("Programma terminato.\n");
    return 0;
}
```

La forma `for(;;)` è una convenzione molto comune in C per indicare un ciclo infinito intenzionale. È funzionalmente identico a `while(1)`. Viene utilizzato per creare loop infiniti per le task di freertos.

## Controllare i Loop: `break` e `continue`

A volte abbiamo bisogno di un controllo più fine sul comportamento di un loop.

### `break`

L'istruzione `break` (già vista nello `switch`) **interrompe immediatamente** l'esecuzione del loop (`for`, `while`, `do-while`) in cui si trova, e il programma prosegue con l'istruzione successiva al loop.

#### `continue`

L'istruzione `continue` **salta il resto dell'iterazione corrente** e passa direttamente all'iterazione successiva del loop.