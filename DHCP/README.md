# DHCP: configurare un server su un'interfaccia virtuale

Difficoltà: intermedia | Tempo stimato: 45-60 minuti | **Non verificato in questo ambiente** (nessun server DHCP installato né avviato; installare e avviare un server DHCP su una macchina condivisa non è stato ritenuto sicuro. I comandi sono quelli standard di `isc-dhcp-server` e di `iproute2`)

> Questa pagina era `DHCP_virtualServer.md`; il contenuto originale è mantenuto e ampliato.

## Obiettivi didattici

- Installare e configurare un server DHCP (`isc-dhcp-server`) su Linux.
- Usare un'interfaccia virtuale per sperimentare senza toccare la rete reale.
- Leggere il file `dhcpd.conf` e capire i parametri: subnet, range, lease, gateway.
- Osservare il protocollo DORA (Discover, Offer, Request, Acknowledge) con `tcpdump`/Wireshark.

## Concetti teorici essenziali

- DHCP assegna dinamicamente IP, maschera, gateway, DNS a un client che non ha configurazione.
- Scambio DORA su UDP: il client (porta 68) invia `DISCOVER` in broadcast al server (porta 67); il server risponde con `OFFER`, il client con `REQUEST`, il server conferma con `ACK`.
- Il **lease** è il tempo per cui l'indirizzo è assegnato (`default-lease-time`, `max-lease-time`); il client rinnova a metà lease.
- Un server DHCP deve ascoltare su un'interfaccia che abbia un indirizzo nella subnet che serve.

## Prerequisiti

- Linux Debian/Ubuntu, privilegi di root (`sudo`).
- Un laboratorio isolato (VM): **non avviare un server DHCP su una rete aziendale o domestica reale**, potrebbe interferire con il DHCP esistente.
- Facoltativo: `tcpdump`/Wireshark, un client (`dhclient`, `udhcpc` o `dhcpcd`).

## Struttura

Il lab è una guida, senza file di codice. File di sistema coinvolti:

| File | Funzione |
|---|---|
| `/etc/dhcp/dhcpd.conf` | configurazione del server (subnet, range, opzioni) |
| `/etc/default/isc-dhcp-server` | interfacce su cui ascolta (`INTERFACESv4`) |
| `/var/lib/dhcp/dhcpd.leases` | database dei lease assegnati |
| `/var/log/syslog` | log del servizio |

## Procedura passo-passo

### 1. Installazione

```bash
sudo apt-get update
sudo apt-get install isc-dhcp-server
```

Alla fine dell'installazione il servizio può risultare in errore perché non è ancora configurato: è normale.

### 2. Interfaccia virtuale

```bash
sudo modprobe dummy
sudo ip link add dummy0 type dummy
sudo ip addr add 192.168.1.1/24 dev dummy0
sudo ip link set dummy0 up
ip addr show dummy0
```

Atteso: `dummy0` in stato `UP` con `inet 192.168.1.1/24`. Un'interfaccia `dummy` non ha un "cavo": nessun client può collegarsi (vedi la sezione "Con un client reale" per una variante).

### 3. Configurazione del server

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Contenuto:

```text
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.10 192.168.1.100;
    option routers 192.168.1.1;
}
```

Significato: lease di 10 minuti (massimo 2 ore); il server è *authoritative* per la subnet; assegna indirizzi da `.10` a `.100`; comunica `192.168.1.1` come gateway. Aggiungi, se vuoi, `option domain-name-servers 8.8.8.8;`.

### 4. Assegnazione dell'interfaccia

```bash
sudo nano /etc/default/isc-dhcp-server
```

Su Ubuntu/Debian recenti la variabile da impostare è `INTERFACESv4="dummy0"` (la vecchia `INTERFACES` non è più usata dalle versioni moderne; la guida originale la citava).

