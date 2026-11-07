# Demo FTP con Docker (server vsftpd + client lftp)

Difficoltà: base-intermedia | Tempo stimato: 45 minuti | Verificato eseguendo i container Docker (login, `ls`, `put`, `get`, log di `vsftpd`)

## Obiettivi didattici

- Avviare un server FTP (`vsftpd`) e un client (`lftp`) in container separati e farli comunicare.
- Capire le **due connessioni** di FTP: controllo (porta 21) e dati (porta 20 in modalità attiva, porte alte in modalità passiva).
- Comprendere chroot degli utenti locali, permessi di scrittura e modalità passiva.
- Osservare che FTP trasmette credenziali in chiaro.

## Concetti teorici essenziali

- **Canale di controllo** (TCP 21): comandi e risposte testuali (`USER`, `PASS`, `LIST`, `STOR`, `RETR`, risposte `230`, `550`...). **Canale dati**: una connessione separata per ogni trasferimento o listing.
- **Modo attivo**: il server apre la connessione dati dalla porta 20 verso il client (problemi con NAT/firewall lato client).
- **Modo passivo** (`PASV`): il server comunica una porta e il client apre la connessione dati; qui il server usa l'intervallo 10090-10100 (`pasv_min_port`/`pasv_max_port`).
- FTP standard non cifra nulla: utente e password viaggiano in chiaro (`ssl_enable=NO`).

## Prerequisiti

- Docker installato (`docker --version`) e permessi per usarlo.
- Rete di accesso a Internet la prima volta (immagine `debian:stable-slim` e pacchetti `vsftpd`, `lftp`).

## Struttura dei file

| File | Cosa fa |
|---|---|
| `server/Dockerfile` | Parte da `debian:stable-slim`, installa `vsftpd`, crea `/var/run/vsftpd/empty`, l'utente `ftpuser` (password `ftppassword`) con home `/home/ftpuser`, la cartella `ftp` (non scrivibile, di `nobody`) con sottocartella `files` di proprietà di `ftpuser`, un file vuoto `test`, copia `vsftpd.conf`, espone `20-21` e `10090-10100`, avvia `vsftpd` in foreground. |
| `server/vsftpd.conf` | Configurazione: niente accesso anonimo (`anonymous_enable=NO`), utenti locali abilitati e scrittura permessa (`local_enable`, `write_enable`), `chroot_local_user=YES` con `allow_writeable_chroot=YES`, passivo abilitato con porte 10090-10100, log dei trasferimenti (`xferlog_enable`), SSL disattivato. |
| `client/Dockerfile` | Parte da `debian:stable-slim`, installa `lftp` e lo usa come `ENTRYPOINT`: tutto ciò che passi dopo il nome dell'immagine diventa argomento di `lftp`. |

Nota: i due Dockerfile sono in sottocartelle diverse, quindi la build va fatta **da `server/` e da `client/`** (la vecchia versione del README eseguiva due `docker build` nella stessa cartella).

## Procedura passo-passo

Usa una rete Docker dedicata, così il client raggiunge il server per nome e la modalità passiva funziona senza configurare IP esterni.

```bash
cd Livello_Applicazione/FTP
docker build -t ftp-server:latest server
docker build -t ftp-client:latest client
docker network create ftp-net
docker run -d --name ftp-server --network ftp-net ftp-server:latest
docker logs ftp-server          # nessun output è normale
```

Verifica che il server sia attivo: `docker ps --filter name=ftp-server` deve mostrare `Up`.

### Sessione client automatica

```bash
docker run --rm --network ftp-net ftp-client:latest \
  -u ftpuser,ftppassword ftp-server \
  -e "cd ftp/files; pwd; put /etc/hostname -o h.txt; ls; bye"
```

Output verificato (le date cambiano):

```text
ftp://ftpuser:ftppassword@ftp-server/ftp/files
-rw-r--r--    1 1000     1000           13 Nov 07 12:28 h.txt
```

Aggiungendo un `ls` all'inizio della sequenza vedi la radice, che contiene `ftp` (`dr-xr-xr-x`, proprietario 65534/`nobody`) e il file vuoto `test` (verificato).

