# Multiplexing e demultiplexing nel livello di trasporto (versione base)

Difficoltà: base | Tempo stimato: 30 minuti | Linguaggio: Python 3 (solo libreria standard)

## Obiettivi didattici

- Capire come il livello di trasporto permette a più applicazioni (o più connessioni) di condividere la stessa macchina e lo stesso indirizzo IP.
- Vedere in pratica cosa distingue una connessione TCP dall'altra: la quaterna (IP sorgente, porta sorgente, IP destinazione, porta destinazione).
- Capire perché un server concorrente ha bisogno di un thread (o di un altro meccanismo) per ogni connessione accettata.

## Concetti teorici essenziali

- **Multiplexing** (lato mittente): il livello di trasporto raccoglie i dati di più socket applicativi, aggiunge l'intestazione con porta sorgente e porta destinazione e li passa al livello di rete.
- **Demultiplexing** (lato destinatario): leggendo le porte nell'intestazione, il livello di trasporto consegna ogni segmento al socket giusto.
- In TCP il demultiplexing usa **quattro valori** (due IP e due porte): il server ascolta su una sola porta (qui 12345), ma ogni `accept()` produce un socket **nuovo**, identificato dalla porta sorgente (effimera) scelta dal client.
- In UDP basterebbero IP e porta di destinazione (vedi `../TCP_1`).

## Prerequisiti

- Python 3.8 o superiore (`python3 --version`).
- Nessuna libreria esterna, nessun privilegio di root.
- La porta TCP 12345 su `localhost` deve essere libera.

## Struttura dei file

| File | Contenuto |
|---|---|
| `service.py` | Un unico script che contiene sia il server sia i client, eseguiti in thread diversi dello stesso processo. |

Funzioni chiave di `service.py`:

- `server()`: crea un socket TCP, fa `bind(('localhost', 12345))`, `listen(5)` e entra in un ciclo infinito di `accept()`. Per ogni connessione avvia un thread che esegue `handle_client`.
- `handle_client(client_socket)`: legge fino a 1024 byte alla volta con `recv`, stampa il messaggio ricevuto, risponde con `"Message received"`; esce dal ciclo quando `recv` restituisce byte vuoti (il client ha chiuso la connessione).
- `client(id)`: apre una connessione verso il server, invia `"Hello from client <id>"`, stampa la risposta e chiude.
- Parte finale del file: avvia il thread del server e poi tre thread-client (`id` = 1, 2, 3), li attende con `join()`.

## Procedura passo-passo

1. Spostati nella cartella del laboratorio:

   ```bash
   cd Livello_Rete/Multiplex_Demultiplex
   ```

2. (Facoltativo) controlla la sintassi:

   ```bash
   python3 -I -m py_compile service.py
   ```

3. Esegui lo script:

   ```bash
   python3 service.py
   ```

4. Output atteso (l'ordine delle righe può cambiare da un'esecuzione all'altra perché i client girano in parallelo; le porte sorgente sono diverse a ogni esecuzione):

   ```text
   Server listening on port 12345
   Connection from ('127.0.0.1', 49734) has been established.
   Received: Hello from client 1 from the client
   Connection from ('127.0.0.1', 49740) has been established.
   Response from server: Message received
   Received: Hello from client 2 from the client
   Response from server: Message received
   Connection from ('127.0.0.1', 49750) has been established.
   Received: Hello from client 3 from the client
   Response from server: Message received
   ```

5. **Lo script non termina da solo**: il thread del server è in un ciclo infinito. Fermalo con `Ctrl+C` (poi, se serve, un secondo `Ctrl+C`).

Questo output è stato verificato eseguendo lo script su loopback (Python 3.10).

## Cosa osservare

- Ogni client compare con una porta sorgente diversa (seconda componente della tupla `('127.0.0.1', porta)`): è questa la chiave che consente al server di distinguere le conversazioni.
- Il server ascolta sempre sulla stessa porta 12345, ma ogni connessione accettata usa un socket dedicato.
- Le righe `Received` e `Response from server` si possono intercalare in modo diverso a ogni esecuzione: nessun ordine è garantito tra thread.

## Esercizi proposti

1. **Più client.** Cambia `range(1, 4)` in `range(1, 11)`. Il server regge dieci connessioni? *Soluzione:* sì, ogni connessione ha il suo thread; `listen(5)` limita solo la coda delle connessioni non ancora accettate.
2. **Vedere le porte.** In un secondo terminale, mentre lo script gira (aggiungi un `time.sleep(30)` in `client` prima di `close()`), esegui `ss -tn | grep 12345`. *Soluzione:* vedi tre connessioni `ESTAB`, con la stessa porta destinazione 12345 e porte sorgente diverse.
3. **Cattura con tcpdump/Wireshark.** Cattura su loopback: `sudo tcpdump -i lo -n 'tcp port 12345'`. Individua i tre handshake a tre vie e le porte sorgente.
4. **Server sequenziale.** Sostituisci `threading.Thread(...).start()` con la chiamata diretta `handle_client(client_socket)`. Cosa cambia? *Soluzione:* i client vengono serviti uno alla volta; un client lento blocca tutti gli altri.
5. **Rimuovere la risposta.** Togli il `send` del server: il client rimane bloccato in `recv`. Perché? *Soluzione:* TCP è uno stream, `recv` attende dati finché la connessione non viene chiusa.

## Troubleshooting

- `OSError: [Errno 98] Address already in use`: la porta 12345 è occupata da un'esecuzione precedente (anche in `TIME_WAIT`) o da un altro lab. Controlla con `ss -ltnp | grep 12345` e termina il processo, oppure attendi circa un minuto.
- `ConnectionRefusedError` nei client: i client partono subito dopo il server; su macchine molto lente il `listen` potrebbe non essere ancora attivo. Rilancia lo script.
- Lo script non torna al prompt: è normale (vedi sopra), usa `Ctrl+C`.
- Le due versioni (`Multiplex_Demultiplex` e `Multiplex_Demultiplex_2`) usano entrambe la porta 12345: non eseguirle contemporaneamente.

## Pulizia

Nessun file viene creato. Se un processo è rimasto in background: `pkill -f service.py` (attenzione: termina tutti i processi con quel nome di script).

## Per continuare

Passa a [`../Multiplex_Demultiplex_2`](../Multiplex_Demultiplex_2/) per vedere un protocollo applicativo con intestazione propria.
