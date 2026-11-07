# Mininet demo4: port forwarding (DNAT) con un router Linux

Difficoltà: intermedia-avanzata | Tempo stimato: 60 minuti | **Mininet non installato in questo ambiente**: gli script Mininet non sono stati eseguiti. La logica `iptables` e il problema indicato sotto sono stati però **verificati con network namespace Linux equivalenti** (topologia ricostruita a mano con `ip netns`, stesso script `router_iptables_port_forwarding.sh`).

## Obiettivi didattici

- Configurare un port forwarding: il router espone la porta 8080 e inoltra al server web interno su 192.168.1.2:80 (DNAT).
- Distinguere le catene `PREROUTING` (DNAT), `FORWARD` (filtro) e `POSTROUTING` (NAT sorgente).
- Imparare a diagnosticare un laboratorio che "non funziona" controllando indirizzi e interfacce.

## Concetti teorici essenziali

- **DNAT**: riscrive l'indirizzo/porta di **destinazione** all'arrivo (prima del routing). Le risposte tornano al client perché il kernel ricorda la traduzione (connection tracking).
- Un host interno senza IP pubblico diventa raggiungibile dall'esterno tramite `IP_router:porta_pubblica`.

## Prerequisiti

- Mininet, OVS, controller, `sudo` (vedi [`../README.md`](../README.md)); `curl`; `python3` (per il web server).

## Struttura dei file

| File | Contenuto |
|---|---|
| `mininet_port_forwarding_demo.py` | Un solo switch `s1` con **tre** nodi collegati: `h1` (`192.168.1.2/24`, server web interno), router `r0` e `h2` (`10.0.0.2/24`, host esterno). Dopo l'avvio: `ifconfig r0-eth0 192.168.1.1/24`, `ifconfig r0-eth1 10.0.0.1/24`, esegue `./router_iptables_port_forwarding.sh` su `r0`, imposta i gateway (`h1` via `192.168.1.1`, `h2` via `10.0.0.1`) e apre il CLI. |
| `router_iptables_port_forwarding.sh` | Forwarding on; svuota `filter`/`nat`; `MASQUERADE` su `-o eth0`; `PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80`; `FORWARD` ACCEPT per `-d 192.168.1.2 --dport 80`. |

## Problema noto: `r0-eth1` non esiste (verificato)

Nello script Python `r0` ha **un solo** link (verso `s1`), quindi esiste solo `r0-eth0`. Il comando `router.cmd('ifconfig r0-eth1 10.0.0.1/24')` fallisce senza messaggi e il router **non ha mai l'indirizzo `10.0.0.1`**. Di conseguenza il test della versione originale:

```text
mininet> h2 curl 10.0.0.1:8080
```

**non funziona** (nella prova con namespace: `timed out`). Inoltre:

- `h1`, `r0` e `h2` stanno sullo stesso segmento L2 anche se con subnet IP diverse.
- Le regole con `-o eth0` (MASQUERADE) non corrispondono a nessuna interfaccia reale (`r0-eth0`), ma non sono necessarie in questo scenario.
- Gli script nel repository non sono eseguibili (`chmod +x router_iptables_port_forwarding.sh`) e vanno lanciati dalla cartella `mininet/demo4`.

### Correzione verificata

Assegna a `r0-eth0` un secondo indirizzo nella subnet di `h2`:

```text
mininet> r0 ip addr add 10.0.0.1/24 dev r0-eth0
mininet> h1 python3 -m http.server 80 &
mininet> h2 curl -s 10.0.0.1:8080 | head -5
```

Nella prova con namespace equivalenti questa sequenza ha restituito HTTP 200 con la pagina dell'indice di directory del server Python, e i contatori hanno mostrato `DNAT ... dpt:8080 to:192.168.1.2:80` con 1 pacchetto e la regola `FORWARD ... 192.168.1.2 dpt:80` con 8 pacchetti.

Per risolvere stabilmente: nello script Python aggiungi, dopo `router.cmd('ifconfig r0-eth0 ...')`, una riga `router.cmd('ip addr add 10.0.0.1/24 dev r0-eth0')` oppure modifica la topologia con un secondo switch.

## Procedura passo-passo

```bash
cd mininet/demo4
chmod +x router_iptables_port_forwarding.sh
sudo python3 mininet_port_forwarding_demo.py
```

Nel CLI:

1. Indirizzi: `r0 ip addr`: atteso `r0-eth0` con `192.168.1.1/24`; applica la correzione sopra per ottenere anche `10.0.0.1/24`.
2. Server web interno: `h1 python3 -m http.server 80 &`; test locale `h1 curl -s localhost | head -3`.
3. Test del forwarding da `h2`: `h2 curl -s 10.0.0.1:8080 | head -3`. Atteso: la stessa pagina di `h1`.
4. Regole e contatori: `r0 iptables -t nat -L -v -n`, `r0 iptables -L FORWARD -v -n`.
5. Per vedere la traduzione: `h1 tcpdump -n -i h1-eth0 port 80 -c 4 &` e ripeti il passo 3: atteso: `h1` vede la connessione provenire da `10.0.0.2` (nessun SNAT necessario perché `h1` risponde tramite il router; non verificato con tcpdump).
6. Prova anche `h2 curl 10.0.0.1:80` (senza forwarding): `Connection refused` o timeout, perché sul router non c'è un server sulla porta 80.

## Cosa osservare

- Il contatore `pkts` della regola DNAT cresce a ogni nuova connessione (conta solo il primo pacchetto, il resto usa conntrack).
- La porta 8080 non esiste su `h1`: è una porta "pubblica" del router.

## Esercizi proposti

1. **Seconda porta**: avvia un secondo server su `h1` (`h1 python3 -m http.server 8000 &`) e aggiungi una regola `DNAT` per inoltrare la porta `9090` del router verso `192.168.1.2:8000`, con la relativa regola `FORWARD`.
2. **Rimuovi la regola `FORWARD`**: funziona ancora? *Soluzione verificata:* sì finché la policy di `FORWARD` è `ACCEPT`; con `iptables -P FORWARD DROP` la regola ACCEPT diventa necessaria.
3. **REDIRECT vs DNAT**: confronta `-j REDIRECT --to-port 80` (locale al router) con `DNAT` (verso un altro host).
4. **Cattura sui due lati**: Wireshark/tcpdump su `r0-eth0` mostra la destinazione `192.168.1.2:80`; sul lato client la destinazione è `10.0.0.1:8080`.
5. **SNAT**: sostituisci la regola `MASQUERADE` con interfaccia corretta e osserva l'indirizzo sorgente visto da `h1`.

## Troubleshooting

- `curl: (28) timed out` / `No route to host`: manca `10.0.0.1` su `r0` (problema noto).
- `Connection refused`: il server web su `h1` non è avviato.
- Nessuna regola applicata: script non eseguibile o lanciato da un'altra cartella.
- Reti residue: `sudo mn -c`.

## Pulizia

`exit` nel CLI e `sudo mn -c`.
