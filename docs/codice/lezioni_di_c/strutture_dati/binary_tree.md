# Binary tree
## Introduzione e terminologia

Mentre le liste sono strutture lineari, gli **Alberi** sono strutture gerarchiche. L'**Albero Binario** è una delle strutture dinamiche non lineari più importanti per la ricerca, l'inserimento e la cancellazione efficienti.

Un **Albero Binario** è una struttura dati gerarchica composta da nodi. La caratteristica fondamentale è il vincolo strutturale: **ogni nodo può avere al massimo due figli**, storicamente chiamati **figlio sinistro** e **figlio destro**.

Il nodo di partenza è la **radice** (root), che non ha antenati. I nodi senza figli sono chiamati **foglie** (leaves). La sequenza di nodi da un punto all'altro si chiama **cammino**, e la lunghezza del cammino dalla radice a un nodo è la sua **profondità** (depth). L'altezza dell'albero è la profondità massima di qualsiasi nodo foglia.


Termini fondamentali:

- **Nodo**: elemento con valore e puntatori a figli.
- **Radice (root)**: nodo senza genitore.
- **Foglia (leaf)**: nodo senza figli.   
- **Altezza** di un nodo: lunghezza del percorso più lungo fino a una foglia (numero di archi).
- **Profondità/level**: distanza dalla radice (root level = 0).
- **Sottalbero**: albero formato da un nodo e tutti i suoi discendenti.

Forme speciali:

