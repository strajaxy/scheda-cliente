# Scheda cliente: formato di importazione (versione 1)

File: `scheda_<cliente>_<data>.json`, UTF-8. Accompagnato da `riepilogo_<cliente>_<data>.pdf` (stessi dati, per lavorare a mano se il JSON è danneggiato) e, se presente, `logo_<cliente>_<data>.png`.

## Struttura del file (ordine delle chiavi fisso)

```
schema            "wiseman-scheda-cliente"
versione          1
creata_il         data e ora ISO 8601 (UTC)
struttura         vedi sotto
dpo               vedi sotto
soggetti          elenco
personale         elenco
opzioni           4 booleani
checksum          "sha256:<64 esadecimali>"  (ultima chiave; null se il telefono non lo supporta)
```

### struttura (1 oggetto)
ragione_sociale, partita_iva (11 cifre, senza spazi), indirizzo, cap, comune, provincia (maiuscolo), telefono, email, pec (obbligatoria per il data breach), profilo (`odontoiatria` | `poliambulatorio` | `fisioterapia` | `ottica`), data_documenti (AAAA-MM-GG), logo (true/false: se true c'è il file logo_*.png).

### dpo (1 oggetto)
- `{"presente": false}`: nessun DPO, si tolgono tutti i riferimenti.
- Se `presente: true`: tipo (`interno` | `esterno`), natura (`fisica` | `giuridica`), nome_denominazione, codice_fiscale, partita_iva, sede, telefono, email. Per il DPO interno codice_fiscale, partita_iva e sede sono sempre vuoti.

### soggetti (N righe)
| campo | note |
|---|---|
| ruolo | `referente_privacy` · `amministratore_sistema` · `autorizzato` · `manutentore` · `pulizie` · `responsabile` |
| nome_denominazione | obbligatorio |
| indirizzo | facoltativo |
| cf_piva | facoltativo (codice fiscale o partita IVA) |
| categoria_mansione | testo libero |
| credenziali | `fisse` \| `occasionali`; solo per amministratore_sistema e manutentore, altrimenti "" |
| ambiti_intervento | solo per amministratore_sistema e manutentore, altrimenti "" |

### personale (N righe)
Stesse colonne dell'Excel attuale: cognome, nome, ruolo, email, codice_fiscale (facoltativo, di norma vuoto), note.

### opzioni
smartphone_tablet, chiavette_internet, accordo_sindacati, videosorveglianza (true/false).

## Regole per l'import nell'app
1. Rifiutare se `schema` non è quello atteso o `versione` è maggiore di quella supportata.
2. Verificare il checksum: togliere la chiave `checksum`, serializzare in modo compatto (Python: `json.dumps(obj, ensure_ascii=False, separators=(',', ':'))`, chiavi nell'ordine del file), SHA-256 dei byte UTF-8. Se non coincide: avvisare "file danneggiato, usa il PDF" e non importare nulla. Se `checksum` è null: avviso, ma si può proseguire.
3. Stringa vuota = dato non fornito: non sovrascrive dati già presenti in app. Gli aggiornamenti di soggetti esistenti restano proposti, mai automatici.
4. Importare tutto in una sola operazione (o tutto o niente).

## Pubblicazione del modulo
La cartella `site/` va servita in https (richiesto per l'uso offline e per l'installazione). Dal telefono si apre una volta con connessione e si sceglie "Aggiungi a Home" (su iPhone da Safari). Nel file index.html i colori sono nel blocco `:root` all'inizio del CSS.


## Versione 2 (gestionale v1.40)
Chiavi esportate: struttura{titolare,piva,indirizzo,cap,comune,provincia,telefono,email,pec,profilo,data(gg/mm/aaaa),logo};
dpo{attivo,origine(esterno|interno),tipo(fisica|giuridica),nome,cf,piva,sede,telefono,email} oppure {attivo:false};
soggetti[{ruolo(responsabile|ads|manutentore|referente|pulizie|amministratore|socio),tipo,nome,cf,piva,indirizzo,mansione,credenziali+ambiti(solo ads/manutentore),categorie(solo responsabile)}];
personale[{nome("Cognome Nome"),ruolo,mansioni[],email,cf,note}];
opzioni{telefonia,chiavette,accordo_sind};
trattamenti{<nome>:{attivo,<opzione>:bool}} solo per i trattamenti toccati (videosorveglianza ora qui).
Profili: odontoiatria, poliambulatorio, fisioterapia, medicina_estetica, ottica, palestra.
Checksum invariato: SHA-256 del JSON compatto senza il campo checksum.

## Versione 3 (gestionale v1.42)
- Profili aggiunti: medico_specialista, medicina_lavoro.
- trattamenti.pazienti: nuove opzioni booleane lavoro, sedi_clienti, medici_collaboratori, cartelle_societa. Con profilo medicina_lavoro: lavoro=true; marketing, foto_ulteriori, minorenni, dermatologia=false.
- Nuova mansione medico_competente. I soggetti "responsabile" con categoria professionista_sanitario possono avere `mansioni` (elenco).
- `sedi` (facoltativa, omessa se sede unica): [{nome, indirizzo, cap, comune, provincia, telefono, email, profilo?, trattamenti? (solo differenze rispetto alla principale), personale?}]. Identità della sede = nome.

## Apri scheda esistente (app telefono)
Legge schema `wiseman-scheda-cliente` o `scheda-cliente`; `versione` non numerica = file del gestionale; checksum assente = nessun avviso, presente e diverso = avviso "modificato"; campi mancanti vuoti, sconosciuti ignorati; sostituisce i dati con conferma. `creata_il` viene riscritta a ogni esportazione.

## Condivisione su Android
Chrome su Android rifiuta i file JSON nel menu Condividi ("Permission denied"): su Android il JSON viene inviato come `scheda_….json.txt` (text/plain), contenuto identico. "Apri scheda esistente" legge anche questi file. I pulsanti Salva JSON / Salva PDF salvano un file per tocco (un download alla volta).

## PDF che contiene il JSON (GPJSON1) e nuovi campi

- Il file `.json` è ora compatto (una riga): è lo stesso testo, carattere per carattere, incorporato nel PDF.
- Nel PDF, campo Keywords: `GPJSON1:<lunghezza base64>:<base64 del JSON in UTF-8>` (senza a capo né spazi, non cifrato).
- Piè di pagina dell'ultima pagina: «Contiene i dati completi per il gestionale (GPJSON1)».
- Soggetti: nuove chiavi `telefono`, `email`, `mansioni` (codici, per tutti i ruoli); chiavi sempre presenti (vuote = "" o []).
- DPO interno: `cognome`, `nome`, `cf` (per il DPO esterno `nome` resta la denominazione).
- Selettore in "Fine visita": Solo JSON / Solo PDF / Tutti e due (ricordato); cache `scheda-v7`.
