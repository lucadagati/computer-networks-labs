# TCP vs UDP: trasferimento di un file da 10 MB

Difficoltà: intermedia | Tempo stimato: 40 minuti | Linguaggio: Python 3 (libreria standard)

## Obiettivi didattici

- Confrontare in pratica affidabilità e velocità di TCP e UDP nel trasferimento di un file.
- Verificare che TCP consegna tutti i byte nell'ordine corretto, mentre UDP può perdere datagrammi.
- Capire perché con UDP il **ricevente** non sa quando il trasferimento è finito, se il protocollo applicativo non lo prevede.

## Concetti teorici essenziali

- **TCP**: affidabile (ACK, ritrasmissioni), con controllo di flusso e di congestione: il mittente rallenta se il ricevente o la rete non tengono il passo.
- **UDP**: nessun ACK, nessun controllo di flusso: se il ricevente (o i buffer del kernel) sono saturi, i datagrammi vengono scartati in silenzio.
- Il tempo misurato dal client indica solo quanto ci ha messo a **consegnare i dati al kernel**, non a farli arrivare a destinazione. Sul loopback i tempi sono quindi molto piccoli e poco significativi per TCP.

## Prerequisiti

- Python 3.8+, due terminali, porte 12345/TCP e 12346/UDP libere, circa 30 MB di spazio.

## Struttura dei file

I nomi reali sono diversi da quelli del vecchio README.

| File | Contenuto |
|---|---|
| `createfile.py` | Crea `your_file_to_send` da 10 MiB (10 · 1024 · 1024 byte, tutti `b'0'`). |
| `serverTCP.py` | `start_tcp_server(12345, 'received_tcp_file')`: ascolta su tutte le interfacce, accetta una connessione, scrive su file ogni blocco da 1024 byte finché `recv` restituisce `b''` (fine connessione). |
| `clientTCP.py` | Legge `your_file_to_send` a blocchi da 1024 byte con `sendall` verso `127.0.0.1:12345`, stampa `File sent in X seconds.` |
| `serverUDP.py` | `start_udp_server(12346, 'received_udp_file')`: riceve datagrammi (`recvfrom(1024)`) e li scrive su file. Esce dal ciclo solo se arriva un datagramma **vuoto**. |
| `clientUDP.py` | Invia lo stesso file a blocchi da 1024 byte con `sendto` verso `127.0.0.1:12346`, stampa il tempo. |

Difetti noti del codice (utili da discutere):

- `serverUDP.py` **non termina mai** in condizioni normali: il client non invia un datagramma vuoto finale. Se per caso ne arrivasse uno, il server andrebbe in errore: `conn.close()` fa riferimento a una variabile `conn` mai definita (`NameError`).
- `clientUDP.py` non ritrasmette né rallenta: invia alla massima velocità possibile.

## Procedura passo-passo

```bash
cd Livello_Rete/TCP_6
python3 createfile.py
ls -l your_file_to_send            # 10485760 byte
```

### Prova TCP

```bash
python3 serverTCP.py               # terminale A
python3 clientTCP.py               # terminale B
```

Output verificato (tempi indicativi, dipendono dalla macchina):

```text
# server
TCP Server listening on port 12345
Connected by ('127.0.0.1', 48686)
# client
File sent in 0.0218... seconds.
```

Verifica l'integrità:

```bash
ls -l received_tcp_file            # 10485760 byte
cmp received_tcp_file your_file_to_send && echo IDENTICI
```

Nelle prove i file erano identici.

### Prova UDP

```bash
python3 serverUDP.py               # terminale A (resta attivo)
python3 clientUDP.py               # terminale B
sleep 2; ls -l received_udp_file   # dimensione
```

Output verificato: il client stampa `File sent in 0.079... seconds.` e `received_udp_file` risultava di **10354688 byte invece di 10485760** (circa 1,2% di datagrammi persi, anche su loopback). Il valore cambia a ogni esecuzione e con il carico della macchina. Termina il server con `Ctrl+C`.

## Cosa osservare

- TCP: dimensioni identiche, file identico. UDP: file più corto (perdite) anche su loopback, e il server UDP non termina.
- Per vedere perdite significative con UDP aumenta la dimensione del file o riduci il buffer di ricezione con `server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 4096)` nel server.
- I tempi su loopback non rappresentano le prestazioni su una rete reale.

## Esercizi proposti

1. **Ripeti più volte.** Esegui la prova UDP 5 volte e annota la dimensione ricevuta. *Soluzione:* varia a ogni prova.
2. **Rallenta il client UDP.** Inserisci `time.sleep(0.0001)` ogni 100 blocchi: le perdite diminuiscono? Perché?
3. **Terminazione UDP.** Fai inviare al client UDP un datagramma finale `b''` e correggi `conn.close()` in `server_socket.close()`: ora il server termina.
4. **Sequenza.** Aggiungi un numero di sequenza a 4 byte a ogni datagramma, conta nel server i buchi: hai costruito un rilevatore di perdite.
5. **Cattura.** Con `sudo tcpdump -i lo -n 'port 12345' -c 20` (e su 12346) confronta i segmenti: in TCP compaiono ACK, in UDP no.
6. **Rete emulata.** Se hai `tc` (iproute2): `sudo tc qdisc add dev lo root netem loss 5%` e ripeti le prove, poi ripristina con `sudo tc qdisc del dev lo root`. Non verificato in questo ambiente: attenzione, agisce su tutta l'interfaccia `lo`.

## Troubleshooting

- `FileNotFoundError: your_file_to_send`: esegui prima `createfile.py` dalla cartella `TCP_6`.
- `Address already in use`: server precedente attivo o in `TIME_WAIT`.
- Il client TCP termina ma il server TCP continua: il server termina quando la connessione viene chiusa; controlla di non aver avviato due server.
- Il server UDP "si blocca": è il comportamento previsto (attende altri datagrammi).

## Pulizia

```bash
rm -f your_file_to_send received_tcp_file received_udp_file
rm -rf __pycache__
```

I file generati sono già elencati nel `.gitignore` del repository.
