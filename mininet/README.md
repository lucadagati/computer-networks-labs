# Mininet: emulazione di reti con host, switch e router Linux

Questa cartella contiene cinque esercitazioni di complessità crescente. Le slide introduttive sono in [`../docs/mininet-slides.pdf`](../docs/mininet-slides.pdf).

**Stato della verifica.** Mininet e Open vSwitch **non erano installati** nell'ambiente in cui è stata scritta questa documentazione, quindi gli script Mininet **non sono stati eseguiti**. Di ogni script è stata verificata la sintassi (`python3 -I -m py_compile`) e il comportamento è descritto leggendo il codice. Per `demo4` la logica `iptables` è stata inoltre verificata con network namespace Linux equivalenti (non con Mininet): dove i risultati si basano su quella prova è indicato nel README del lab. Tutto il resto è marcato "non verificato in questo ambiente".

## Percorso consigliato

| Cartella | Argomento | Difficoltà | Tempo |
|---|---|---|---|
| [`pdf_example_ch/`](pdf_example_ch/) | prima rete: 2 host + 1 switch, ping, iperf, Wireshark | base | 30-45 min |
| [`demo1/`](demo1/) | topologie con classe `Topo`, `mn --custom`, salvataggio con `pickle` | base | 30 min |
| [`demo2/`](demo2/) | firewall `iptables` per host (filtri, REJECT, LOG, limit) | intermedia | 60 min |
| [`demo3/`](demo3/) | router Linux con due subnet e NAT (MASQUERADE) | intermedia-avanzata | 60 min |
| [`demo4/`](demo4/) | port forwarding (DNAT) 8080 verso 80 | intermedia-avanzata | 60 min |

## Concetti teorici essenziali

- Mininet crea host come **processi in network namespace separati** (ciascuno con proprie interfacce, tabella di routing e regole `iptables`), collegati da coppie `veth` a switch Open vSwitch (OVS).
- Nel CLI di Mininet `h1 comando` esegue `comando` dentro l'host `h1`; i nomi degli host usati come argomenti (`h1 ping h2`) vengono sostituiti dagli indirizzi IP.
- Indirizzi di default: `h1 = 10.0.0.1/8`, `h2 = 10.0.0.2/8`, ... in ordine di creazione.
- Le interfacce si chiamano `<nodo>-eth<N>` (ad es. `h1-eth0`, `r0-eth1`): **non** `eth0`/`eth1`. Questo è importante per le regole `iptables` con `-i`/`-o`.
- I file e il filesystem sono **condivisi** con la macchina ospite (lo script `./host1_iptables.sh` si trova nella cartella corrente).

## Prerequisiti comuni

- Linux con privilegi di root (`sudo`), Mininet 2.3+ e Open vSwitch: `sudo apt install mininet openvswitch-switch` (oppure installazione da sorgente come nelle slide).
- Un controller: gli script usano `Controller` (il controller di riferimento, pacchetto `openvswitch-testcontroller` su Debian/Ubuntu se `controller` non è presente).
- Facoltativi: `iperf`, `xterm` (serve un server X), Wireshark, `curl`.
- Verifica dell'installazione: `sudo mn --test pingall`.

## Comandi base del CLI

| Comando | Effetto |
|---|---|
| `nodes`, `net`, `links` | elenca nodi, collegamenti |
| `pingall` | ping tra tutte le coppie di host |
| `h1 ifconfig`, `h1 ip addr`, `h1 route -n` | stato di un host |
| `xterm h1` | terminale sull'host (richiede X) |
| `h1 iptables -L -v -n` | regole firewall dell'host |
| `exit` | esce dal CLI e ferma la rete |

Pulizia dopo un'esecuzione interrotta (dentro la macchina di laboratorio): `sudo mn -c`.

## Cheat sheet iptables

Regole di esempio (da eseguire dentro un host o router Mininet, ad es. `r0 iptables ...`):

```bash
iptables -A INPUT -p tcp --dport 80 -j DROP                         # blocca TCP 80 in ingresso
iptables -A INPUT -p tcp --dport 22 -j ACCEPT                       # consente SSH
iptables -A INPUT -s 192.168.1.10 -j DROP                           # blocca un indirizzo sorgente
iptables -A INPUT -p tcp --dport 80 -m limit --limit 10/min -j ACCEPT   # limita (poi DROP)
iptables -A INPUT -p tcp --dport 80 -j DROP
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80   # redirect locale
iptables -A INPUT -j LOG --log-prefix "INPUT: "                     # log
iptables -A INPUT -p tcp -m multiport ! --dports 22,80 -j DROP      # tutto tranne 22 e 80
iptables -L -v -n ; iptables -t nat -L -v -n                        # visualizza con contatori
iptables -F ; iptables -t nat -F                                    # svuota le regole
```

Ricorda che `iptables -F` svuota le regole ma **non** cambia le *policy* di default delle catene.

## Troubleshooting generale

- `Exception: Error creating interface pair` o interfacce residue: `sudo mn -c` e riprova.
- `Please shut down the controller which is running on port 6653`: un altro controller occupa la porta; `sudo mn -c` oppure termina il processo.
- `RTNETLINK answers: File exists` dopo run interrotti: `sudo mn -c`.
- `xterm` non si apre: serve `DISPLAY` (con `sudo` usa `sudo -E` o `xhost +local:`).
- Su WSL1 Mininet non funziona (mancano i namespace di rete); usa una VM o WSL2.

## Pulizia

```bash
sudo mn -c
```
