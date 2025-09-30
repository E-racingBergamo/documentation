# ERB documentation

## Introduzione

Questa documentazione raccoglie tutte le informazioni necessarie per comprendere e lavorare sul progetto della **centralina di controllo della macchina**.  
È pensata sia per i membri esperti del team sia per i nuovi ragazzi che iniziano a collaborare: fornisce quindi spiegazioni passo per passo, a partire dalle basi del linguaggio C fino ad arrivare ai dettagli di firmware, hardware e schede elettroniche.

---

## Obiettivi della documentazione

- Offrire una panoramica chiara del progetto nella sua interezza.  
- Fornire materiale di formazione per i nuovi membri, senza dare nulla per scontato.  
- Creare un manuale di riferimento tecnico per lo sviluppo e la manutenzione.  
- Centralizzare tutte le informazioni in un unico posto facilmente consultabile.  

---

## Cosa troverai in queste pagine

1. **Fondamenti di programmazione in C**  
   Per chi non ha mai programmato o ha bisogno di ripasso.  

2. **Embedded e microcontrollore**  
   Spiegazione del funzionamento di un microcontrollore e degli strumenti di sviluppo.  

3. **FreeRTOS**  
   Introduzione al sistema operativo real-time usato nel firmware.  

4. **Architettura software**  
   Struttura del progetto: task, FSM, logging, parser CAN.  

5. **Hardware e schede elettroniche**  
   Panoramica della centralina, dei sensori e delle altre PCB.  

6. **Comunicazioni**  
   CAN bus, UART e telemetria.  

7. **Guide pratiche**  
   Esempi passo per passo per aggiungere nuove funzionalità al sistema.  

---

## Destinatari

Questa documentazione è rivolta a:
- Studenti e nuovi membri del team che devono imparare da zero.  
- Chi si occupa di sviluppo software e firmware.  
- Chi lavora sull’hardware o sull’integrazione dei sistemi.  

---

## Come utilizzare la documentazione

Ti consigliamo di partire dalla sezione **Fondamenti di C**, per poi proseguire con le parti embedded e gradualmente arrivare alle implementazioni specifiche della macchina.  
Ogni capitolo è indipendente e può essere letto anche separatamente come riferimento tecnico.
