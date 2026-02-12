# Tipi di `stdint.h`

In C la dimensione di tipi come `int`, `long`, `short` può variare a seconda del compilatore e dell’architettura.  
Per i microcontrollori questo è un problema, perché dobbiamo sapere con certezza quanti **bit** occupa una variabile.

Per questo si usa l’header **`<stdint.h>`**, che definisce tipi con dimensione **fissa**.

---

## Tipi interi con dimensione precisa

- `int8_t` → intero con segno a 8 bit (da -128 a 127)  
- `uint8_t` → intero senza segno a 8 bit (da 0 a 255)  
- `int16_t` → intero con segno a 16 bit  
- `uint16_t` → intero senza segno a 16 bit  
- `int32_t` → intero con segno a 32 bit  
- `uint32_t` → intero senza segno a 32 bit  
- `int64_t` → intero con segno a 64 bit  
- `uint64_t` → intero senza segno a 64 bit  

Esempio:
```c
uint16_t valore = 50000; // sicuro che è 16 bit senza segno
````

---

## Tipi minimi e massimi

Oltre ai tipi a dimensione fissa, `stdint.h` fornisce anche tipi che garantiscono almeno una certa dimensione:

- `int_least8_t` → almeno 8 bit con segno
- `uint_least16_t` → almeno 16 bit senza segno
- `int_fast32_t` → il tipo intero “più veloce” con almeno 32 bit

Questi sono meno usati in embedded, ma possono tornare utili se non importa la dimensione esatta, ma solo la minima.

---

## Costanti con dimensione fissa

`stdint.h` definisce anche **macro** per scrivere costanti con la dimensione giusta:

```c
#include <stdint.h>

uint32_t mask = UINT32_C(0xFFFF0000);
```

Così evitiamo warning o errori quando usiamo numeri grandi.

---

## Buona pratica

In un progetto embedded è consigliato:

- usare sempre i tipi di `stdint.h` (`uint8_t`, `int32_t`, …)
- evitare i tipi “classici” (`int`, `long`, …) perché non sempre hanno la stessa dimensione tra PC e microcontrollore
- abbinare `stdbool.h` e `stdint.h` per scrivere codice più leggibile e portabile
## Confronto tra tipi standard e tipi di `stdint.h`

| Tipo classico | Dimensione (dipende da architettura) | Possibili valori | Equivalente `stdint.h` (fisso) |
|---------------|---------------------------------------|------------------|--------------------------------|
| `char`        | 8 bit (di solito)                     | -128 … 127 / 0 … 255 | `int8_t` / `uint8_t` |
| `short`       | almeno 16 bit (spesso 16)             | -32.768 … 32.767 | `int16_t` |
| `unsigned short` | almeno 16 bit (spesso 16)          | 0 … 65.535       | `uint16_t` |
| `int`         | almeno 16 bit (spesso 32 su ARM)      | dipende da piattaforma | `int32_t` (se serve 32 bit certi) |
| `unsigned int`| almeno 16 bit (spesso 32)             | dipende da piattaforma | `uint32_t` |
| `long`        | almeno 32 bit (può essere 32 o 64)    | dipende da compilatore | `int32_t` o `int64_t` |
| `unsigned long` | almeno 32 bit (può essere 32 o 64)  | dipende da compilatore | `uint32_t` o `uint64_t` |

---

### Perché è importante?

Su un **PC a 64 bit**:  
- `int` è spesso 32 bit.  
- `long` è 64 bit (Linux) o 32 bit (Windows).  

Su un **microcontrollore ARM Cortex-M**:  
- `int` è quasi sempre 32 bit.  
- `long` è 32 bit (non 64!).  

Quindi lo stesso codice scritto con `int` e `long` può comportarsi in modo diverso a seconda di dove gira.  
Con `stdint.h` questo problema non esiste: la dimensione è sempre quella dichiarata.