### Sessione interattiva

```bash
docker run -it --rm --network ftp-net ftp-client:latest -u ftpuser,ftppassword ftp-server
```

Al prompt `lftp ftpuser@ftp-server:~>` prova:

```text
ls                       # elenca /: ftp e test
cd ftp/files             # cartella scrivibile
put /etc/hostname        # carica un file locale del container client
ls
get hostname -o /tmp/copia.txt
bye
```

### Log lato server

```bash
docker exec ftp-server tail -5 /var/log/vsftpd.log
```

Mostra righe come `OK LOGIN: Client "::ffff:172.x.x.x"`, `OK UPLOAD: ... "/up.txt", 9 bytes` e `OK DOWNLOAD: ...` (verificate).

### Variante originale con porte pubblicate sull'host (non verificata)

```bash
docker rm -f ftp-server
docker run -d --name ftp-server -p 20-21:20-21 -p 10090-10100:10090-10100 ftp-server:latest
docker run -it --rm --network host ftp-client:latest ftp://ftpuser:ftppassword@localhost
```

Il server non imposta `pasv_address`: in modalità passiva annuncia l'IP interno del container, quindi l'elenco/trasferimento può bloccarsi quando il client non è sulla stessa rete Docker. Non verificato in questo ambiente.

## Cosa osservare

- **Chroot**: la radice vista dal client è `/home/ftpuser`, non `/`. `ftp` non è scrivibile (`dr-xr-xr-x`), `ftp/files` sì. Il `put` nella radice funziona comunque grazie a `allow_writeable_chroot=YES` (verificato: `up.txt` è stato caricato in `/`).
- Un `cd files` dalla radice dà `550 Failed to change directory`: la cartella è `ftp/files` (errore reale riscontrato durante la verifica).
- Login anonimo: `lftp -u anonymous,x ...` risponde `530 Login incorrect` (verificato).
- `lftp` stampa solo l'esito; per vedere i comandi grezzi usa `-d` (debug): `lftp -d -u ftpuser,ftppassword ftp-server`.

## Esercizi proposti

1. **Cattura del traffico in chiaro.** Su un host con `tcpdump`: `sudo tcpdump -i any -A 'tcp port 21'` e fai il login. *Soluzione:* vedi `USER ftpuser` e `PASS ftppassword`.
2. **Passivo vs attivo.** Avvia lftp con `-e "set ftp:passive-mode off; ls; bye"`. Cosa cambia? *Soluzione attesa:* il server prova ad aprire la connessione dati dal client; nella rete Docker può funzionare, dietro NAT no (non verificato).
3. **Trova le porte dati.** Con `lftp -d` individua la risposta `227 Entering Passive Mode (...)` e decodifica la porta (`p1*256+p2`): deve essere tra 10090 e 10100.
4. **Permessi.** Prova `put` in `ftp` (non scrivibile) e in `ftp/files`. Spiega la differenza leggendo `server/Dockerfile`.
5. **Aggiungi un secondo utente.** Modifica `server/Dockerfile` con un altro `useradd` e riesegui la build.
6. **Usa FTP cifrato.** Abilita `ssl_enable=YES` (certificato snakeoil già referenziato) e collegati con `set ftp:ssl-force true`; cosa vedi nella cattura?

## Troubleshooting

- `docker: Cannot connect to the Docker daemon`: avvia il servizio Docker o usa `sudo`.
- `ftp-server` si ferma subito: `docker logs ftp-server`; `vsftpd.conf` con errori di sintassi viene riportato lì.
- `Login failed: 530`: utente/password sbagliati (`ftpuser` / `ftppassword`), oppure login anonimo (disabilitato).
- `Access failed: 550 Failed to change directory`: percorso sbagliato, usa `cd ftp/files`.
- Elenco che si blocca: problema di modalità passiva/porte (vedi variante originale).
- Porta 21 già usata: usa la rete Docker senza `-p`, come nella procedura principale.

## Pulizia

```bash
docker rm -f ftp-server
docker network rm ftp-net
docker rmi ftp-server:latest ftp-client:latest
```
