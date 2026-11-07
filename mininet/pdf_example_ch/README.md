# Mininet: primo esempio con due host e uno switch

Difficoltà: base | Tempo stimato: 30-45 minuti | **Non verificato in questo ambiente** (Mininet non installato; `script.py` ha solo il controllo sintattico `python3 -I -m py_compile`)

## Obiettivi didattici

- Costruire una rete Mininet da codice Python (API a basso livello: `addHost`, `addSwitch`, `addLink`).
- Usare il CLI per verificare la connettività, ispezionare le interfacce e misurare le prestazioni.
- Catturare e leggere il traffico ICMP con Wireshark/tcpdump.

## Concetti teorici essenziali

- Switch L2 (OVS con controller di riferimento): impara gli indirizzi MAC e inoltra i frame.
- ICMP Echo Request / Echo Reply (`ping`), RTT.
- `iperf` misura il throughput TCP tra due endpoint.

## Prerequisiti

- Mininet + Open vSwitch + controller (vedi [`../README.md`](../README.md)), `sudo`, `iperf`.
- Per la finestra `xterm` degli host serve un ambiente grafico (X11): senza di esso togli le righe `xterm` dallo script.

## Struttura dei file

| File | Contenuto |
|---|---|
| `script.py` | Funzione `firstNetwork()`: crea `Mininet()` vuota, aggiunge il controller `c0`, gli host `h1` (IP `10.0.0.1`) e `h2` (IP automatico, `10.0.0.2`), lo switch `s12` e due link; avvia la rete; apre un `xterm` per ciascun host (`h1.cmd("xterm ... &")`); apre il CLI; alla chiusura esegue `killall xterm` e `net.stop()`. |

Topologia:

```text
h1 (10.0.0.1) ---- s12 ---- h2 (10.0.0.2)
                    |
                   c0 (controller)
```

Nota: i comandi `xterm` nello script non hanno uno spazio prima di `-T` (`...false'-T h1 &`); dipende dalla shell se funzionano. Se gli `xterm` non si aprono non è un problema per gli esercizi: usa `h1 ...` nel CLI.

## Procedura passo-passo

1. ```bash
   cd mininet/pdf_example_ch
   sudo python3 script.py
   ```
2. Output atteso (da verificare): messaggi `*** Adding controller`, `*** Adding hosts`, `*** Adding switch`, `*** Creating links`, `*** Starting network`, ... e il prompt `mininet>`.
3. Esegui gli esercizi qui sotto nel CLI.
4. Esci con `exit`: lo script chiude gli xterm e ferma la rete.

## Esercizi proposti

### Esercizio 1: connettività

```text
mininet> h1 ping -c 3 h2
mininet> pingall
```

Atteso: 0% packet loss; `pingall` riporta `0% dropped (2/2 received)`.

### Esercizio 2: analisi del traffico con Wireshark

Avvia Wireshark sull'host (fuori da Mininet) sull'interfaccia dello switch, ad esempio `s12-eth1` (`sudo wireshark -i s12-eth1 -k` oppure `sudo tcpdump -i s12-eth1 -n icmp`), poi esegui `h1 ping -c 3 h2`. Atteso: coppie ICMP Echo request / Echo reply tra `10.0.0.1` e `10.0.0.2`, con una richiesta ARP iniziale.

### Esercizio 3: configurazione delle interfacce

```text
mininet> h1 ifconfig
mininet> h2 ifconfig
mininet> h1 ifconfig h1-eth0 10.0.0.3 netmask 255.255.255.0
mininet> h1 ping -c 3 h2
```

Atteso: dopo il cambio, `h1` è `10.0.0.3/24` e `h2` è `10.0.0.2/8`: stanno ancora sulla stessa rete logica (10.0.0.0), quindi il ping dovrebbe continuare a funzionare.

### Esercizio 4: prestazioni

```text
mininet> h1 iperf -s &
mininet> h2 iperf -c 10.0.0.1
```

Se hai cambiato l'IP di `h1` nell'esercizio 3 usa `10.0.0.3` oppure ripristinalo (`h1 ifconfig h1-eth0 10.0.0.1 netmask 255.0.0.0`). In alternativa, il comando integrato `iperf h1 h2` esegue server e client. Per la latenza: `h1 ping -c 3 -s 1000 h2` (dimensione pacchetto variabile). Se `iperf` non c'è: `sudo apt install iperf`.

## Esercizi aggiuntivi

1. Aggiungi un terzo host `h3` e ripeti `pingall`: quante coppie vengono testate? (6, ovvero 3x2).
2. Usa `net.addLink(h1, s12, bw=10, delay='50ms')` con `TCLink` (`from mininet.link import TCLink`, `Mininet(link=TCLink)`) e osserva RTT e throughput.
3. Spegni un link: `link h1 s12 down` e ripeti il ping; poi `link h1 s12 up`.

## Troubleshooting

Vedi [`../README.md`](../README.md): `sudo mn -c`, controller mancante, DISPLAY per `xterm`.

## Pulizia

`exit` nel CLI, poi `sudo mn -c`.
