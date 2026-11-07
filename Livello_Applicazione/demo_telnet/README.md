# Demo Telnet: server telnetd in un container (e perché è insicuro)

Difficoltà: base | Tempo stimato: 30 minuti | Verificato eseguendo il container: login `demo`/`demo123` da un client telnet e cattura dei pacchetti con `tcpdump` (credenziali visibili)

La traccia originale del laboratorio è in [`../../docs/lab-livello-applicazione-2023-10-17.pdf`](../../docs/lab-livello-applicazione-2023-10-17.pdf).

## Obiettivi didattici

- Usare `telnet` come client di terminale remoto e come strumento per testare servizi TCP su porte diverse.
- Dimostrare che Telnet trasmette utente e password **in chiaro**.
- Preparare il confronto con SSH (`../telnet_vs_ssh`).

## Concetti teorici essenziali

- Telnet (porta TCP 23) offre una sessione di terminale remota senza cifratura né autenticazione forte.
- Ogni tasto digitato viene inviato in un pacchetto separato, quindi uno sniffer può ricostruire utente e password.
- Come strumento diagnostico, `telnet host porta` apre una connessione TCP grezza: se si connette, la porta è aperta (e puoi parlare un protocollo testuale come HTTP o SMTP).

## Prerequisiti

- Docker; un client `telnet` (host: `sudo apt install telnet`, oppure usa un altro container).

## Struttura dei file

| File | Cosa fa |
|---|---|
| `Dockerfile` | **Base `ubuntu:22.04`**, installa `telnetd`, `curl`, `wget`, `dnsutils`; crea l'utente `demo` con password `demo123`; avvia `/usr/sbin/in.telnetd -debug 23` in foreground. |
| `index.html` | Pagina HTML di esempio **non usata** da questo Dockerfile (copia della demo web). |

Modifica rispetto all'originale: il Dockerfile usava `FROM ubuntu:latest`. Con le versioni attuali di Ubuntu (verificato su 26.04) il pacchetto `telnetd` è solo un pacchetto di transizione che installa `inetutils-telnetd`, e il binario `/usr/sbin/in.telnetd` **non esiste**: il container usciva subito con `exec: "/usr/sbin/in.telnetd": stat ... no such file or directory`. La base è quindi fissata a `ubuntu:22.04`, dove `in.telnetd` esiste (verificato).

## Procedura passo-passo

```bash
cd Livello_Applicazione/demo_telnet
docker build -t telnet-server .
docker network create telnet-net
docker run -d --name telnet-server --network telnet-net telnet-server
```

Da un altro container con client telnet (qui riuso del client dell'immagine `smtp-demo`, in alternativa installa `telnet` sull'host e usa `-p 23:23`):

```bash
docker run --rm -it --network telnet-net ubuntu:22.04 bash -c "apt-get update -qq && apt-get install -y -qq telnet >/dev/null && telnet telnet-server 23"
```

Atteso (verificato):

```text
Connected to telnet-server.
Ubuntu 22.04.5 LTS
<id-container> login: demo
Password: demo123
Welcome to Ubuntu 22.04.5 LTS ...
demo@<id-container>:~$
```

Con la variante dell'originale (porta pubblicata): `docker run -d --name telnet-server -p 23:23 telnet-server` e `telnet localhost 23`; la porta 23 dell'host deve essere libera (non verificato).

**Comportamento da conoscere (verificato):** con `-debug` `in.telnetd` serve **una sola sessione** e poi termina, quindi il container passa a `Exited (0)` dopo il primo logout. Per una nuova prova: `docker rm telnet-server` e ricrealo (oppure usa `--restart always`).

## Cattura delle credenziali

In un terminale `docker exec` dentro il client (o sull'host sul bridge Docker) lancia `tcpdump -l -A -n -i eth0 port 23` prima del login. Nella verifica l'output contiene il testo `demo` e `demo123` inviati un carattere per volta (`d`, `e`, `m`, `o`...), cioè in chiaro.

## Cosa osservare

- Le credenziali compaiono nel payload TCP dei pacchetti, un carattere per segmento.
- Con `-debug` il server è avviato in modalità standalone; puoi guardare `docker logs telnet-server` (contenuto non verificato).
- Il server richiede una sessione completa con `login`: l'utente `demo` esiste solo in questo container.

## Esercizi proposti

1. **Test di una porta.** `telnet telnet-server 23` collega, `telnet telnet-server 80` no (connessione rifiutata): è un modo semplice per verificare i servizi.
2. **HTTP a mano.** Se hai un server web (vedi `../demo_webserver`): `telnet web-demo 80`, poi digita `GET / HTTP/1.0` e due volte Invio. Vedi la risposta HTML (non eseguito in questo ambiente per questa demo).
3. **Strumenti DNS.** Dentro il container (`docker exec -it ... bash` prima di terminare la sessione): `dig`, `nslookup` (installati per il laboratorio sul DNS).
4. **Confronto con SSH.** Segui `../telnet_vs_ssh` e ripeti la cattura: cosa si legge?
5. **Rendilo persistente.** Come servire più sessioni? (usa `inetd`/`xinetd` come nell'immagine `telnet_vs_ssh/Dockerfile.server`.)

## Troubleshooting

- Il container esce subito con `no such file or directory`: stai usando una base Ubuntu recente; usa `ubuntu:22.04`.
- `Connection refused`: sessione già consumata (container terminato) o rete errata.
- `login incorrect`: utente `demo`, password `demo123`.
- Usa l'utente `demo`: è l'unico creato dal Dockerfile (il login di `root` non è stato provato).

## Pulizia

```bash
docker rm -f telnet-server
docker network rm telnet-net
docker rmi telnet-server
```
