# Demo P2P_2: scambio di file tra peer (TCP) con Docker

Difficoltà: intermedia | Tempo stimato: 45 minuti | Verificato con container `docker run` su rete Docker (server e client, richiesta di file esistente e inesistente); `docker-compose.yml` non eseguito (porte host 8001-8004 e volumi `./data/...`), il suo uso è dedotto dalla lettura

## Obiettivi didattici

- Capire il modello P2P in cui lo stesso programma può fare da **server** (offre file) e da **client** (li richiede).
- Studiare un semplice protocollo applicativo su TCP: il client invia il nome del file, il server risponde con il contenuto e chiude la connessione.
- Usare Docker Compose, volumi e porte per far comunicare più peer.

## Concetti teorici essenziali

- Ogni peer è sia client sia server (qui scelto con il primo argomento: `server` o `client`).
- TCP come trasporto affidabile; la **chiusura della connessione** da parte del server segnala la fine del file (nessuna lunghezza trasmessa).
- Un thread per ogni client (`threading.Thread`) permette di servire più richieste contemporaneamente.

## Prerequisiti

- Docker (e Compose per la variante a quattro peer), Python non necessario sull'host.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `p2pnode.py` | Funzioni chiave: `handle_client(sock)`: legge il nome del file (`recv(1024)`), lo apre in binario e lo invia con `sendfile`, stampa l'esito, chiude. `server_loop(port)`: `bind(('0.0.0.0', port))`, `listen(5)`, `accept()` in ciclo e un thread per connessione. `main(...)`: modalità `server` oppure `client`; il client si connette, invia il nome del file, **scrive la risposta in un file locale con lo stesso nome** e stampa `Ricevuto file: <nome>`. Argomenti: `server <porta>` (2 argomenti) oppure `client <file> <host> <porta>` (4 argomenti). |
| `Dockerfile` | `python:3.8-slim`, `WORKDIR /app`, copia `p2pnode.py`, `ENTRYPOINT ["python", "./p2pnode.py"]`: gli argomenti passati a `docker run` diventano gli argomenti dello script. |
| `docker-compose.yml` | Quattro servizi `p2pnode1..4`, ciascuno con volume `./data/nodeN:/data`, porta host `800N` mappata su 8000 e comando `server 8000`. |

Il vecchio README citava un `peer.py` che non esiste: il file reale è `p2pnode.py`.

## Procedura passo-passo (verificata con `docker run`)

```bash
cd Livello_Applicazione/P2P_2
docker build -t p2p-node .
docker network create p2p-net
docker run -d --name peer1 --network p2p-net p2p-node server 8000
docker run -d --name peer2 --network p2p-net p2p-node server 8000

# crea un file da condividere dentro peer1
docker exec peer1 sh -c 'echo "contenuto del peer1" > /app/file1.txt'

# peer3 (client temporaneo) scarica il file da peer1
docker run --rm --network p2p-net --entrypoint sh p2p-node -c \
  'python ./p2pnode.py client file1.txt peer1 8000; cat file1.txt'
```

Output atteso (verificato):

```text
Ricevuto file: file1.txt
contenuto del peer1
```

Il file richiesto è cercato dal server **relativamente a `/app`** (la sua directory di lavoro).

### Variante con Compose (dedotta, non eseguita)

```bash
docker compose up --build -d
mkdir -p data/node1 && echo "ciao" > data/node1/a.txt     # visibile in /data del peer 1
# nodo 2 scarica /data/a.txt dal nodo 1: salverà il file in ./data/node2/a.txt
docker compose run --rm --no-deps p2pnode2 client /data/a.txt p2pnode1 8000
```

Poiché client e server usano lo stesso nome di file (remoto e locale), con il percorso `/data/...` il file ricevuto finisce nel volume del nodo che esegue il client. Dall'host i server sono anche raggiungibili su `localhost:8001..8004`.

## Cosa osservare

- **Buffer dell'output**: `docker logs peer1` può risultare **vuoto** (verificato): Python bufferizza `print` quando stdout non è un terminale, quindi le righe `Ascolto come server sulla porta 8000` e `Accettata connessione da ...` non compaiono subito. Rimedio atteso: avviare con `-e PYTHONUNBUFFERED=1` (non verificato).
- **File inesistente**: il server stampa l'errore nel suo log, chiude la connessione e il client crea comunque un file **vuoto** e stampa `Ricevuto file: ...` (verificato per `mancante.txt`, 0 byte).
- Il server non invia la lunghezza né un codice di errore: il client non può distinguere un file vuoto da uno inesistente.

## Esercizi proposti

1. **Protocollo di errore.** Fai rispondere il server con un primo byte `0`/`1` (assente/presente) e il client reagire di conseguenza.
2. **Più sorgenti.** Scrivi un client che chiede lo stesso file a `peer1` e a `peer2`, prendendo la prima risposta valida (idea alla base del download P2P).
3. **Divisione in parti.** Fai richiedere a ciascun peer una parte diversa del file (offset e lunghezza) e ricomponi: è la logica di BitTorrent.
4. **Sicurezza.** Cosa succede se il client chiede `../../etc/passwd`? Spiega la vulnerabilità (*path traversal*) e correggi con `os.path.basename` e una cartella radice fissa.
5. **Cattura.** `docker run --rm --net container:peer1 nicolaka/netshoot tcpdump -n -A port 8000` (o `tcpdump` installato nel container) durante un download: individua il nome del file e il contenuto in chiaro.

## Troubleshooting

- `Connection refused`: server non avviato o nome/porta errati (`docker ps`, `docker logs`).
- `Name or service not known` per `peer1`: i container non sono sulla stessa rete Docker definita dall'utente.
- Il client "si blocca": il server non chiude la connessione (errore nel server); controlla `docker logs`.
- `Bind for 0.0.0.0:8001 failed: port is already allocated` con Compose: cambia le porte host.
- `FileNotFoundError` nel client: la directory locale non esiste (es. `/data` senza volume).

## Pulizia

```bash
docker rm -f peer1 peer2
docker network rm p2p-net
docker rmi p2p-node
# variante Compose
docker compose down && rm -rf data
```
