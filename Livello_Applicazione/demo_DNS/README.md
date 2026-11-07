# Demo DNS: server autorevole BIND9 per la zona `demo.test`

Difficoltà: intermedia | Tempo stimato: 45 minuti | Verificato eseguendo il container e interrogandolo con `dig` e `nslookup` da dentro e da un altro container della stessa rete Docker

## Obiettivi didattici

- Configurare un server DNS autorevole con BIND9 e una zona master.
- Interrogare il DNS con `dig`/`nslookup` e leggere header, sezioni e flag della risposta.
- Distinguere record `SOA`, `NS`, `A` e il codice `NXDOMAIN`.

## Concetti teorici essenziali

- **Zona**: porzione dello spazio dei nomi gestita da un server (qui `demo.test`).
- **Record**: `SOA` (autorità della zona, serial, timer), `NS` (server autorevoli), `A` (nome -> IPv4).
- **Flag** nella risposta: `qr` (è una risposta), `aa` (autorevole), `rd`/`ra` (ricorsione richiesta/disponibile).
- `NXDOMAIN`: il nome non esiste nella zona.
- Il DNS usa UDP/53 (e TCP/53 per risposte grandi e trasferimenti di zona).

## Prerequisiti

- Docker. Per interrogare dall'esterno servono `dig` (pacchetto `dnsutils`/`bind9-dnsutils`) oppure un container con `dig`.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `Dockerfile` | `ubuntu:20.04`, installa `bind9`, `dnsutils`, `nano`, `net-tools`; prepara `/run/named`; copia i tre file di configurazione in `/etc/bind`; espone 53/udp e 53/tcp; avvia `named -g` (foreground, log su stderr). |
| `named.conf.local` | Dichiara la zona `"demo.test"` di tipo `master`, con file `/etc/bind/db.demo.local`. |
| `db.demo.local` | File di zona: `$TTL 604800`; `SOA demo.test. root.demo.test.` con serial `20231016`; `NS ns.demo.test.`; `ns A 192.0.2.2`; `server A 192.0.2.3`. Gli indirizzi `192.0.2.0/24` sono riservati alla documentazione (RFC 5737). |
| `named.conf.options` | Ascolta su tutte le interfacce IPv4 (`listen-on { any; }`), IPv6 disattivato, `allow-query { any; }`, `recursion yes`, `dnssec-validation no`, `auth-nxdomain no`. |

Attenzione: la zona si chiama **`demo.test`** (anche se il file si chiama `db.demo.local`). Il record `ns` punta a `192.0.2.2`, che non è l'indirizzo reale del container: è solo un dato di esempio.

## Procedura passo-passo

```bash
cd Livello_Applicazione/demo_DNS
docker build -t dns-demo .
docker run -d --name dns-demo -p 5353:53/udp -p 5353:53/tcp dns-demo
docker logs dns-demo | tail -3
```

Output atteso nei log (verificato): `zone demo.test/IN: loaded serial 20231016`, `all zones loaded`, `running`.

### Interrogazioni (verificate dall'interno del container)

```bash
docker exec dns-demo dig @127.0.0.1 server.demo.test +noall +answer
docker exec dns-demo dig @127.0.0.1 ns.demo.test +short           # 192.0.2.2
docker exec dns-demo dig @127.0.0.1 demo.test NS +short           # ns.demo.test.
docker exec dns-demo dig @127.0.0.1 demo.test SOA +short
docker exec dns-demo dig @127.0.0.1 nonesiste.demo.test           # NXDOMAIN
```

Risultati verificati:

```text
server.demo.test.   604800  IN  A   192.0.2.3
demo.test. root.demo.test. 20231016 604800 86400 2419200 604800        (SOA)
status: NXDOMAIN ... flags: qr aa rd ra                                  (nome inesistente)
```

Dall'host (porta mappata 5353, non verificato per il mapping, ma il servizio è stato verificato dalla rete Docker): `dig @127.0.0.1 -p 5353 server.demo.test`. Da un altro container: `docker run --rm --network container:dns-demo ...` oppure su una rete definita dall'utente `dig @dns-demo server.demo.test` (verificato con `dig` e `nslookup`).

## Cosa osservare

- Nella risposta a `server.demo.test` compare il flag `aa` (risposta autorevole) e il TTL 604800 (7 giorni).
- Il nome inesistente produce `NXDOMAIN` e nella sezione AUTHORITY il record `SOA` (serve alla *negative caching*).
- Il `SERVER:` stampato da `dig` indica quale server ha risposto, con protocollo UDP.
- Domande fuori zona (es. `www.example.com`): il server ha `recursion yes`, quindi tenta la ricorsione; **nel nostro ambiente di prova la risoluzione verso Internet è andata in timeout** (`communications error ... timed out`), dipende dalla rete del tuo host (esito non garantito).

## Esercizi proposti

1. **Aggiungi un record.** In `db.demo.local` aggiungi `www IN A 192.0.2.10` e **incrementa il serial** (es. `20231017`); ricostruisci e interroga `www.demo.test`.
2. **Alias.** Aggiungi `web IN CNAME server` e interroga `web.demo.test`: vedrai CNAME e A.
3. **Record MX.** Aggiungi `@ IN MX 10 mail.demo.test.` e `mail IN A 192.0.2.4` (collegamento con `../demo_smtp`).
4. **Cattura.** `docker exec dns-demo apt-get install -y tcpdump` (o dall'host su `docker0`) e osserva query/risposte UDP 53 con Wireshark: quanti byte sono? Confronta con `Wireshark/UDP/udp.pcap` (DNS su UDP).
5. **Forza TCP.** `dig @127.0.0.1 server.demo.test +tcp`: cambia il protocollo mostrato in `SERVER:`.
6. **Reverse DNS.** Crea una zona `2.0.192.in-addr.arpa` con `PTR` per `192.0.2.3`.

## Troubleshooting

- Container che si ferma: `docker logs dns-demo`; un errore nel file di zona (punto finale mancante nei nomi, serial non intero) impedisce il caricamento della zona (`zone ... loading failed`).
- `dig` risponde `connection timed out; no servers could be reached`: porta/rete sbagliata (`-p 5353` per la porta mappata).
- Modifica non vista: serve ricostruire l'immagine (i file sono copiati nell'immagine) e incrementare il serial.
- Porta 53 occupata sull'host (es. `systemd-resolved`): per questo si usa 5353.

## Pulizia

```bash
docker rm -f dns-demo
docker rmi dns-demo
```
