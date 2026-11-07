# Telnet vs SSH: confronto del traffico in chiaro e cifrato

Difficoltà: intermedia | Tempo stimato: 60 minuti | Parzialmente verificato: **server SSH verificato** (avvio manuale di `sshd`, login con chiave); **non verificata** la build di `Dockerfile.client` (installa Wireshark, pacchetto molto grande) e la parte Telnet del `Dockerfile.server` (vedi sotto)

## Obiettivi didattici

- Confrontare Telnet e SSH catturando il traffico di una sessione.
- Capire che SSH offre cifratura, autenticazione del server (chiavi host) e integrità.
- Imparare a leggere un'acquisizione di pacchetti per riconoscere dati in chiaro.

## Concetti teorici essenziali

- **Telnet** (TCP 23): testo in chiaro, nessuna autenticazione del server.
- **SSH** (TCP 22): handshake con scambio di chiavi (Diffie-Hellman), verifica della chiave host del server (*known_hosts*), poi payload cifrato e autenticato; permette anche trasferimento file e tunneling.
- Con uno sniffer in Telnet vedi username, password e comandi; in SSH vedi solo il banner di versione iniziale (`SSH-2.0-OpenSSH_...`) e dati cifrati.

## Prerequisiti

- Docker. Conoscenza di base di `tcpdump`/Wireshark.
- Per la parte Telnet usa l'immagine di [`../demo_telnet`](../demo_telnet/) (verificata).

## Struttura dei file

| File | Cosa fa |
|---|---|
| `Dockerfile.server` | `ubuntu:latest`; installa `xinetd`, `telnetd` e `openssh-server`; esegue `RUN service ssh start` **durante la build** (non ha effetto a runtime); `CMD ["tail", "-f", "/dev/null"]` lascia il container inattivo. |
| `Dockerfile.client` | `ubuntu:latest`; installa `openssh-client`, `telnet` e `wireshark` (con la risposta preconfigurata per l'installazione non interattiva); `CMD tail -f /dev/null`. |

### Cosa è stato verificato sul server (immagine costruita ed esaminata)

- Il container parte e resta in esecuzione (`tail -f /dev/null`), ma **non c'è nessun `sshd` in ascolto** e **nessun utente** oltre a root: `service ssh start` nel `RUN` serve solo in fase di build.
- Il binario `/usr/sbin/sshd` esiste. Con questi passi manuali SSH funziona (verificato, OpenSSH 10.x su Ubuntu 26.04):

  ```bash
  docker exec ssh-server sh -c 'useradd -m demo && echo "demo:demo123" | chpasswd && mkdir -p /run/sshd && /usr/sbin/sshd'
  ```

  Poi un login riuscito con una chiave (la password richiede `sshpass` o un terminale interattivo) e un tentativo non autenticato con `BatchMode`, che risponde `Permission denied (publickey,password)`.
- `telnetd` è `inetutils-telnetd` (`/usr/sbin/telnetd`) e `xinetd` è installato, ma **non esiste** `/etc/xinetd.d/telnet`: senza configurare xinetd (o inetd) **il server Telnet non è attivo**. Per la parte Telnet usa quindi `../demo_telnet`.

## Procedura passo-passo

```bash
cd Livello_Applicazione/telnet_vs_ssh
docker network create tvs-net
docker build -t tvs-server -f Dockerfile.server .
docker run -d --name ssh-server --network tvs-net tvs-server
docker exec ssh-server sh -c 'useradd -m demo && echo "demo:demo123" | chpasswd && mkdir -p /run/sshd && /usr/sbin/sshd'
```

Client: costruire `Dockerfile.client` richiede molto tempo e spazio per Wireshark (non verificato). In alternativa usa un client leggero:

```bash
docker build -t tvs-client -f Dockerfile.client .      # come da repository (non verificato)
docker run -d --name ssh-client --network tvs-net tvs-client
docker exec -it ssh-client ssh demo@ssh-server          # password: demo123
```

Alla prima connessione compare la richiesta di fiducia nella chiave host (`Are you sure you want to continue connecting (yes/no)?`); rispondi `yes`.

### Catturare e confrontare

1. Nel server installa lo sniffer: `docker exec ssh-server sh -c 'apt-get update -qq && apt-get install -y -qq tcpdump'`.
2. `docker exec ssh-server tcpdump -l -A -n -i eth0 port 22` mentre fai login da `ssh-client`: vedi la riga iniziale `SSH-2.0-OpenSSH_...` e poi dati illeggibili (comportamento atteso, non catturato qui).
3. Ripeti con Telnet usando `../demo_telnet` (container `telnet-server` sulla stessa rete): nella cattura sono leggibili utente e password (verificato con `tcpdump -A` su quell'immagine).
4. In Wireshark (sul client con GUI o analizzando un pcap) usa il filtro `tcp.port == 22` e `telnet`; per Telnet `Follow TCP Stream` mostra la sessione intera.

## Cosa osservare

- Telnet: ogni carattere digitato appare in un pacchetto; la password è leggibile.
- SSH: dopo il banner, `Key Exchange Init`, scambio di chiavi e poi pacchetti `Encrypted packet`.
- La chiave host viene salvata in `~/.ssh/known_hosts`: ricreando il server, la chiave cambia e SSH avvisa con `REMOTE HOST IDENTIFICATION HAS CHANGED`.

## Esercizi proposti

1. Individua il banner di versione SSH nella cattura e indica quale versione del software usa il server.
2. Misura la differenza di numero di pacchetti per digitare lo stesso comando in Telnet e in SSH.
3. Esegui `ssh -v demo@ssh-server` e leggi gli algoritmi negoziati (scambio chiavi, cifrario, MAC).
4. Prova l'accesso con chiave: `ssh-keygen`, `ssh-copy-id`, poi disabilita la password in `sshd_config` (`PasswordAuthentication no`).
5. Tunneling: `ssh -L 8080:localhost:80 demo@ssh-server` e spiega a cosa serve.

## Troubleshooting

- `ssh: connect to host ssh-server port 22: Connection refused`: `sshd` non avviato (passo manuale sopra).
- `Missing privilege separation directory: /run/sshd`: manca `mkdir -p /run/sshd`.
- `Permission denied`: utente/password errati o creati nel container sbagliato.
- La build del client è lunghissima o fallisce: non è necessario Wireshark nel container; analizza con `tcpdump` nel server e apri il pcap sull'host.
- Telnet non risponde nel server: non configurato (vedi sopra).

## Pulizia

```bash
docker rm -f ssh-server ssh-client
docker network rm tvs-net
docker rmi tvs-server tvs-client 2>/dev/null
```
