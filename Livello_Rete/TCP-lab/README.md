# TCP-lab: finestra "a mano" con ACK applicativi

Difficoltà: intermedia | Tempo stimato: 30 minuti | Linguaggio: Python 3 (libreria standard)

## Obiettivi didattici

- Sperimentare l'idea di *finestra* che cresce quando arrivano gli ACK e si riduce in caso di timeout, imitando (in modo molto semplificato) il controllo della congestione.
- Capire la differenza tra **ACK del protocollo TCP** (gestiti dal kernel) e **ACK applicativi** inviati dal programma.
- Scoprire, osservando un comportamento "strano", che TCP è uno stream di byte senza confini tra messaggi.

## Concetti teorici essenziali

- Nel vero TCP la finestra di congestione (`cwnd`) cresce in slow start e congestion avoidance e viene ridotta dopo una perdita (vedi `../TCP_3`).
- Qui non c'è nessuna finestra TCP reale: il "window size" è semplicemente la **dimensione in byte** del blocco che il client invia con una `sendall`. Il kernel decide comunque in modo indipendente segmentazione e finestra.
- `recv(n)` restituisce **al più** `n` byte: un messaggio più lungo di `n` richiede più letture.

## Prerequisiti

- Python 3.8+, due terminali, porta TCP 9999 libera su `127.0.0.1`.

## Struttura dei file

| File | Contenuto |
|---|---|
| `tcp-server.py` | `start_server()`: ascolta su `0.0.0.0:9999`, accetta **una sola** connessione, e per ogni `recv(1024)` stampa `Ricevuto pacchetto di N bytes.` e risponde con i 3 byte `ACK`. Termina quando il client chiude. |
| `tcp-client.py` | `start_client()`: parte con `window_size = 1024`, a ogni ciclo invia `window_size` byte (`b'x' * window_size`) e attende `recv(1024)` con timeout di 2 s. Se riceve esattamente `b'ACK'` aumenta di 512, altrimenti (o a timeout) diminuisce di 512, con minimo 1024. Pausa di 1 s a ogni ciclo. Il ciclo termina solo se `window_size > 8192`. |

## Procedura passo-passo

1. Terminale A:

   ```bash
   cd Livello_Rete/TCP-lab
   python3 tcp-server.py
   ```

   Output: `Server TCP in ascolto sulla porta 9999...`

2. Terminale B:

   ```bash
   cd Livello_Rete/TCP-lab
   python3 tcp-client.py
   ```

3. Output atteso sul client (verificato su loopback; l'inizio è deterministico):

   ```text
   Connesso al server TCP.
   Inviato pacchetto di 1024 bytes.
   ACK ricevuto. Aumento della finestra.
   Inviato pacchetto di 1536 bytes.
   ACK ricevuto. Aumento della finestra.
   Inviato pacchetto di 2048 bytes.
   ACK ricevuto. Aumento della finestra.
   Inviato pacchetto di 2560 bytes.
   ACK non ricevuto. Riduzione della finestra.
   Inviato pacchetto di 2048 bytes.
   ACK non ricevuto. Riduzione della finestra.
   ...
   ```

   Sul server compaiono righe `Ricevuto pacchetto di 1024 bytes.` e, per i blocchi più grandi di 1024, anche una riga `Ricevuto pacchetto di 512 bytes.` (e simili).

4. **Il client non termina da solo** nelle prove effettuate: la finestra oscilla tra 1024 e circa 2560 e non raggiunge mai 8192. Interrompi con `Ctrl+C` nel terminale B; il server, ricevendo la chiusura, termina (se è stato il client a essere interrotto con `Ctrl+C` il server può stampare un `ConnectionResetError` oppure uscire pulito, a seconda della temporizzazione).

## Perché la finestra non cresce come ci si aspetta

Il server legge al massimo 1024 byte per volta e risponde con un `ACK` a **ogni** `recv`. Quando il client invia 1536 byte, il server li legge in due pezzi (1024 + 512) e invia **due** `ACK`. Il client ne legge uno solo e il secondo resta nel buffer; alla `recv` successiva può leggere `ACKACK` (6 byte) che non è uguale a `b'ACK'`: scatta il ramo "ACK non ricevuto" anche se in rete non è stato perso nulla. L'effetto è un'oscillazione artificiale della finestra. Questa spiegazione deriva dalla lettura del codice e dal comportamento osservato; è un ottimo spunto per l'esercizio 2.

## Cosa osservare

- Le righe del server mostrano come i blocchi grandi vengano spezzati in letture da 1024 byte.
- Il numero di ACK ricevuti dal client non coincide con il numero di blocchi inviati.

## Esercizi proposti

1. **Cattura i pacchetti.** `sudo tcpdump -i lo -n 'tcp port 9999'` durante l'esecuzione. Trova i segmenti che trasportano i 3 byte `ACK` e quelli da 1024/512 byte di dati (su loopback l'MSS è molto grande, quindi il kernel non spezza i blocchi, è `recv(1024)` a farlo).
2. **Correggi il client.** Fai leggere al client tutti gli ACK in sospeso (per esempio `ack = client_socket.recv(1024)` e controllo `ack.startswith(b'ACK')`, oppure `ack.count(b'ACK')`). Ora la finestra cresce fino a 8192 e il programma termina? *Soluzione (verificata sostituendo `ack == b'ACK'` con `ack.startswith(b'ACK')`):* sì, il client arriva a 8192, invia l'ultimo blocco e stampa `Connessione chiusa.` (circa 15 cicli, quindi circa 15 s con la pausa originale da 1 s). Il server può comunque terminare con `ConnectionResetError`: il client chiude con ACK ancora non letti nel buffer e il kernel invia un RST invece del FIN. Prova a svuotare il buffer prima della `close()`.
3. **Simula una perdita.** Nel server, salta l'invio dell'ACK una volta su cinque (`if random.random() > 0.2`). Osserva il ramo `socket.timeout`.
4. **Confronto con AIMD.** Modifica client e server in modo che la finestra cresca di 512 per ACK ma si dimezzi in caso di timeout: stai implementando una versione semplificata di AIMD.
5. **Server più robusto.** Il server accetta una sola connessione: avvia due client contemporaneamente. Cosa succede al secondo? *Soluzione:* resta in coda nel backlog (`listen(1)`) senza ricevere servizio finché il primo non chiude.

## Troubleshooting

- `Address already in use` sulla 9999: un server precedente è ancora attivo o in `TIME_WAIT`. `ss -ltnp | grep 9999`.
- `ConnectionRefusedError` nel client: avvia prima il server.
- `ConnectionResetError` sul server: il client è stato interrotto in modo brusco, è normale.

## Pulizia

Nessun file creato. Termina i processi residui con `Ctrl+C` (o `pkill -f tcp-server.py`).