- **Albero pieno (full/proper)**: ogni nodo ha 0 o 2 figli.
- **Albero perfetto (perfect)**: pieno e tutti i livelli sono completi; ha `2^{h+1}-1` nodi (h = altezza).
- **Albero completo (complete)**: tutti i livelli compresi (tranne l'ultimo) sono pieni, e i nodi dell'ultimo sono a sinistra.
- **Albero bilanciato**: altezza `O(log n)`; definizione può variare (AVL, Red‑Black, B‑Tree per multiway ecc.).

## Tipologie di alberi binari

- **Binary Tree**: generico.
- **Binary Search Tree (BST)**: per ogni nodo, `left->key < node->key < right->key` (chiave univoca o gestire duplicati).
- **AVL tree**: BST auto‑bilanciato con fattore di bilanciamento `height(left)-height(right)` ∈ {-1,0,1}; rotazioni per mantenere bilanciamento.
- **Red‑Black tree**: BST bilanciato che garantisce altezza `O(log n)` usando colori e rotazioni.
- **Splay tree**: BST auto‑adattivo che porta l'ultimo elemento accesso in radice (good for locality).
- **Treap / Cartesian tree**: BST con priorità casuali (heap property) — utile per implementazioni probabilisticamente bilanciate.
- **Binary Heap**: struttura completa rappresentata con array; utile per priority queue; non è un BST.
- **Threaded binary tree**: usa puntatori "thread" al successore/predecessore inorder per traversal senza stack/ricorsione.
- **Albero di espressione**: nodi interni sono operatori, foglie sono operandi; valutazione postfix/infix.
- **Segment tree / interval tree**: alberi binari (implicitamente) per query su intervalli (range sum/min/max) con aggiornamenti.

## Traversal: algoritmi e implementazioni

Quando si parla di **visita di un albero binario**, ci si riferisce all’ordine con cui vengono “toccati” i nodi.  
Esistono due grandi famiglie: la visita **in profondità (DFS, Depth-First Search)** e la visita **in ampiezza (BFS, Breadth-First Search)**.

**Le tre visite DFS classiche**

1. **Preorder**  
    Si parte sempre dal nodo corrente (la radice, se è la prima volta), lo si visita subito e poi si scende prima nel sottoalbero sinistro e poi nel destro.  
    È come se il padre avesse la priorità: lo guardi subito, prima di interessarti ai figli.  
    Questo tipo di visita è molto utile, ad esempio, se vuoi salvare o clonare la struttura dell’albero, perché mantieni prima l’informazione del nodo e poi dei suoi discendenti.
    
2. **Inorder**  
    In questo caso l’ordine è: prima il sottoalbero sinistro, poi il nodo corrente e infine il sottoalbero destro.  
    È la visita più importante quando si parla di alberi binari di ricerca (BST), perché restituisce gli elementi **in ordine crescente**.  
    È come leggere un libro da sinistra a destra: prima quello che c’è a sinistra, poi il centro, poi la parte destra.
    
3. **Postorder**  
    Qui invece si visita prima il sottoalbero sinistro, poi il destro e solo per ultimo il nodo corrente.  
    È come dire: “prima mi occupo dei figli e solo dopo del padre”.  
    Questo ordine è molto comodo quando bisogna cancellare un albero dalla memoria, perché si eliminano prima i nodi più in basso e solo alla fine le radici. Allo stesso modo, viene usato negli alberi di espressioni per calcolare il valore: prima si calcolano i termini elementari e solo alla fine l’operazione principale.

**BFS — Level Order**

Diverso è il caso della visita **in ampiezza (BFS)**, detta anche “per livelli”.  
Qui non si scende in profondità subito, ma si procede un livello alla volta: prima la radice, poi tutti i figli della radice, poi i figli di quei figli, e così via.  
È un po’ come leggere l’albero riga per riga, da sinistra a destra.  
Questa modalità viene spesso usata quando serve dare priorità alla distanza dalla radice, ad esempio per calcolare la profondità minima o per trovare il cammino più breve in alberi o grafi.
### Traversal ricorsivi (C)

```c
#include <stdio.h>

typedef struct Node {
    int key;
    struct Node *left, *right;
} Node;

void preorder(Node* r){
    if(!r) return;
    printf("%d ", r->key);
    preorder(r->left);
    preorder(r->right);
}

void inorder(Node* r){
    if(!r) return;
    inorder(r->left);
    printf("%d ", r->key);
    inorder(r->right);
}

void postorder(Node* r){
    if(!r) return;
    postorder(r->left);
    postorder(r->right);
    printf("%d ", r->key);
}
```

> I traversal ricorsivi sono semplici ma usano stack di chiamate `O(h)` (altezza). In sistemi embedded bisogna fare attenzione alla profondità dello stack.

### Traversal iterativi con stack

```c
#include <stdlib.h>

typedef struct StackNode {
	Node* t;
	struct StackNode* next;
} StackNode;

void push(StackNode** top, Node* t) { 
	StackNode* n = malloc(sizeof(StackNode));
	n->t=t;
	n->next=*top;
	*top=n;
}

Node* pop(StackNode** top) {
	if(!*top) return NULL;
	StackNode* n=*top;
	Node* t=n->t;
	*top = n->next;
	free(n);
	return t;
}

int empty(StackNode* top) {
	return top==NULL;
}

void inorder_iter(Node* root){
    StackNode* st = NULL;
    Node* cur = root;
    while(cur || !empty(st)){
        while(cur){ push(&st, cur); cur = cur->left; }
        cur = pop(&st);
        printf("%d ", cur->key);
        cur = cur->right;
    }
}
```

### Morris Inorder (O(1) space)

Idea: usare collegamenti temporanei al predecessore inorder.

```c
void inorder_morris(Node* root){
    Node* cur = root;
    while(cur){
        if(!cur->left){
            printf("%d ", cur->key);
            cur = cur->right;
        } else {
            Node* pred = cur->left;
            while(pred->right && pred->right != cur) pred = pred->right;
            if(!pred->right){
                pred->right = cur; // thread
                cur = cur->left;
            } else {
                pred->right = NULL; // restore
                printf("%d ", cur->key);
                cur = cur->right;
            }
        }
    }
}
```

> Morris è utile quando la memoria è critica; modifica temporaneamente la struttura e la ripristina.

### Level‑order (BFS)

Usa una coda (queue):

```c
#include <stdlib.h>

typedef struct QNode {
  Node* t;
  struct QNode* next;
} QNode;

typedef struct Queue {
  QNode *head, *tail;
} Queue;

void qpush(Queue* q, Node* t) {
  QNode* n = malloc(sizeof(QNode));
  n->t = t;
  n->next = NULL;
  if (!q->tail)
    q->head = q->tail = n;
  else {
    q->tail->next = n;
    q->tail = n;
  }
}

Node* qpop(Queue* q) {
  if (!q->head) return NULL;
  QNode* n = q->head;
  Node* t = n->t;
  q->head = n->next;
  if (!q->head) q->tail = NULL;
  free(n);
  return t;
}

int qempty(Queue* q) { return q->head == NULL; }

void level_order(Node* root) {
  if (!root) return;
  Queue q = {0};
  qpush(&q, root);
  while (!qempty(&q)) {
    Node* n = qpop(&q);
    printf("%d ", n->key);
    if (n->left) qpush(&q, n->left);
    if (n->right) qpush(&q, n->right);
  }
}
```

---

## BST (Binary Search Tree)

### Proprietà e complessità

Le operazioni fondamentali su un albero binario di ricerca, cioè **ricerca, inserimento e cancellazione**, hanno una complessità che dipende fortemente dalla forma dell’albero.  
Nel caso “ideale”, quando l’albero è **bilanciato**, ogni livello è riempito in modo abbastanza uniforme e quindi l’altezza dell’albero cresce come il logaritmo del numero di nodi. Questo significa che in media la ricerca, l’inserimento e la cancellazione richiedono un numero di passi proporzionale a `log n`, dove `n` è il numero di elementi.

Tuttavia, il BST di per sé **non garantisce il bilanciamento**. Se, ad esempio, i dati vengono inseriti in ordine crescente, l’albero tende a degenerare in una semplice linked list: ogni nodo ha solo il figlio destro, e l’altezza diventa `n`. In questa situazione, tutte le operazioni fondamentali (ricerca, inserimento, cancellazione) peggiorano fino a richiedere tempo lineare `O(n)`.
## Implementazione base in C (ricorsiva)

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
  int key;
  struct Node *left, *right;
} Node;

