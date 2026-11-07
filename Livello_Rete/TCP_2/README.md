# Server TCP concorrente con i thread (echo)

Difficoltà: base-intermedia | Tempo stimato: 30 minuti | Linguaggio: Python 3 (libreria standard)

## Obiettivi didattici

- Passare da un server che serve un solo client (`../TCP_1`) a un server **concorrente** che gestisce più client contemporaneamente.
- Capire la differenza tra socket di ascolto e socket di connessione (uno per client).
- Riconoscere un protocollo di tipo *echo*.

## Concetti teorici essenziali

- `accept()` è bloccante: finché il server non torna ad `accept`, nessun nuovo client può essere servito. Delegando ogni connessione a un thread, il thread principale resta libero di accettare.
- `recv()` restituisce byte vuoti (`b''`) quando l'altro lato ha chiuso la connessione: è il segnale di fine stream.
- I thread condividono la memoria; qui ogni thread usa solo il proprio socket, quindi non servono lock.

## Prerequisiti

- Python 3.8+, due o più terminali, porta TCP 12345 libera su `localhost`.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `serverTCP.py` | `handle_client(client_socket)`: in ciclo `recv(1024)`, stampa `Dati ricevuti: ...` e rimanda gli stessi byte con `sendall` (echo); a fine stream chiude il socket. Il corpo principale fa `bind/listen` su `localhost:12345` e in un ciclo infinito `accept()` + `threading.Thread(target=handle_client, ...)`. |
| `clientTCP.py` | Si connette, invia `Ciao, sono il client TCP!` e chiude subito. **Non legge** l'eco. |

## Procedura passo-passo

1. Terminale A:

   ```bash
   cd Livello_Rete/TCP_2
   python3 serverTCP.py
   ```

2. Terminali B e C (anche più volte):

   ```bash
   python3 clientTCP.py
   ```

3. Output atteso sul server (verificato con due client in sequenza; le porte variano):

   ```text
   Server TCP in attesa di connessioni...
   Connessione da ('127.0.0.1', 40168)
   Dati ricevuti: Ciao, sono il client TCP!
   Connessione da ('127.0.0.1', 40180)
   Dati ricevuti: Ciao, sono il client TCP!
   ```

   I client non stampano nulla. Il server non termina: `Ctrl+C`.

   Nota: se redirigi l'output su file (`python3 serverTCP.py > log`), Python bufferizza e vedrai le righe solo alla chiusura; usa `python3 -u`.

## Cosa osservare

- Ogni client ha una porta sorgente diversa.
- Il client chiude subito: il server riceve il dato e poi `b''`, quindi il thread termina.
- Il thread principale resta in `accept()` e può servire altri client anche se un thread è occupato.

## Esercizi proposti

1. **Client interattivo.** Usa `nc localhost 12345` (o `telnet localhost 12345`) in due terminali e scrivi righe di testo: ricevi l'eco e il server stampa ogni riga. Due client sono serviti in parallelo.
2. **Dimostra il blocco.** Sostituisci il thread con la chiamata diretta `handle_client(client_socket)` e riprova con due `nc`: il secondo non riceve eco finché il primo non chiude.
3. **Leggi l'eco.** Modifica `clientTCP.py` aggiungendo `print(client_socket.recv(1024))` prima della `close()`. Dovresti vedere `b'Ciao, sono il client TCP!'`.
4. **Thread attivi.** Aggiungi in `handle_client` una stampa di `threading.active_count()`. Apri 5 `nc` e controlla come cresce e cala.
5. **Terminazione pulita.** I thread non sono `daemon`: aggiungi `daemon=True` e gestisci `KeyboardInterrupt` per chiudere il server con ordine.

## Troubleshooting

- `Address already in use`: porta occupata (`ss -ltnp | grep 12345`) o `TIME_WAIT`.
- `ConnectionRefusedError`: server non avviato.
- Nessun output sul server con redirezione su file: buffering, usa `-u`.

## Pulizia

Nessun file creato. Ferma il server con `Ctrl+C`.