### 5. Controllo della sintassi e avvio

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf       # controlla la sintassi
sudo systemctl restart isc-dhcp-server        # oppure: sudo service isc-dhcp-server start
sudo systemctl status isc-dhcp-server
```

Atteso: `active (running)`. In caso di errore: `sudo journalctl -u isc-dhcp-server -e` o `/var/log/syslog`.

### Con un client reale (variante con coppia veth e network namespace, non verificata)

Per vedere davvero un client ottenere un indirizzo, sostituisci la `dummy0` con un'estremità di una coppia `veth`; l'altra estremità va in un namespace che fa da client:

```bash
sudo ip netns add cnl-client
sudo ip link add veth-srv type veth peer name veth-cli
sudo ip link set veth-cli netns cnl-client
sudo ip addr add 192.168.1.1/24 dev veth-srv && sudo ip link set veth-srv up
sudo ip netns exec cnl-client ip link set veth-cli up
# in /etc/default/isc-dhcp-server: INTERFACESv4="veth-srv", poi riavvia il servizio
sudo ip netns exec cnl-client dhclient -v veth-cli       # richiede un indirizzo
sudo ip netns exec cnl-client ip addr show veth-cli       # atteso: 192.168.1.x/24
cat /var/lib/dhcp/dhcpd.leases                            # lease registrato
```

Per osservare DORA: `sudo tcpdump -i veth-srv -n -vv 'udp port 67 or udp port 68'` (prima di lanciare `dhclient`). Atteso: quattro messaggi `DHCP-Message ... Discover/Offer/Request/ACK`. In Wireshark: filtro `bootp` (o `dhcp`).

## Cosa osservare

- Il primo indirizzo assegnato dovrebbe cadere nel range `192.168.1.10-.100`.
- `dhcpd.leases` riporta `starts`, `ends` e `hardware ethernet` del client.
- I messaggi usano UDP 67/68 e il `DISCOVER` è inviato a `255.255.255.255`.

## Esercizi proposti

1. **Riserva un indirizzo** a un MAC specifico con `host client1 { hardware ethernet aa:bb:cc:dd:ee:ff; fixed-address 192.168.1.50; }` dentro la subnet.
2. **Cambia il lease** a 60 s e osserva i rinnovi (`REQUEST` ripetuti) con tcpdump.
3. **Più subnet**: aggiungi una seconda interfaccia `veth` e una seconda `subnet`.
4. **Restringi il range** a 2 indirizzi, avvia tre client (tre namespace): cosa succede al terzo? *Soluzione:* non ottiene un lease (pool esaurito), il log riporta `no free leases`.
5. **Server DNS e opzioni**: aggiungi `option domain-name-servers ...;` e verifica `cat /etc/resolv.conf` o l'output di `dhclient`.

## Troubleshooting

- Servizio che non parte, con messaggio `Not configured to listen on any interfaces!`: manca la subnet corrispondente all'indirizzo dell'interfaccia, o `INTERFACESv4` è vuoto.
- `Can't create PID file` / permessi: controlla `dhcpd -t` e i log.
- Errore di sintassi: manca un `;` o una `}` in `dhcpd.conf`.
- Il client non riceve risposta: l'interfaccia è `dummy` (nessun client collegabile), oppure c'è un firewall che blocca UDP 67/68.
- Conflitti: un altro server DHCP (router domestico) risponde sulla stessa rete. Lavora solo su reti isolate.
- Log: `/var/log/syslog` o `journalctl -u isc-dhcp-server`.

## Pulizia

```bash
sudo systemctl stop isc-dhcp-server
sudo ip netns del cnl-client 2>/dev/null     # solo se hai usato la variante veth
sudo ip link del veth-srv 2>/dev/null
sudo ip link del dummy0 2>/dev/null
# opzionale: sudo apt-get remove isc-dhcp-server
```

Ripristina anche `/etc/dhcp/dhcpd.conf` e `/etc/default/isc-dhcp-server` se li avevi modificati su una macchina che usi per altro.