Node* new_node(int key) {
  Node* n = malloc(sizeof(Node));
  if (!n) return NULL;
  n->key = key;
  n->left = n->right = NULL;
  return n;
}

Node* bst_insert(Node* root, int key) {
  if (!root) return new_node(key);
  if (key < root->key)
    root->left = bst_insert(root->left, key);
  else if (key > root->key)
    root->right = bst_insert(root->right, key);
  // se permetti duplicati, scegli una policy
  return root;
}

Node* bst_search(Node* root, int key) {
  while (root && root->key != key)
    root = (key < root->key) ? root->left : root->right;
  return root;
}

Node* bst_find_min(Node* root) {
  while (root && root->left) root = root->left;
  return root;
}

Node* bst_delete(Node* root, int key) {
  if (!root) return NULL;
  if (key < root->key)
    root->left = bst_delete(root->left, key);
  else if (key > root->key)
    root->right = bst_delete(root->right, key);
  else {
    if (!root->left) {
      Node* r = root->right;
      free(root);
      return r;
    } else if (!root->right) {
      Node* l = root->left;
      free(root);
      return l;
    } else {
      Node* succ = bst_find_min(root->right);
      root->key = succ->key;
      root->right = bst_delete(root->right, succ->key);
    }
  }
  return root;
}
```

> `bst_delete` gestisce i tre casi (0, 1, 2 figli). L'approccio mostrato copia la chiave del successore e cancella il successore.

---

# Bilanciamento: AVL e Red‑Black

## AVL (dettagli pratici)
Gli **alberi AVL** sono alberi binari di ricerca che mantengono sempre un buon livello di equilibrio. Per fare questo, ad ogni nodo viene associato un numero chiamato **fattore di bilanciamento (Balance Factor, BF)**, che non è altro che la differenza tra l’altezza del sottoalbero sinistro e quella del sottoalbero destro.  
Finché questo valore rimane compreso tra −1 e +1, l’albero è considerato bilanciato. Se invece un’operazione di inserimento o cancellazione fa sì che in qualche nodo il fattore di bilanciamento esca da questo intervallo, bisogna **riequilibrare la struttura** tramite una o più **rotazioni**.

Le rotazioni sono semplici ristrutturazioni locali dell’albero che ne modificano la forma ma non l’ordinamento dei dati. Esistono quattro casi principali:

- Se l’albero è troppo “pesante” a sinistra (BF > 1) perché il nuovo nodo è stato inserito a sua volta nel ramo sinistro del figlio sinistro, si esegue una **rotazione a destra**. Questo caso è chiamato **LL**.
    
- Al contrario, se è troppo sbilanciato a destra (BF < −1) e il nuovo nodo si trova nel ramo destro del figlio destro, la soluzione è una **rotazione a sinistra**: il caso **RR**.
    
- Le cose diventano leggermente più complesse se lo sbilanciamento deriva da un inserimento “incrociato”. Se il nodo è andato a finire nel ramo destro del figlio sinistro, prima si fa una rotazione a sinistra sul figlio sinistro e poi una rotazione a destra sulla radice: questo è il caso **LR**.
    
- Infine, se il nodo è nel ramo sinistro del figlio destro, la correzione è simmetrica: prima una rotazione a destra sul figlio destro e poi una rotazione a sinistra sulla radice, cioè il caso **RL**.
    

Con queste quattro operazioni si riesce a riportare sempre l’albero entro i limiti di bilanciamento. In questo modo, gli alberi AVL garantiscono che le operazioni fondamentali come ricerca, inserimento e cancellazione restino sempre efficienti, con complessità `O(log n)` anche nei casi peggiori.

### Codice essenziale (AVL insert + rotazioni)

```c
#include <stdlib.h>
#include <stdio.h>

