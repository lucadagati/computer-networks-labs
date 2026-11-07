# Chiusura della connessione TCP (four-way handshake) simulata

Difficoltà: base | Tempo stimato: 30 minuti | Linguaggio: Python 3 (libreria standard)

## Obiettivi didattici

- Ripassare la chiusura "graceful" di TCP: FIN, ACK, FIN, ACK.
- Capire la differenza tra **messaggi applicativi** che si chiamano "FIN"/"ACK" e i **flag reali** dell'intestazione TCP.
- Imparare a confrontare un log applicativo con una cattura di pacchetti.

## Concetti teorici essenziali

- Ogni direzione di una connessione TCP si chiude indipendentemente: chi ha finito di inviare manda un segmento con flag FIN, l'altro lo conferma con ACK (*half-close*).
- Sequenza classica: `FIN` (A→B), `ACK` (B→A), `FIN` (B→A), `ACK` (A→B). Spesso l'ACK e il secondo FIN viaggiano nello stesso segmento (FIN+ACK), da cui i tre pacchetti visti in pratica.
- Chi chiude per primo entra in `TIME_WAIT` (circa 2·MSL).

## Prerequisiti

- Python 3.8+, due terminali, porta TCP 12345 su `localhost` libera.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `server.py` | Accetta **una** connessione, poi chiama `terminate_connection(conn)`: stampa `Ricevuto FIN dal client`, invia i byte `ACK`, poi i byte `FIN`, attende `recv(1024)` e controlla se è `ACK`, quindi chiude. Ogni passo è separato da `time.sleep(0.1)` e stampato con timestamp `HH:MM:SS.mmm`. |
| `client.py` | Si connette, stampa l'inizio della terminazione, invia i byte `FIN`, aspetta `ACK`, aspetta `FIN`, risponde `ACK` e chiude. |

Attenzione: `FIN` e `ACK` qui sono **stringhe inviate dentro il payload**, non flag TCP. La vera chiusura TCP avviene solo con `close()`, che fa generare al kernel i veri segmenti FIN/ACK.

## Procedura passo-passo

```bash
cd Livello_Rete/TCP_4
python3 server.py        # terminale A
python3 client.py        # terminale B, entro pochi secondi
```

Output atteso sul client (verificato su loopback; i timestamp cambiano):

```text
10:06:52.564 - Client: Inizia il processo di terminazione.
10:06:52.665 - Client: Invio FIN al server.
10:06:52.765 - Client: Ricevuto ACK dal server.
10:06:52.865 - Client: Ricevuto FIN dal server. Invio ACK finale.
10:06:52.865 - Client: Connessione chiusa.
```

Output atteso sul server:

```text
10:06:52.564 - Server: Connessione stabilita con ('127.0.0.1', 43594)
10:06:52.565 - Server: Ricevuto FIN dal client.
10:06:52.665 - Server: Invio ACK al client.
10:06:52.765 - Server: Invio FIN al client.
10:06:52.865 - Server: Aspetto l'ACK finale dal client.
```

### Comportamento "anomalo" da notare

Il server **non stampa** mai `Ricevuto ACK finale dal client` (verificato in tre esecuzioni). Motivo, leggendo il codice: il server stampa "Ricevuto FIN dal client" senza aver mai fatto una `recv`, quindi i byte `FIN` inviati dal client restano nel buffer; l'unica `recv` del server legge `FIN` (non `ACK`) e il confronto `== b'ACK'` fallisce, quindi il server chiude in silenzio. È un ottimo esercizio di debug (esercizio 2).

## Cosa osservare

- Le pause da 100 ms rendono visibile la sequenza temporale.
- Confronta il log con una cattura reale (esercizio 1): i segmenti TCP veri FIN/ACK compaiono **dopo** lo scambio dei messaggi applicativi, al momento delle `close()`.

## Esercizi proposti

1. **Cattura reale.** `sudo tcpdump -i lo -n 'tcp port 12345'` (o Wireshark su `lo` con filtro `tcp.port == 12345`) e lancia server e client. Individua handshake (SYN, SYN-ACK, ACK), i due segmenti con `PSH` che contengono "FIN" e "ACK" come testo, e infine i veri FIN. In `tshark`: `-Y "tcp.flags.fin==1"`.
2. **Correggi il server.** Aggiungi `conn.recv(1024)` all'inizio di `terminate_connection` (per consumare il `FIN` applicativo). Ora il server stampa `Ricevuto ACK finale`? Attenzione: se `FIN` e `ACK` arrivano nello stesso `recv` possono comparire concatenati (`b'FINACK'`).
3. **Half-close vero.** Sostituisci i messaggi applicativi con `client_socket.shutdown(socket.SHUT_WR)`: il server vedrà `recv` restituire `b''`.
4. **TIME_WAIT.** Subito dopo l'esecuzione lancia `ss -tan | grep 12345` e cerca lo stato `TIME-WAIT`. *Soluzione:* compare sul lato che ha chiuso per primo; qui il server (il client chiude per ultimo, ma `close()` del server avviene dopo `recv`; l'esito esatto va osservato).
5. **Connessione resettata.** Cosa succede se il client chiude con dati non letti nel buffer? Provalo e cerca il flag RST.

## Troubleshooting

- `Address already in use`: connessione precedente in `TIME_WAIT` o server ancora attivo; attendi circa 60 s o usa `SO_REUSEADDR`.
- `ConnectionRefusedError`: il server non è avviato (accetta una sola connessione e poi termina: riavvialo prima di ogni prova).
- Client bloccato: il server non ha inviato `ACK`/`FIN`; riavvia entrambi.

## Pulizia

Nessun file creato. Termina i processi con `Ctrl+C`.
