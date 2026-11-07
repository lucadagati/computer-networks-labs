# TCP e UDP con i socket Python: il primo contatto

Difficoltà: base | Tempo stimato: 30 minuti | Linguaggio: Python 3 (libreria standard)

## Obiettivi didattici

- Scrivere e capire un server e un client TCP e UDP minimi con il modulo `socket`.
- Distinguere la sequenza di chiamate dei due protocolli: `bind/listen/accept/recv/send` (TCP) contro `bind/recvfrom/sendto` (UDP).
- Capire cosa significa "orientato alla connessione" rispetto a "senza connessione".

## Concetti teorici essenziali

- **TCP** (`SOCK_STREAM`): prima si stabilisce la connessione (handshake a tre vie), poi si scambia uno stream di byte affidabile e ordinato; il server ha un socket di ascolto e, per ogni client, un socket di connessione restituito da `accept()`.
- **UDP** (`SOCK_DGRAM`): nessuna connessione, ogni `sendto` è un datagramma indipendente, senza garanzie di consegna né ordine; il server usa un solo socket e con `recvfrom` scopre anche l'indirizzo del mittente.
- Porte usate: TCP 12345, UDP 12346 (due protocolli diversi possono usare lo stesso numero di porta, qui sono comunque distinte).

## Prerequisiti

- Python 3.8+, due terminali, porte 12345/TCP e 12346/UDP libere su `localhost`.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `serverTCP.py` | Crea un socket TCP, `bind(('localhost', 12345))`, `listen()`, stampa `Server TCP in attesa di connessioni...`, `accept()` **una** connessione, fa **una** `recv(1024)`, stampa i dati e chiude. |
| `clientTCP.py` | Crea il socket, `connect(('localhost', 12345))`, `sendall(b'Ciao, sono il client TCP!')`, chiude. |
| `serverUDP.py` | Socket `SOCK_DGRAM` su `localhost:12346`, una sola `recvfrom(1024)`, stampa mittente e dati, chiude. |
| `clientUDP.py` | `sendto(b'Ciao, sono il client UDP!', ('localhost', 12346))` e chiude. Non c'è `connect`. |

## Procedura passo-passo

### Parte 1: TCP

```bash
cd Livello_Rete/TCP_1
python3 serverTCP.py          # terminale A (resta in attesa)
python3 clientTCP.py          # terminale B
```

Output atteso sul server (verificato su loopback; la porta del client cambia):

```text
Server TCP in attesa di connessioni...
Connessione da ('127.0.0.1', 43592)
Dati ricevuti: Ciao, sono il client TCP!
```

Il client non stampa nulla. Il server termina da solo.

### Parte 2: UDP

```bash
python3 serverUDP.py          # terminale A
python3 clientUDP.py          # terminale B
```

Output atteso sul server:

```text
Server UDP in attesa di dati...
Dati ricevuti da ('127.0.0.1', 40312): Ciao, sono il client UDP!
```

## Cosa osservare

- **Ordine di avvio.** Con TCP, se avvii il client prima del server ottieni `ConnectionRefusedError`. Con UDP il client termina senza errori anche se il server non c'è: il datagramma va perso e nessuno te lo dice.
- Il server TCP riceve l'indirizzo del client da `accept()`; il server UDP da `recvfrom()`.
- Entrambi i server gestiscono un solo messaggio e poi terminano: per un servizio vero servirebbe un ciclo (vedi `../TCP_2`).

## Esercizi proposti

1. **Client senza server.** Lancia `clientTCP.py` senza server, poi `clientUDP.py` senza server. *Soluzione:* TCP genera `ConnectionRefusedError` (il SYN riceve un RST), UDP non segnala nulla.
2. **Cattura.** `sudo tcpdump -i lo -n 'port 12345 or port 12346'` durante le due prove (oppure Wireshark su `lo`). *Soluzione:* TCP mostra SYN, SYN-ACK, ACK, i dati con PSH, poi FIN/ACK; UDP mostra un solo datagramma. Se il server UDP è assente, tcpdump può mostrare anche un messaggio ICMP *port unreachable* di ritorno (non verificato in questo ambiente).
3. **Più messaggi.** Metti `accept`/`recv` e `recvfrom` in un ciclo `while True` e invia più messaggi da due client in parallelo. Cosa osservi? *Soluzione:* il server TCP serve i client uno alla volta; il server UDP riceve da tutti perché non ha connessioni.
4. **Eco.** Fai rispondere il server con gli stessi dati (`sendall`/`sendto`) e il client leggere la risposta. Per UDP il client deve usare `recvfrom`.
5. **Dimensioni.** Invia da UDP un datagramma da 70000 byte. *Soluzione:* errore `OSError: [Errno 90] Message too long` (il massimo è 65507 byte di payload su IPv4).

## Troubleshooting

- `Address already in use`: il server TCP precedente lascia la porta in `TIME_WAIT`; attendi o aggiungi `server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)` prima del `bind`.
- `ConnectionRefusedError`: il server non è avviato, o è già terminato (accetta una sola connessione).
- Il client UDP "funziona" ma il server non stampa nulla: il server era partito dopo, oppure usa un'altra porta.

## Pulizia

Nessun file creato. Processi residui: `Ctrl+C`.
