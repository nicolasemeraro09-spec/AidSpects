# AidSpects
Per il progetto sulle biotecnologie
# Smart Glasses AI Agent

Workflow n8n che simula un paio di occhiali smart per pazienti. Gli occhiali pongono al paziente le domande del medico, lo aiutano con degli indizi quando sbaglia e, a fine sessione, producono un referto clinico per il medico curante.

Il paziente è simulato da un secondo agente IA, così l'intero flusso può essere testato senza persone reali.

## Come funziona

1. **Avvio e pulizia.** Uno `Schedule Trigger` avvia la sessione. Il nodo `Pulisci Storico` svuota la tabella `risultati_simulazione` (`TRUNCATE ... RESTART IDENTITY`).
2. **Caricamento domande.** Una query SQL legge da `domande` l'id, il testo della domanda e la risposta attesa dal medico. `Loop Over Items` le elabora una alla volta.
3. **Ciclo di domanda e risposta**, per ogni domanda:
   - `Init Tentativo` imposta il contatore dei tentativi (parte da 1) e il feedback precedente.
   - **Agente Occhiali** (Gemini) pronuncia la domanda. Al tentativo 2 o 3 spiega che la risposta precedente era incompleta, dà un piccolo indizio basato sull'errore e ripone la domanda, senza mai svelare la risposta.
   - `HTTP Request` converte il testo in audio con Google Translate TTS.
   - **Agente Paziente** (Gemini) risponde in prima persona. Con probabilità del 60% dà una risposta sbagliata o incompleta, altrimenti quella corretta.
   - **Giudice AI** confronta la risposta con quella attesa dal medico e restituisce un output strutturato: `corretta` (true/false) e `motivazione`.
   - `Insert rows in a table` salva ogni tentativo in `risultati_simulazione`.
4. **Logica di retry.** Il nodo `If` controlla se la risposta è corretta oppure se si è raggiunto il terzo tentativo.
   - Sì: si passa alla domanda successiva.
   - No: `Prepara Riprova` incrementa il contatore, salva la motivazione del giudice come feedback e il ciclo riparte con un indizio.
5. **Referto per il medico.** Finite le domande, una query legge lo storico dei tentativi. **Report Dottore** (Gemini) redige il referto: per ogni domanda elenca in ordine cronologico tutte le risposte, indicando se sono Errate o Corrette, e aggiunge una breve valutazione sull'aderenza del paziente. Il testo viene inserito in un template `HTML` ed esportato come `Referto_Medico.html`.

## Struttura del flusso

```
Schedule Trigger -> Pulisci Storico -> Carica domande -> Loop Over Items
                                                          |
                         +--------------------------------+
                         v
   Init Tentativo -> Agente Occhiali -> TTS -> Agente Paziente -> Giudice AI
         ^                                                           |
         |                                                           v
   Prepara Riprova <-- (no) -- If (corretta o 3 tentativi?) <-- Salva tentativo
                                   | (sì)
                                   v
                    prossima domanda / fine ciclo
                                   |
                                   v
        Query storico -> Report Dottore -> HTML -> Referto_Medico.html
```

## Requisiti

- n8n (importa il file `My workflow.json` da *Workflows > Import from file*).
- Un database PostgreSQL con due tabelle:
  - `domande`: `id`, `testo_domanda`, `risposta_attesa`
  - `risultati_simulazione`: `id`, `domanda_id`, `risposta_paziente`, `tentativi_usati`, `esito`, `data_test`
- Credenziali configurate in n8n per PostgreSQL e per l'API di Google Gemini.

## Personalizzazione

- **Numero massimo di tentativi:** condizione nel nodo `If` (ora 3).
- **Difficoltà del paziente simulato:** soglia `Math.random() < 0.60` nel prompt di `Agente Paziente`.
- **Modelli:** ogni agente usa un nodo Google Gemini Chat Model separato, quindi si possono cambiare in modo indipendente.
- **Aspetto del referto:** template nel nodo `HTML`.
- **Lingua della voce:** parametro `tl=it` nell'URL del nodo `HTTP Request`.

## Possibili usi

Aderenza alle terapie, educazione del paziente e monitoraggio a distanza.
