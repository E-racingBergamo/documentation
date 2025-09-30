# Centralina

Questa sezione documenta il software sviluppato per la centralina.

---

## Architettura generale

Il codice è organizzato in più moduli, ciascuno con responsabilità precise:

- **CAN controller**: codifica e decodifica di messaggi sulla linea CAN.
- **ADC controller**: gestisce i sensori analogici.
- **Digital I/O**: per ingressi e uscite digitali.
- **Hall sensors**: legge i sensori a effetto Hall.
- **Inverter**: comunicazione con inverter e motori
- **Telemetry**: formatta i messaggi di telemetria e li mette in coda per essere spediti su una periferica a scelta.
- **Ticker**: sincronizza le task periodiche come la lettura dei sensori o l'esecuzione della macchina a stati dell'inverter.
- **Tractive system manager**: gestisce la macchina a stati complessiva della macchina e fornisce le autorizzazioni per accendere gli inverter.

---

## FreeRTOS

FreeRTOS è un sistema operativo real-time open source progettato per microcontrollori.  
Offre meccanismi per:

- Creare e gestire **task** che vengono eseguiti in modo concorrente.
- Sincronizzare l’esecuzione tramite **code**, **semafori** e **mutex**.
- Pianificare attività periodiche con **timer software**.

Il suo utilizzo permette di strutturare il firmware in moduli indipendenti e ben organizzati, migliorando la scalabilità e la manutenibilità del progetto.

---

## Macchine a stati

Le FSM sono implementate in **C** utilizzando la struttura `switch-case`.
Ogni stato rappresenta una fase di funzionamento, ad esempio:

```c
switch (state) {
    case INIT:
        // Inizializzazione
        break;
    case RUN:
        // Esecuzione principale
        break;
    case ERROR:
        // Gestione errori
        break;
}
```

---

## Logging

Il logging è gestito tramite **UART** con code di output.
Caratteristiche principali:

* Supporto a più livelli di log (INFO, WARN, ERROR).
* Invio asincrono dei messaggi.

Esempio di utilizzo:

```c
LOG_INFO("Sistema inizializzato correttamente");
LOG_ERROR("Errore durante la trasmissione CAN");
```

---

## Comunicazioni

### CAN

* Supporto a due linee can.
* Callback per la gestione asincrona dei messaggi.

### UART

* Utilizzata per logging e debug.
* Interfaccia DMA per migliorare le prestazioni.

---

## Moduli futuri

* **Telemetria**: invio dati in tempo reale a un server esterno.
* **Strategie di controllo avanzate**: da integrare in base agli sviluppi del progetto.

