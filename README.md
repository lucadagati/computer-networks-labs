# Laboratori di Reti di Calcolatori

Raccolta di esercitazioni pratiche per il corso di **Reti di Calcolatori**, organizzate per livello della pila protocollare. Ogni cartella contiene codice eseguibile (Python, Docker, Mininet) e un README in italiano con obiettivi, teoria essenziale, prerequisiti, struttura dei file, procedura passo-passo con output atteso, esercizi, troubleshooting e pulizia.

## Indice delle aree

| Area | Cartella | Contenuto |
|---|---|---|
| Livello applicazione | [Livello_Applicazione/](Livello_Applicazione/) | demo containerizzate di web server, DNS (BIND), SMTP, Telnet vs SSH, FTP (vsftpd) e due varianti di rete **P2P** |
| Livello trasporto / rete | [Livello_Rete/](Livello_Rete/) | socket TCP e UDP in Python, multiplexing/demultiplexing, simulazione di TCP (controllo di congestione), calcolo dell'RTO, confronto TCP vs UDP nel trasferimento di file |
| Analisi del traffico | [Wireshark/](Wireshark/) | laboratori guidati su TCP e UDP con catture `.pcap` |
| Emulazione di reti | [mininet/](mininet/) | topologie Mininet, regole `iptables`, NAT e port forwarding |
| Configurazione servizi | [DHCP/](DHCP/) | guida a un server DHCP su interfaccia virtuale |
| Materiale | [docs/](docs/) | slide Mininet e traccia del laboratorio sul livello applicazione |

## Percorso didattico consigliato

1. **Socket e trasporto** (solo Python): [TCP_1](Livello_Rete/TCP_1/), [TCP_2](Livello_Rete/TCP_2/), [Multiplex_Demultiplex](Livello_Rete/Multiplex_Demultiplex/), [Multiplex_Demultiplex_2](Livello_Rete/Multiplex_Demultiplex_2/).
2. **Comportamento di TCP**: [TCP_4](Livello_Rete/TCP_4/) (chiusura), [TCP_5](Livello_Rete/TCP_5/) (RTO), [TCP_3](Livello_Rete/TCP_3/) (congestione), [TCP-lab](Livello_Rete/TCP-lab/), [TCP_6](Livello_Rete/TCP_6/) (TCP vs UDP).
3. **Osservare i pacchetti veri**: [Wireshark/UDP](Wireshark/UDP/) poi [Wireshark/TCP](Wireshark/TCP/).
4. **Protocolli applicativi con Docker**: [demo_webserver](Livello_Applicazione/demo_webserver/) -> [demo_DNS](Livello_Applicazione/demo_DNS/) -> [demo_smtp](Livello_Applicazione/demo_smtp/) -> [demo_telnet](Livello_Applicazione/demo_telnet/) -> [telnet_vs_ssh](Livello_Applicazione/telnet_vs_ssh/) -> [FTP](Livello_Applicazione/FTP/) -> [P2P](Livello_Applicazione/P2P/) -> [P2P_2](Livello_Applicazione/P2P_2/).
5. **Reti emulate e firewall/NAT**: [DHCP](DHCP/), [mininet/pdf_example_ch](mininet/pdf_example_ch/), [demo1](mininet/demo1/), [demo2](mininet/demo2/), [demo3](mininet/demo3/), [demo4](mininet/demo4/).

## Tabella dei laboratori

Difficoltà: B = base, I = intermedia, A = avanzata. I tempi sono stime per uno studente.