typedef struct AVLNode { int key; int height; struct AVLNode *left, *right; } AVLNode;

int max(int a,int b){ return a>b?a:b; }
int height(AVLNode* n){ return n ? n->height : 0; }

AVLNode* new_avl_node(int key){ AVLNode* n = malloc(sizeof(AVLNode)); if(!n) return NULL; n->key=key; n->left=n->right=NULL; n->height=1; return n; }

AVLNode* right_rotate(AVLNode* y){
    AVLNode* x = y->left;
    AVLNode* T2 = x->right;
    x->right = y;
    y->left = T2;
    y->height = 1 + max(height(y->left), height(y->right));
    x->height = 1 + max(height(x->left), height(x->right));
    return x; // nuova radice
}

AVLNode* left_rotate(AVLNode* x){
    AVLNode* y = x->right;
    AVLNode* T2 = y->left;
    y->left = x;
    x->right = T2;
    x->height = 1 + max(height(x->left), height(x->right));
    y->height = 1 + max(height(y->left), height(y->right));
    return y;
}

AVLNode* avl_insert(AVLNode* node, int key){
    if(!node) return new_avl_node(key);
    if(key < node->key) node->left = avl_insert(node->left, key);
    else if(key > node->key) node->right = avl_insert(node->right, key);
    else return node; // duplicati non gestiti

    node->height = 1 + max(height(node->left), height(node->right));
    int balance = height(node->left) - height(node->right);

    // LL
    if(balance > 1 && key < node->left->key) return right_rotate(node);
    // RR
    if(balance < -1 && key > node->right->key) return left_rotate(node);
    // LR
    if(balance > 1 && key > node->left->key){ node->left = left_rotate(node->left); return right_rotate(node); }
    // RL
    if(balance < -1 && key < node->right->key){ node->right = right_rotate(node->right); return left_rotate(node); }

    return node;
}
```

> L'implementazione della cancellazione in AVL è più lunga (si applicano rotazioni dopo la cancellazione per ribilanciare). L'idea è la stessa: dopo ogni modifica si aggiorna `height` e si controlla il `balance`.

## 5.2 Red‑Black (concetti)
Gli **alberi Red-Black** sono un’altra famiglia di alberi binari di ricerca bilanciati. L’idea alla base è un po’ diversa rispetto agli AVL: invece di tenere sotto controllo con precisione l’altezza di ogni nodo, si assegna a ciascun nodo un **colore**, che può essere rosso o nero, e si impongono alcune regole che garantiscono che l’albero resti “abbastanza” bilanciato.

Le regole fondamentali sono queste:

- Ogni nodo deve essere colorato in rosso o in nero.
    
- La radice dell’albero è sempre nera.
    
- Le foglie vuote (i nodi NIL) vengono considerate nere.
    
- Non possono esserci due nodi rossi consecutivi: se un nodo è rosso, allora i suoi figli devono per forza essere neri.
    
- Qualunque percorso che parte da un nodo e arriva a una foglia NIL deve attraversare lo stesso numero di nodi neri. Questo valore prende il nome di **black height** e garantisce che i cammini nell’albero non possano divergere troppo in lunghezza.
    

Grazie a queste proprietà, l’albero Red-Black resta sempre bilanciato in modo “lasco”: non perfettamente come un AVL, ma comunque con altezza proporzionale a `log n`. Questo significa che operazioni come inserimento, ricerca e cancellazione restano sempre efficienti.

Naturalmente, mantenere queste regole non è automatico. Dopo un’operazione di inserimento o di cancellazione può capitare che alcune proprietà vengano violate. In questi casi bisogna **aggiustare** la struttura con una combinazione di **rotazioni** (come negli AVL) e cambi di colore. L’algoritmo è un po’ più articolato rispetto agli AVL, ma la garanzia è che dopo pochi passi l’albero torna valido.

Proprio per la loro efficienza e stabilità, gli alberi Red-Black vengono usati molto spesso in librerie e sistemi reali. Ad esempio, le strutture dati `map` e `set` della libreria standard del C++ sono basate su alberi Red-Black, così come molte implementazioni interne nei kernel dei sistemi operativi.

---

# Binary Heap (array) e Heapsort

## Rappresentazione
Un **binary heap** è una struttura dati molto efficiente che si può rappresentare in modo estremamente semplice utilizzando un **array**. L’idea è che l’albero binario non venga memorizzato tramite puntatori, ma piuttosto sfruttando la posizione degli elementi nell’array per ricostruire i legami padre-figlio.

Se consideriamo un array `A[0..n-1]`, le relazioni sono fisse:

- il padre di un nodo che si trova in posizione `i` si calcola come `(i - 1) / 2`;
- il figlio sinistro è in `2*i + 1`;
- il figlio destro è in `2*i + 2`.

In questo modo, non serve memorizzare puntatori espliciti: basta l’indice.

Ci sono due varianti principali di heap: **min-heap** e **max-heap**. Nel **max-heap**, che è quello più spesso utilizzato, la proprietà da rispettare è che ogni nodo sia maggiore o uguale ai propri figli. In altre parole, il valore nella posizione del padre deve sempre essere almeno grande quanto quello dei suoi figli. Grazie a questa regola, l’elemento massimo dell’intera struttura si trova sempre nella radice, cioè nella cella `A[0]`. Nel caso del **min-heap**, invece, vale la proprietà opposta: il minimo si trova sempre in cima.

Questa rappresentazione permette di gestire in maniera molto veloce operazioni come inserimento, estrazione del massimo/minimo e costruzione di un heap a partire da un array. Inoltre, proprio grazie alla sua semplicità, l’heap binario è alla base di un algoritmo di ordinamento molto famoso: l’**heapsort**.
## Operazioni principali
Le operazioni fondamentali che permettono di lavorare con un **binary heap** sono poche ma molto potenti.

Quando si vuole **inserire** un nuovo elemento nell’heap, lo si mette inizialmente in fondo all’array, cioè nella prima posizione libera. A questo punto però l’elemento potrebbe violare la proprietà dell’heap (ad esempio, in un max-heap potrebbe essere più grande del suo padre). Per ristabilire l’ordine, si esegue l’operazione chiamata **sift up**: in pratica, si confronta il nodo con il suo padre e, se è maggiore, i due vengono scambiati. Questo controllo continua a salire lungo l’albero finché la proprietà dell’heap non è di nuovo rispettata.

L’operazione opposta avviene quando si vuole **estrarre l’elemento massimo** (nel max-heap). In questo caso si prende la radice, cioè `A[0]`, che contiene sempre il valore più grande. Per non lasciare un buco, l’ultimo elemento dell’array viene spostato provvisoriamente in cima. A questo punto però è probabile che la proprietà dell’heap non sia rispettata, quindi bisogna farlo “scendere” nel posto giusto. Questo si fa con l’operazione di **sift down**: si confronta il nodo con i figli e, se uno dei figli è più grande, si scambia con quello più grande dei due. L’operazione continua finché il nodo non si trova in una posizione che rispetta le regole.

Infine, c’è l’operazione di **build heap**, che costruisce un heap partendo da un array arbitrario di `n` elementi. Un metodo ingenuo sarebbe inserire gli elementi uno alla volta e fare `sift up` ogni volta, ma questo porta a una complessità `O(n log n)`. In realtà, esiste un algoritmo più efficiente: si parte dal basso e si applica `sift down` solo ai nodi interni (cioè quelli che hanno figli). In questo modo si riesce a trasformare l’intero array in un heap in tempo lineare `O(n)`.

## Codice e heapsort

```c
#include <stdio.h>
#include <stdlib.h>

