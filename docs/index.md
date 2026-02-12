# Benvenuto nella Documentazione Tecnica ERB

Questa è la knowledge base ufficiale del reparto **Elettronica & Software** di E-Racing Bergamo.
Qui non troverai solo codice, ma il know-how accumulato che permette alla nostra vettura di muoversi, comunicare e performare in pista.

La documentazione è divisa in due macro-aree:

1. **Academy**: Il percorso formativo per portare i nuovi membri da zero a sviluppatori embedded.
2. **Engineering**: Le specifiche tecniche, l'architettura firmware e i datasheet dell'hardware installato in vettura.

---

## La nostra Filosofia

Il software in un veicolo da corsa è mission-critical. Un errore non causa solo un crash del programma, ma può fermare la macchina in gara o creare situazioni di pericolo. Per questo motivo, questa documentazione serve a garantire:

* **Affidabilità**: Ogni riga di codice deve essere comprensibile e manutenibile.
* **Knowledge Transfer**: Il sapere non deve "laurearsi" con i membri senior, ma rimanere nel team.
* **Standardizzazione**: Scriviamo codice seguendo regole comuni, non stili personali.

---

## Da dove inizio?

Non tutti devono leggere tutto. Scegli il percorso più adatto al tuo ruolo:

!!! tip "Sono una nuova recluta 👶"
    Benvenuto a bordo! Il tuo obiettivo ora è configurare il PC e imparare le basi.

    1.  Vai su **Onboarding > Setup Ambiente** per installare i tool.
    2.  Studia la sezione **Academy**: parti dai *Fondamenti di C* se serve ripasso, poi passa a *Embedded & RTOS*.
    3.  Non toccare il codice della macchina finché non hai completato i primi esercizi!

!!! abstract "Sono uno Sviluppatore Firmware 💻"
    Sai già come programmare. Ti serve capire come funziona questa macchina.

    1.  Leggi **Firmware Architecture** per capire come girano i task e la FSM.
    2.  Consulta **Moduli di Controllo** per la logica specifica (es. Traction Control).
    3.  Usa le **Guide Operative** per imparare a flashare e debuggare la ECU.

!!! success "Mi occupo di Hardware/Cablaggi 🔌"
    Ti serve sapere dove collegare i fili.

    1.  Vai diretto alla sezione **Hardware & Elettronica**.
    2.  Consulta i pinout della **Centralina** e le specifiche dei **Sensori**.

---

## Stack Tecnologico

Una panoramica rapida degli strumenti che utilizziamo:

| Ambito | Tecnologia |
| :--- | :--- |
| **Linguaggio** | C (Standard C99/C11) |
| **OS** | FreeRTOS, Mbed |
| **Hardware** | STM32 (ARM Cortex-M), MIMXRT1064 |
| **Protocolli** | CAN Bus, UART, SPI |
| **Version Control** | Git & GitHub |
| **Ambiente di sviluppo** | MCUXpressoIDE, STM32CubeIDE |

> *"Talk is cheap. Show me the code."* — Linus Torvalds
