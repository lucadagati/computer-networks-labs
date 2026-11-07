# Multiplexing e demultiplexing con intestazione applicativa (versione avanzata)

Difficoltà: base-intermedia | Tempo stimato: 40 minuti | Linguaggio: Python 3 (solo libreria standard)

## Obiettivi didattici

- Vedere come un protocollo applicativo può aggiungere una propria intestazione (qui un campo `client_id` in JSON) per distinguere il mittente logico di un messaggio, oltre a quanto già fa il livello di trasporto con le porte.
- Capire il problema dei **confini dei messaggi** in uno stream TCP e la soluzione classica: prefisso con la lunghezza (*length-prefix framing*).
- Confrontare il demultiplexing a livello di trasporto (porte) con quello a livello applicativo (`client_id`).

## Concetti teorici essenziali

- TCP offre un flusso di byte **senza confini**: due `send` consecutivi possono arrivare in un'unica `recv`, e un solo `send` può arrivare spezzato in più `recv`.
- Per delimitare i messaggi si prefissa la lunghezza: qui 4 byte big-endian (`int.to_bytes(4, 'big')`) seguiti dal corpo.
- Il demultiplexing a livello di trasporto distingue le connessioni con la quaterna IP/porte; l'intestazione applicativa permette di distinguere ulteriori "mittenti logici" anche all'interno della stessa connessione.

## Prerequisiti

- Python 3.8+ (solo libreria standard: `socket`, `threading`, `json`).
- Porta TCP 12345 su `localhost` libera.
- Aver svolto `../Multiplex_Demultiplex` (consigliato).

## Struttura dei file

| File | Contenuto |
|---|---|
| `service.py` | Server e client nello stesso script, in thread distinti. |

Funzioni chiave:

- `server()`: socket TCP su `localhost:12345`, `accept()` in ciclo, un thread per connessione.
- `handle_client(sock)`: in ciclo legge **4 byte** (lunghezza), poi `recv(lunghezza)` per il corpo, decodifica il JSON e stampa `Received: <data> from client ID <client_id>`; risponde `"Message received"`. Gli errori sono catturati e la connessione viene sempre chiusa (`finally`).
- `client(id, message)`: costruisce `{"client_id": id, "data": message}`, invia prima la lunghezza (4 byte) e poi il JSON con `sendall`, quindi stampa la risposta.
- Parte finale: lancia il server e tre client (messaggi `Hello from client 1`, `Data from client 2`, `Message from client 3`).

Nota: il commento nel `README` originale parla di "una singola connessione"; nel codice **ogni client apre una propria connessione TCP**. Il `client_id` è quindi ridondante rispetto alla porta sorgente, ma mostra come funzionerebbe un demultiplexing applicativo.

## Procedura passo-passo

1. ```bash
   cd Livello_Rete/Multiplex_Demultiplex_2
   python3 -I -m py_compile service.py     # controllo sintassi (facoltativo)
   python3 service.py
   ```

2. Output atteso (ordine variabile; verificato su loopback):

   ```text
   Server listening on port 12345
   Connection from ('127.0.0.1', 58834) has been established.
   Received: Data from client 2 from client ID 2
   Connection from ('127.0.0.1', 58838) has been established.
   Response from server: Message received
   Received: Hello from client 1 from client ID 1
   Response from server: Message received
   Connection from ('127.0.0.1', 58854) has been established.
   Received: Message from client 3 from client ID 3
   Response from server: Message received
   ```

3. Lo script **non termina** da solo (thread del server infinito): `Ctrl+C`.

## Cosa osservare

- Il server sa chi ha scritto sia dalla porta sorgente (livello di trasporto) sia dal campo `client_id` (livello applicativo).
- Le due `sendall` del client (lunghezza, poi corpo) sono separate: il server deve leggere esattamente 4 byte per primi.

## Esercizi proposti

1. **Cattura i byte.** Con `sudo tcpdump -i lo -n -X 'tcp port 12345'` trova i primi 4 byte del payload: sono `00 00 00 xx`, dove `xx` è la lunghezza del JSON.
2. **Messaggio lungo.** Fai inviare a un client un messaggio di 5000 caratteri e poi di 200000. *Soluzione:* `handle_client` fa una sola `recv(msg_length)`, che per TCP può restituire **meno** byte di quelli richiesti. Su loopback con messaggi piccoli di solito funziona, con messaggi grandi o su rete reale il `json.loads` può fallire (viene stampato `An error occurred: ...`). Il comportamento esatto dipende da buffer e scheduler e non è stato verificato per queste dimensioni. Correzione: un ciclo che legge finché non ha ricevuto tutti i byte (funzione `recv_exact`).
3. **Più messaggi sulla stessa connessione.** Modifica `client` per inviare due messaggi consecutivi sullo stesso socket. Il ciclo `while True` del server li gestisce entrambi? Fai attenzione alle `recv` della risposta: possono unirsi.
4. **Dispatcher.** Nel server crea un dizionario `{1: gestore_a, 2: gestore_b, 3: gestore_c}` e chiama la funzione corrispondente al `client_id`: hai realizzato un demultiplexing applicativo.
5. **Header sbagliato.** Invia un JSON senza il campo `client_id`: il `KeyError` viene catturato dal blocco `except`, stampato e la connessione chiusa.

## Troubleshooting

- `Address already in use`: un'altra esecuzione (o `Multiplex_Demultiplex`) occupa la 12345. `ss -ltnp | grep 12345`.
- Messaggio `An error occurred: ...`: JSON troncato o lunghezza errata, vedi esercizio 2.
- Non torna il prompt: normale, `Ctrl+C`.

## Pulizia

Nessun file creato. Eventuali processi residui: `pkill -f service.py`.