void swap(int *a,int *b){ int t=*a; *a=*b; *b=t; }

void sift_down(int *A, int n, int i){
    int largest = i;
    int l = 2*i+1, r = 2*i+2;
    if(l < n && A[l] > A[largest]) largest = l;
    if(r < n && A[r] > A[largest]) largest = r;
    if(largest != i){ swap(&A[i], &A[largest]); sift_down(A, n, largest); }
}

void build_heap(int *A, int n){
    for(int i = n/2 - 1; i >= 0; --i) sift_down(A, n, i);
}

void heapsort(int *A, int n){
    build_heap(A, n);
    for(int i = n-1; i > 0; --i){ swap(&A[0], &A[i]); sift_down(A, i, 0); }
}
```

> `heapsort` è `O(n log n)` worst/average; uso in‑place, stabile? non stabile.

# Tree sort

### Cos’è il TreeSort

Il **TreeSort** è un algoritmo di ordinamento che sfrutta un **albero binario di ricerca (BST)**.  
L’idea è molto semplice:

1. Si parte da una sequenza di numeri non ordinati.
2. Si inseriscono tutti gli elementi in un BST.
    - L’inserimento segue la regola del BST: i valori minori vanno a sinistra, i maggiori a destra.
3. Una volta costruito l’albero, si esegue una **visita inorder**.
    - Ricorda che in un BST la visita inorder restituisce sempre gli elementi in ordine crescente.

Il risultato finale sarà la sequenza ordinata.
### Complessità

- **Caso medio (albero bilanciato)**:
    - Inserire `n` elementi costa `O(n log n)` (perché ogni inserimento richiede log n).
    - La visita inorder costa `O(n)`.
    - Complessivamente: **O(n log n)**.
- **Caso peggiore (albero degenerato)**:
    - Se i dati sono già ordinati e si usa un BST semplice (non bilanciato), l’albero diventa una lista.
    - Ogni inserimento costa `O(n)` → tempo totale `O(n²)`.

Per questo, in pratica il TreeSort è efficiente solo se si utilizza un **albero bilanciato** (AVL, Red-Black). Con un AVL, ad esempio, si garantisce sempre `O(n log n)`.

```c
#include <stdio.h>
#include <stdlib.h>