| Laboratorio | Concetto | Diff. | Tempo | Verifica |
|---|---|---|---|---|
| [Livello_Rete/TCP_1](Livello_Rete/TCP_1/) | socket TCP vs UDP, connessione vs datagramma | B | 30 min | eseguito |
| [Livello_Rete/TCP_2](Livello_Rete/TCP_2/) | server concorrente con thread, echo | B | 30 min | eseguito |
| [Livello_Rete/TCP_3](Livello_Rete/TCP_3/) | controllo di congestione Tahoe/Reno/NewReno | I | 45 min | calcolo eseguito, grafico no |
| [Livello_Rete/TCP_4](Livello_Rete/TCP_4/) | chiusura della connessione (FIN/ACK) | B | 30 min | eseguito |
| [Livello_Rete/TCP_5](Livello_Rete/TCP_5/) | retransmission timeout, backoff esponenziale | B | 20 min | eseguito |
| [Livello_Rete/TCP_6](Livello_Rete/TCP_6/) | TCP vs UDP: trasferimento di file, perdite | I | 40 min | eseguito |
| [Livello_Rete/TCP-lab](Livello_Rete/TCP-lab/) | finestra e ACK applicativi (stream senza confini) | I | 30 min | eseguito |
| [Livello_Rete/Multiplex_Demultiplex](Livello_Rete/Multiplex_Demultiplex/) | multiplexing/demultiplexing, porte | B | 30 min | eseguito |
| [Livello_Rete/Multiplex_Demultiplex_2](Livello_Rete/Multiplex_Demultiplex_2/) | header applicativo, framing con lunghezza | B-I | 40 min | eseguito |
| [Wireshark/UDP](Wireshark/UDP/) | intestazione UDP, DNS su UDP | B | 30-45 min | pcap analizzato con tshark |
| [Wireshark/TCP](Wireshark/TCP/) | sequenze, ACK, finestra, RTT, slow start | I | 60-90 min | pcap analizzato con tshark |
| [Livello_Applicazione/demo_webserver](Livello_Applicazione/demo_webserver/) | HTTP, Apache, port mapping | B | 20 min | eseguito |
| [Livello_Applicazione/demo_DNS](Livello_Applicazione/demo_DNS/) | zona DNS, record SOA/NS/A, `dig` | I | 45 min | eseguito |
| [Livello_Applicazione/demo_smtp](Livello_Applicazione/demo_smtp/) | dialogo SMTP, relay, consegna | I | 45 min | eseguito |
| [Livello_Applicazione/demo_telnet](Livello_Applicazione/demo_telnet/) | Telnet, traffico in chiaro | B | 30 min | eseguito |
| [Livello_Applicazione/telnet_vs_ssh](Livello_Applicazione/telnet_vs_ssh/) | confronto cifrato/in chiaro | I | 60 min | server SSH eseguito, client no |
| [Livello_Applicazione/FTP](Livello_Applicazione/FTP/) | FTP, canale controllo/dati, chroot | B-I | 45 min | eseguito |
| [Livello_Applicazione/P2P](Livello_Applicazione/P2P/) | rete P2P con Compose (scheletro) | B | 30 min | immagine eseguita |
| [Livello_Applicazione/P2P_2](Livello_Applicazione/P2P_2/) | scambio file tra peer su TCP | I | 45 min | eseguito con `docker run` |
| [DHCP](DHCP/) | DHCP, lease, DORA | I | 45-60 min | non verificato |
| [mininet/pdf_example_ch](mininet/pdf_example_ch/) | prima rete Mininet, ping, iperf | B | 30-45 min | non verificato |
| [mininet/demo1](mininet/demo1/) | topologie con `Topo`, `mn --custom` | B | 30 min | non verificato |
| [mininet/demo2](mininet/demo2/) | firewall `iptables` per host | I | 60 min | non verificato |
| [mininet/demo3](mininet/demo3/) | router, due subnet, NAT | I-A | 60 min | non verificato |
| [mininet/demo4](mininet/demo4/) | port forwarding (DNAT) | I-A | 60 min | regole verificate con namespace |

"Non verificato" significa che l'ambiente in cui è stata scritta la documentazione non aveva gli strumenti necessari (Mininet, server DHCP) e che i README lo dichiarano esplicitamente.

## Requisiti

- Linux (Ubuntu/Debian consigliato), Python 3.8+.
- Docker (e Docker Compose) per le demo del livello applicazione.
- Mininet e Open vSwitch per la sezione `mininet`; Wireshark o `tshark` per la sezione `Wireshark`.
- Matplotlib solo per `Livello_Rete/TCP_3`.

## Come usarle

```bash
git clone https://github.com/lucadagati/computer-networks-labs.git
cd computer-networks-labs/Livello_Rete/TCP_1     # esempio
python3 serverTCP.py &  python3 clientTCP.py
```

Le demo Docker hanno istruzioni specifiche nel proprio README (per esempio `docker compose up --build` in `Livello_Applicazione/P2P`).

Note: nel laboratorio `Livello_Rete/TCP_6` il file di prova da 10 MiB si genera con `python3 createfile.py` e non è versionato. Alcuni script (Mininet, demo Docker) richiedono correzioni minime o hanno comportamenti particolari: sono descritti nella sezione "Punti critici" o "Troubleshooting" del rispettivo README.

## Autori

Luca D'Agati, con contributi di Ilenia Ficili.
