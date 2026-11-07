# Demo P2P (scheletro): quattro nodi su una rete Docker con Compose

Difficoltà: base | Tempo stimato: 30 minuti | Verificato: build dell'immagine ed esecuzione di `main.py` con `docker run` (il file `docker-compose.yml` non è stato eseguito perché i nomi fissi `node1`-`node4` possono collidere con altri container)

## Obiettivi didattici

- Capire come descrivere una rete di nodi con `docker-compose.yml` (servizi, rete condivisa, nomi dei container).
- Preparare un'infrastruttura su cui implementare un protocollo peer-to-peer.
- Vedere la differenza con l'architettura client-server (nessun nodo centrale, ogni nodo è sia client sia server).

## Concetti teorici essenziali

- In una rete **P2P** i peer comunicano direttamente tra loro; non c'è un server dedicato (o ce n'è uno solo per la scoperta dei peer).
- Con una rete Docker definita dall'utente i container si raggiungono per **nome** (DNS interno di Docker).

## Prerequisiti

- Docker con il plugin Compose (`docker compose version`) oppure il vecchio `docker-compose`.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `Dockerfile` | Immagine `python:3.9`, copia il codice in `/app`, esegue `pip install asyncio` (il pacchetto è in realtà già nella libreria standard; l'installazione è superflua ma innocua) e avvia `python main.py`. |
| `main.py` | **Scheletro**: una coroutine `main()` che stampa `Hello from main`. Il commento `# Codice per la simulazione P2P` indica dove implementare il comportamento dei peer. Non c'è ancora alcuna comunicazione tra nodi. |
| `docker-compose.yml` | Quattro servizi `node1`..`node4` costruiti dalla stessa immagine (`build: .`), con `container_name` omonimo, tutti sulla rete `p2pnet`. |

## Procedura passo-passo

```bash
cd Livello_Applicazione/P2P
docker compose build            # (o docker-compose build)
docker compose up
```

Output atteso: quattro righe `Hello from main`, una per nodo (prefissate da `node1-1  |` ecc. a seconda della versione di Compose); poi i container terminano con codice 0 perché `main.py` finisce subito. `docker compose` può anche segnalare che l'attributo `version` è obsoleto: è solo un avviso.

Verifica eseguita con un container singolo:

```bash
docker build -t p2p-demo .
docker run --rm p2p-demo        # stampa: Hello from main
```

## Cosa osservare

- I container terminano subito: per vederli attivi, `main.py` deve contenere un server e un ciclo di attesa.
- Con `docker network ls` vedi la rete `<progetto>_p2pnet` creata da Compose.

## Esercizi proposti

1. **Rendi i nodi attivi.** In `main()` aggiungi `await asyncio.sleep(3600)` e riavvia: `docker compose ps` mostra quattro container `Up`.
2. **Server echo asincrono.** Con `asyncio.start_server` fai ascoltare ogni nodo sulla porta 9000 e rispondere con eco.
3. **Scoperta dei peer.** Ogni nodo prova a connettersi a `node1`..`node4` (escluso sé stesso) e stampa chi risponde.
4. **Gossip.** Un nodo invia un messaggio a due vicini casuali che lo inoltrano: contare quanti round servono a raggiungere tutti.
5. **Confronto.** Implementa la stessa funzione con un server centrale e misura cosa succede se il server si ferma (vedi `../P2P_2` per un esempio di trasferimento file tra peer).

## Troubleshooting

- `container name "/node1" is already in use`: esiste già un container con quel nome (anche fermo): `docker rm node1` solo se è il tuo, oppure cambia `container_name` in Compose.
- `docker-compose: command not found`: usa `docker compose` (plugin v2).
- Dopo modifiche a `main.py` il vecchio codice viene eseguito: ricostruisci con `docker compose up --build`.

## Pulizia

```bash
docker compose down
docker rmi p2p-demo 2>/dev/null     # solo se hai creato l'immagine di prova
```