// Nodo dell’albero
typedef struct Node {
    int key;
    struct Node *left, *right;
} Node;

// Creazione nuovo nodo
Node* newNode(int key) {
    Node* node = (Node*) malloc(sizeof(Node));
    node->key = key;
    node->left = node->right = NULL;
    return node;
}

// Inserimento nel BST
Node* insert(Node* root, int key) {
    if (root == NULL) return newNode(key);
    if (key < root->key)
        root->left = insert(root->left, key);
    else
        root->right = insert(root->right, key);
    return root;
}

// Visita inorder (stampa ordinata)
void inorder(Node* root) {
    if (root != NULL) {
        inorder(root->left);
        printf("%d ", root->key);
        inorder(root->right);
    }
}

// TreeSort: costruisce BST e stampa ordinato
void treeSort(int arr[], int n) {
    Node* root = NULL;

    // Costruzione albero
    for (int i = 0; i < n; i++) {
        root = insert(root, arr[i]);
    }

    // Stampa ordinata con inorder
    inorder(root);
    printf("\n");
}

int main() {
    int arr[] = {5, 3, 7, 2, 8, 1, 4};
    int n = sizeof(arr) / sizeof(arr[0]);

    printf("Array ordinato: ");
    treeSort(arr, n);

    return 0;
}
```

TreeSort è elegante perché sfrutta direttamente le proprietà del BST. Tuttavia, nella pratica viene usato poco rispetto a **QuickSort** o **HeapSort**, proprio perché senza bilanciamento rischia di degradare a `O(n²)`.

## Alberi Threaded

Un problema degli alberi binari tradizionali è che spesso contengono molti **puntatori NULL** (quando un nodo non ha figlio sinistro o destro). Un **threaded binary tree** sfrutta questi puntatori vuoti per contenere invece riferimenti speciali che collegano i nodi tra loro in modo da facilitare la visita **inorder**.

In pratica, se un nodo non ha figlio destro, il puntatore right viene usato per collegarlo direttamente al “successore inorder”. Allo stesso modo, se non ha figlio sinistro, left può puntare al “predecessore inorder”.  
Grazie a questi collegamenti (“threads”), è possibile attraversare l’albero **senza ricorsione né stack**, scorrendo i nodi in ordine.

Esempio minimo in C:
```c
typedef struct Node {
    int key;
    struct Node *left, *right;
    int ltag, rtag; // 0 = figlio, 1 = thread
} Node;
```

Il traversal inorder diventa una semplice camminata seguendo i thread quando non ci sono figli.

### Alberi di espressione

Un **expression tree** rappresenta un’espressione aritmetica come albero binario:
- Le **foglie** contengono operandi (numeri o variabili).
- I **nodi interni** contengono operatori (+, -, *, /).

Esempio: l’espressione `(3 + 5) * (2 - 1)` si rappresenta così:
- Radice = `*`
- Figlio sinistro = `+` con figli `3` e `5`
- Figlio destro = `-` con figli `2` e `1`

La valutazione si fa con una visita **postorder**: prima si calcolano i sottoalberi, poi si applica l’operatore.

Snippet in C per valutazione:
```c
typedef struct Node {
    char op; // operatore o '\0' se foglia
    int value;
    struct Node *left, *right;
} Node;

