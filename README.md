# AidSpects
Per il progetto sulle biotecnologie

# Smart Glasses AI Agent

Workflow [n8n](https://n8n.io) che simula AidSpects, un paio di occhiali smart per pazienti anziani. Contiene due circuiti indipendenti:

1. **Verifica giornaliera (avviata dalla pianificazione).** Si basa sui dati forniti dal medico: le domande e le risposte attese nella tabella `domande`. AidSpects pone al paziente le domande del medico, dà un indizio quando il paziente sbaglia e, a fine sessione, produce un referto clinico in HTML.
2. **Assistente vocale (avviato dal paziente).** Si basa sui dati forniti dai familiari. Il paziente preme il tasto e fa una domanda agli occhiali. L'assistente risponde usando le note inserite dai familiari, e alla fine viene generato un riepilogo della conversazione in HTML.

In entrambi i circuiti il paziente è simulato da un agente IA, quindi l'intero flusso si può provare senza persone reali.

> **Attenzione:** è un progetto dimostrativo. Non è un dispositivo medico e non va usato per decisioni cliniche.

## Indice

- [Come funziona](#come-funziona)
- [Requisiti](#requisiti)
- [Installazione](#installazione)
- [Configurazione](#configurazione)
- [Utilizzo](#utilizzo)
- [Personalizzazione](#personalizzazione)
- [Risoluzione dei problemi](#risoluzione-dei-problemi)

## Come funziona

### Circuito 1: verifica giornaliera

Si avvia con `Schedule Trigger`, pensato per partire una volta al giorno. I contenuti (domande e risposte attese) sono forniti dal medico.

1. **Avvio e pulizia.** `Pulisci Storico` svuota la tabella `risultati_simulazione`.
2. **Caricamento domande.** Una query legge da `domande` il testo e la risposta attesa dal medico. `Loop Over Items` le elabora una alla volta.
3. **Ciclo di domanda e risposta**, per ogni domanda:
   - `Init Tentativo` imposta il contatore (parte da 1).
   - **Agente Occhiali** pronuncia la domanda. Ai tentativi 2 e 3 spiega che la risposta era incompleta, dà un indizio e ripone la domanda, senza svelare la soluzione.
   - `HTTP Request` trasforma il testo in audio (Google Translate TTS).
   - **Agente Paziente** risponde in prima persona: sbaglia con probabilità del 60%, altrimenti risponde correttamente.
   - **Giudice AI** confronta la risposta con quella attesa e restituisce `corretta` (true/false) e `motivazione`.
   - `Insert rows in a table` salva il tentativo nel database.
4. **Retry.** `If` verifica se la risposta è corretta oppure se si è arrivati al terzo tentativo. Se sì passa alla domanda successiva, altrimenti `Prepara Riprova` incrementa il contatore, salva la motivazione come feedback e ripete il ciclo.
5. **Referto.** Finite le domande, una query legge lo storico. **Report Dottore** scrive il referto (ogni tentativo in ordine cronologico, Corretto/Errato, valutazione sull'aderenza). `HTML` lo inserisce in un template e `Convert to File` genera `Referto_Medico.html`.

```
Schedule Trigger -> Pulisci Storico -> Carica domande -> Loop Over Items
                                                          |
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

### Circuito 2: assistente vocale avviato dal paziente

Si avvia con il nodo `When clicking 'Execute workflow'`, che rappresenta il tasto premuto dal paziente sugli occhiali. Le informazioni usate per rispondere sono quelle inserite dai familiari.

1. **Caricamento domande.** `Leggi Domande Paziente` legge gli argomenti dalla tabella `domande_paziente`. `Loop Over Items 1` li elabora uno alla volta.
2. **Domanda del paziente.** **Agente Paziente 1** simula una persona anziana e trasforma ogni argomento in una domanda spontanea, breve e naturale.
3. **Note dei familiari.** `Leggi note parenti` legge dalla tabella `note_familiari` le informazioni inserite dai parenti (argomento, dettaglio, chi le ha inserite) e le raccoglie in un unico JSON.
4. **Risposta.** **Assistente Occhiali** risponde in italiano in modo chiaro, rassicurante e breve (3-4 frasi), con queste regole:
   - se la domanda riguarda oggetti di casa, medicine, appuntamenti o contatti presenti nelle note, usa esattamente quei dettagli;
   - se la domanda è generica, risponde con le proprie conoscenze;
   - se serve un dettaglio personale che non è nelle note, dice che i familiari non l'hanno ancora salvato e suggerisce di chiedere a loro.

   `Simple Memory` mantiene la memoria della conversazione.
5. **Riepilogo.** `Riepilogo Conversazione` salva domanda e risposta di ogni interazione. Finito il ciclo, `Genera html Conversazione` costruisce la pagina e `Convert to File 1` genera `Riepilogo_Assistente.html`.

```
Manual Trigger -> Leggi Domande Paziente -> Loop Over Items 1
                                                   |
   Agente Paziente 1 -> Leggi note parenti -> Assistente Occhiali (+ Simple Memory)
         ^                                                  |
         |                                                  v
         +--------------- Riepilogo Conversazione <---------+
                                   |
                         (fine ciclo)
                                   v
        Genera html Conversazione -> Convert to File 1 -> Riepilogo_Assistente.html
```

## Requisiti

| Cosa | Dettagli |
|---|---|
| n8n | Versione recente, cloud o self-hosted. Il flusso usa i nodi AI LangChain (Agent, Basic LLM Chain, Structured Output Parser, Simple Memory), il nodo Code e il nodo Postgres. |
| PostgreSQL | Un database raggiungibile da n8n, con utente che può leggere, scrivere e fare `TRUNCATE`. |
| Google Gemini | Una API key di Google AI Studio, per i modelli usati dal flusso. |
| Accesso a internet | Il nodo `HTTP Request` del circuito 1 chiama `translate.google.com`. |

## Installazione

### 1. Crea il database

Esegui questo script su PostgreSQL.

**Tabelle del circuito 1:**

```sql
CREATE TABLE domande (
  id SERIAL PRIMARY KEY,
  testo_domanda TEXT NOT NULL,
  risposta_attesa TEXT NOT NULL
);

CREATE TABLE risultati_simulazione (
  id SERIAL PRIMARY KEY,
  domanda_id INTEGER REFERENCES domande(id),
  risposta_paziente TEXT,
  tentativi_usati INTEGER,
  esito BOOLEAN,
  data_test TIMESTAMP DEFAULT NOW()
);
```

**Tabelle del circuito 2:**

```sql
CREATE TABLE domande_paziente (
  id SERIAL PRIMARY KEY,
  testo_domanda TEXT NOT NULL
);

CREATE TABLE note_familiari (
  id SERIAL PRIMARY KEY,
  argomento TEXT NOT NULL,
  dettaglio TEXT NOT NULL,
  inserito_da TEXT
);
```

**Dati di esempio** (sostituiscili con i tuoi):

```sql
INSERT INTO domande (testo_domanda, risposta_attesa) VALUES
('Quante compresse di paracetamolo puoi assumere al massimo in 24 ore?',
 'Massimo 3 compresse da 1000 mg (3 grammi al giorno), a distanza di almeno 6-8 ore.'),
('Cosa devi fare prima di misurare la pressione arteriosa?',
 'Restare seduto a riposo per almeno 5 minuti senza aver bevuto caffè o fumato.');

INSERT INTO domande_paziente (testo_domanda) VALUES
('Dove ho messo gli occhiali da lettura'),
('Quando devo prendere la medicina per la pressione');

INSERT INTO note_familiari (argomento, dettaglio, inserito_da) VALUES
('Occhiali da lettura', 'Sono nel primo cassetto del mobile del soggiorno.', 'Maria'),
('Medicina per la pressione', 'Una compressa ogni mattina dopo colazione.', 'Giovanni');
```

### 2. Importa il workflow in n8n

1. Scarica il file `.json` del workflow da questo repository.
2. In n8n apri **Workflows → Add workflow → Import from file** e seleziona il file.

### 3. Crea le credenziali

Il file esportato non contiene password né chiavi, quindi vanno ricreate in n8n (**Credentials → Add credential**):

- **Postgres**: host, porta, database, utente, password, SSL se richiesto. Va associata a tutti i nodi Postgres: `Pulisci Storico`, `Execute a SQL query`, `Insert rows in a table`, `Execute a SQL query1`, `Leggi note parenti`, `Leggi Domande Paziente`.
- **Google Gemini (PaLM) API**: la tua API key di [Google AI Studio](https://aistudio.google.com/apikey). Va associata a tutti i nodi `Google Gemini Chat Model` (`Chat Model`, `1`, `2`, `3`, `4` e `5`).

### 4. Controlla i nodi

- Apri ogni nodo con un'icona di avviso e seleziona le credenziali create sopra.
- Nei nodi Postgres che usano `schema` e `table` in modalità lista, riseleziona `public` e `risultati_simulazione`.
- Nel nodo `Schedule Trigger` imposta l'intervallo (ad esempio una volta al giorno): nel file importato la regola è vuota.
- Verifica che i nomi dei modelli Gemini nei nodi Chat Model siano disponibili con la tua API key; se non lo sono, selezionane uno dall'elenco.

## Configurazione

| Parametro | Dove | Valore attuale |
|---|---|---|
| Frequenza della verifica | Nodo `Schedule Trigger` | Da impostare (es. giornaliera) |
| Numero massimo di tentativi | Nodo `If`, seconda condizione | 3 |
| Probabilità di risposta sbagliata | Prompt di `Agente Paziente`, `Math.random() < 0.60` | 60% |
| Lingua della voce | URL di `HTTP Request`, parametro `tl` | `it` |
| Modelli | Nodi `Google Gemini Chat Model` | Gemini Flash |
| Aspetto del referto | Nodo `HTML` | Template con intestazione e corpo |
| Limite righe del referto | Query di `Execute a SQL query1`, `LIMIT` | 30 |
| Memoria della conversazione | Nodo `Simple Memory`, chiave di sessione | `sessione_paziente` |
| Aspetto del riepilogo | Codice del nodo `Genera html Conversazione` | Pagina con una scheda per interazione |

## Utilizzo

### Circuito 1: verifica giornaliera

1. Apri il workflow in n8n.
2. Per una prova manuale, esegui il flusso a partire da `Schedule Trigger`. Per l'uso normale attiva il workflow (**Active**): partirà in base alla pianificazione.
3. Al termine apri l'output del nodo `Convert to File` e scarica `Referto_Medico.html`.
4. Apri il file nel browser per leggere il referto.

> Ogni esecuzione cancella lo storico precedente (`TRUNCATE ... RESTART IDENTITY`). Se vuoi conservare i dati, copiali prima altrove o rimuovi il nodo `Pulisci Storico`.

### Circuito 2: assistente vocale

1. Inserisci le note dei familiari nella tabella `note_familiari` e gli argomenti da chiedere in `domande_paziente`.
2. Premi **Execute workflow** sul nodo `When clicking 'Execute workflow'`.
3. Al termine apri l'output del nodo `Convert to File 1` e scarica `Riepilogo_Assistente.html`.

## Personalizzazione

- **Nuove domande del medico:** aggiungi righe alla tabella `domande`. Se i tentativi totali sono più di 30, alza il `LIMIT` della query del referto.
- **Nuove domande del paziente:** aggiungi righe a `domande_paziente`.
- **Nuove informazioni dei familiari:** aggiungi righe a `note_familiari`. L'assistente le legge a ogni domanda, senza modificare il workflow.
- **Tono degli agenti:** modifica il messaggio di sistema di `Agente Occhiali`, `Agente Paziente`, `Agente Paziente 1` e `Assistente Occhiali`.
- **Criteri di valutazione:** modifica il prompt di `Giudice AI` (nel campo *Text*) e lo schema del `Structured Output Parser`.
- **Formato del referto:** modifica il prompt di `Report Dottore` e il template del nodo `HTML`.
- **Formato del riepilogo:** modifica il codice del nodo `Genera html Conversazione`.
- **Avvio da parte del paziente reale:** per usare il circuito 2 con un dispositivo vero, sostituisci il trigger manuale con un nodo `Webhook` chiamato dal tasto degli occhiali.


| Nel riepilogo compaiono "Domanda non disponibile" e "Risposta non disponibile" | I campi `domanda_paziente` e `risposta_occhiali` del nodo `Riepilogo Conversazione` sono vuoti | Valorizzali, ad esempio `{{ $('Agente Paziente 1').last().json.output }}` e `{{ $json.output }}` |
