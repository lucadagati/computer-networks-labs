# Mininet demo3: router Linux con due subnet e NAT

Difficoltà: intermedia-avanzata | Tempo stimato: 60 minuti | **Non verificato in questo ambiente** (Mininet non installato; le osservazioni derivano dalla lettura degli script e non sono state riprodotte nemmeno con namespace equivalenti)

## Obiettivi didattici

- Costruire una rete con due subnet collegate da un router Linux (nodo `r0` con due interfacce).
- Abilitare l'IP forwarding e configurare i gateway predefiniti degli host.
- Capire NAT/MASQUERADE e perché le regole `iptables` dipendono dai nomi delle interfacce.

## Concetti teorici essenziali

- Un router inoltra pacchetti tra subnet diverse (`net.ipv4.ip_forward=1`); gli host usano il router come *default gateway*.
- **NAT con MASQUERADE** (`-t nat -A POSTROUTING -o <if_uscita> -j MASQUERADE`): riscrive l'IP sorgente con quello dell'interfaccia di uscita.
- La catena `FORWARD` filtra i pacchetti in transito; una regola `-m state --state RELATED,ESTABLISHED` consente le risposte a connessioni già avviate.

## Prerequisiti

- Mininet, OVS, controller, `sudo` (vedi [`../README.md`](../README.md)).

## Struttura dei file

| File | Contenuto |
|---|---|
| `mininet_nat_demo.py` | Crea `s1`, `s2`; `h1 = 192.168.1.2/24` e `h2 = 192.168.1.3/24` su `s1`; `h3 = 10.0.0.2/24` ("server esterno") su `s2`; il router `r0` (classe `Node`) con due link: `r0-eth0` verso `s1` e `r0-eth1` verso `s2`. Dopo l'avvio assegna `192.168.1.1/24` a `r0-eth0` e `10.0.0.1/24` a `r0-eth1`, esegue `./router_iptables.sh` su `r0`, imposta i default gateway (h1, h2 verso `192.168.1.1`; h3 verso `10.0.0.1`) e apre il CLI. |
| `router_iptables.sh` | Abilita il forwarding, svuota `filter` e `nat`, aggiunge `MASQUERADE` su `-o eth0` e due regole `FORWARD` (`-i eth1 -o eth0` ACCEPT; `-i eth0 -o eth1` ACCEPT solo `RELATED,ESTABLISHED`). |

Topologia:

```text
h1 (192.168.1.2) \                                   
                  s1 --(r0-eth0 192.168.1.1) r0 (r0-eth1 10.0.0.1)-- s2 -- h3 (10.0.0.2)
h2 (192.168.1.3) /
```

## Punti critici da conoscere (dedotti dal codice)

1. **Nomi delle interfacce.** In Mininet le interfacce del router si chiamano `r0-eth0` e `r0-eth1`. Le regole di `router_iptables.sh` usano `eth0`/`eth1`, che **non esistono** nel namespace di `r0`: le regole `MASQUERADE` e `FORWARD` quindi non corrispondono a nessun pacchetto (stesso errore verificato in `demo4`, dove la prova con namespace equivalenti ha confermato che l'inoltro funziona comunque grazie alla policy `FORWARD ACCEPT`). Conseguenza attesa: il traffico viene instradato **senza NAT** e anche `h3 -> h1/h2` può funzionare, al contrario di quanto suggerisce la vecchia versione del README. Controlla con i contatori (`r0 iptables -t nat -L -v -n`): se `pkts` resta 0 la regola non scatta.
2. **Permessi e cartella.** `./router_iptables.sh` è un percorso relativo e il file non è eseguibile nel repository: `chmod +x router_iptables.sh` ed esecuzione da `mininet/demo3`.
3. Per un NAT funzionante sostituisci `eth0`/`eth1` con `r0-eth1`/`r0-eth0` nelle regole (cioè `-o r0-eth1` per l'uscita verso la rete "esterna" 10.0.0.0/24).

## Procedura passo-passo

```bash
cd mininet/demo3
chmod +x router_iptables.sh
sudo python3 mininet_nat_demo.py
```

Nel CLI:

1. Indirizzi: `h1 ip addr`, `h2 ip addr`, `h3 ip addr`, `r0 ip addr` (o `ifconfig`).
2. Routing: `h1 route -n`, `h3 route -n`, `r0 route -n`. Atteso: h1/h2 con default via `192.168.1.1`, h3 con default via `10.0.0.1`, r0 con le due reti direttamente connesse.
3. Connettività interna verso l'esterno: `h1 ping -c 3 10.0.0.2` e `h2 ping -c 3 10.0.0.2`: attesi con successo (grazie al routing).
4. Direzione opposta: `h3 ping -c 3 192.168.1.2`: vedi punto critico 1 (con le regole originali probabilmente funziona; con un NAT corretto e `FORWARD` con policy `DROP` fallirebbe).
5. Regole: `r0 iptables -t nat -L -v -n` e `r0 iptables -L -v -n`.
6. Per vedere il NAT in azione dopo la correzione, avvia in h3 `h3 tcpdump -n -i h3-eth0 icmp &` e fai `h1 ping -c 1 10.0.0.2`: con MASQUERADE funzionante l'indirizzo sorgente atteso visto da h3 è `10.0.0.1` (router), senza NAT è `192.168.1.2`.
7. `pingall` per il test complessivo.

## Cosa osservare

- L'indirizzo sorgente visto dall'host esterno (con o senza NAT).
- I contatori delle regole `iptables`.
- La tabella di routing dei tre tipi di nodo.

## Esercizi proposti

1. **Correggi i nomi delle interfacce** (`r0-eth0`, `r0-eth1`) e ripeti il passo 6: la sorgente vista da h3 diventa `10.0.0.1`.
2. **Policy restrittiva**: `r0 iptables -P FORWARD DROP` dopo aver corretto le regole. Verifica che h1 raggiunga h3 ma non il contrario.
3. **Server web interno**: `h1 python3 -m http.server 80 &`, poi da h3 `curl 192.168.1.2`: fallisce con policy restrittiva, riesce con il port forwarding della demo4.
4. **Conntrack**: `r0 conntrack -L` (pacchetto `conntrack`) mostra le traduzioni NAT.
5. **Aggiungi un terzo host interno** `h4` e verifica che usi lo stesso gateway.

## Troubleshooting

- `ping` fallisce ovunque: controlla `r0 cat /proc/sys/net/ipv4/ip_forward` (deve essere 1) e i default gateway.
- Regole non applicate: vedi punti critici 1 e 2; lo script non segnala errori perché `cmd` scarta l'output.
- Reti residue: `sudo mn -c`.

## Pulizia

`exit`, poi `sudo mn -c`.