int eval(Node* root) {
    if (root->op == '\0') return root->value; // foglia
    int l = eval(root->left);
    int r = eval(root->right);
    switch(root->op) {
        case '+': return l + r;
        case '-': return l - r;
        case '*': return l * r;
        case '/': return l / r;
    }
    return 0;
}
```

### ## Segment Tree

Il **segment tree** è un albero che permette di rispondere in modo molto veloce a query su intervalli (es. somma, minimo, massimo su un sottoarray).  
L’idea è rappresentare un array `A[0..n-1]` in un albero binario dove ogni nodo memorizza l’informazione aggregata di un segmento.

- La radice rappresenta l’intero array.
- Ogni nodo viene diviso in due: sinistra = prima metà, destra = seconda metà.
- Le foglie corrispondono agli elementi singoli.

Con questa struttura si ottengono:
- **Costruzione**: `O(n)`
- **Query su intervallo**: `O(log n)`
- **Aggiornamento**: `O(log n)`

Esempio in C per segment tree che calcola somme:
```c
#define MAXN 1000
int seg[4*MAXN];

void build(int arr[], int idx, int l, int r) {
    if (l == r) {
        seg[idx] = arr[l];
        return;
    }
    int mid = (l + r) / 2;
    build(arr, 2*idx, l, mid);
    build(arr, 2*idx+1, mid+1, r);
    seg[idx] = seg[2*idx] + seg[2*idx+1];
}

int query(int idx, int l, int r, int ql, int qr) {
    if (qr < l || ql > r) return 0; // fuori intervallo
    if (ql <= l && r <= qr) return seg[idx]; // interamente dentro
    int mid = (l + r) / 2;
    return query(2*idx, l, mid, ql, qr) + query(2*idx+1, mid+1, r, ql, qr);
}

void update(int idx, int l, int r, int pos, int val) {
    if (l == r) {
        seg[idx] = val;
        return;
    }
    int mid = (l + r) / 2;
    if (pos <= mid) update(2*idx, l, mid, pos, val);
    else update(2*idx+1, mid+1, r, pos, val);
    seg[idx] = seg[2*idx] + seg[2*idx+1];
}
```

## ## Implementazione per sistemi embedded

Negli ambienti embedded non è sempre possibile affidarsi al normale `malloc`/`free`, sia per motivi di frammentazione, sia perché spesso la memoria dinamica è limitata o non affidabile. Una strategia molto usata è il **pool allocator**: si riserva un array statico di nodi e si gestisce a mano l’allocazione (assegnando nodi liberi da una lista interna). Questo elimina sorprese e rende prevedibili i consumi.

Altro punto importante è la **ricorsione**. Molti algoritmi sugli alberi si basano su chiamate ricorsive (inorder, inserimenti, balancing, ecc.), ma sugli MCU lo stack può essere ridotto. Due approcci tipici:

- riscrivere le funzioni in modo **iterativo** usando uno stack esplicito o una coda circolare;
- usare strutture alternative (ad es. alberi threaded) per eliminare del tutto la ricorsione.

## Tabelle di complessità e casi d’uso

Una visione d’insieme delle principali strutture:

|Struttura|Inserimento|Ricerca|Cancellazione|Note d’uso principali|
|---|---|---|---|---|
|BST semplice|O(log n) medio / O(n) peggiore|O(log n) / O(n)|O(log n) / O(n)|Semplice, didattico, inefficiente se non bilanciato|
|AVL|O(log n)|O(log n)|O(log n)|Ottimo per query molto frequenti, altezza minima garantita|
|Red-Black|O(log n)|O(log n)|O(log n)|Usato in librerie e kernel, inserimenti/cancellazioni più veloci di AVL|
|Heap binario|O(log n) inserimento / estrazione|O(1) per massimo|O(log n)|Base di heapsort, ottimo per code di priorità|
|Segment Tree|O(log n) aggiornamento / query|—|—|Query su intervalli e aggiornamenti veloci|
|Expression Tree|dipende dalla costruzione|—|—|Usato per calcoli simbolici e compilatori|
|Threaded Tree|come BST|come BST|come BST|Evita stack/ricorsione, traversal più leggero|

## Errori comuni, debugging e best practice

- **Puntatori NULL mal gestiti**: la causa più tipica di crash negli alberi. Conviene centralizzare la creazione e inizializzazione dei nodi in una funzione unica.
- **Sbilanciamento non trattato**: se usi BST puri senza bilanciamento, gli input già ordinati creano liste, degradando a `O(n)`. Per debugging puoi stampare l’altezza dopo ogni inserimento e controllare se cresce troppo velocemente.
- **Memory leak**: dimenticare di liberare i nodi alla cancellazione. Meglio scrivere funzioni di `destroy_tree()` (postorder) o usare pool allocator che riciclano i nodi.
- **Ricorsione profonda**: rischi di stack overflow in MCU. Riscrivi gli algoritmi con cicli e stack manuali.
- **Debugging traversal**: per capire lo stato di un albero, stampa sempre le visite preorder/inorder/postorder: danno una “firma” utile. Ad esempio, inorder ti mostra se un BST è ancora valido.
- **Rotazioni sbagliate in AVL/RB**: errori classici stanno nella mancata aggiornazione dei puntatori o dei fattori di bilanciamento. Conviene scrivere test unitari con piccoli alberi e disegnare i casi a mano.

Best practice generali:

- Scrivi funzioni piccole e modulari (es. `rotate_left`, `update_height`, `fix_violation`).
- Inserisci assert o logging sui fattori di bilanciamento / colori per verificare le invarianti.
- Se lavori in embedded, considera la versione **iterativa + pool allocator** come default.    

```c
#include <stdio.h>
#include <stdint.h>

#define MAX_NODES 32   // massimo numero di nodi gestibili
#define STACK_SIZE_32

typedef struct Node {
    int key;
    struct Node* left;
    struct Node* right;
} Node;

// Pool statico
static Node node_pool[MAX_NODES];
static int free_list[MAX_NODES];   // stack di indici liberi
static int free_top = -1;

// Inizializza il pool
void init_pool(void) {
    for (int i = 0; i < MAX_NODES; i++) {
        free_list[++free_top] = i;  // tutti liberi all’inizio
    }
}

// Alloca un nodo dal pool
Node* alloc_node(int key) {
    if (free_top < 0) {
        return NULL; // pool esaurito
    }
    int idx = free_list[free_top--];
    Node* n = &node_pool[idx];
    n->key = key;
    n->left = NULL;
    n->right = NULL;
    return n;
}

// Rilascia un nodo (lo rimette nel pool)
void free_node(Node* n) {
    int idx = n - node_pool;   // calcola indice nell’array
    free_list[++free_top] = idx;
}
```

Implementazione senza ricorsione:
```c
Node* insert_bst(Node* root, int key) {
    Node* new_node = alloc_node(key);
    if (!new_node) return root;  // pool pieno

    if (root == NULL) {
        return new_node; // primo nodo
    }

    Node* curr = root;
    Node* parent = NULL;

    while (curr != NULL) {
        parent = curr;
        if (key < curr->key)
            curr = curr->left;
        else
            curr = curr->right;
    }

    if (key < parent->key)
        parent->left = new_node;
    else
        parent->right = new_node;

    return root;
}

void inorder_iter(Node* root) {
    Node* stack[STACK_SIZE];
    int top = -1;
    Node* curr = root;

    while (curr != NULL || top >= 0) {
        while (curr != NULL) {
            stack[++top] = curr;
            curr = curr->left;
        }
        curr = stack[top--];
        printf("%d ", curr->key);
        curr = curr->right;
    }
}

```

