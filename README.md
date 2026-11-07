# Laboratori di Reti di Calcolatori

Raccolta di esercitazioni pratiche per il corso di **Reti di Calcolatori**, organizzate per livello della pila protocollare. Ogni cartella contiene codice eseguibile (Python, Docker, Mininet) e un README con obiettivi e istruzioni passo-passo.

## Indice

| Area | Cartella | Contenuto |
|---|---|---|
| Livello applicazione | [Livello_Applicazione/](Livello_Applicazione/) | demo containerizzate di web server, DNS (BIND), SMTP, Telnet vs SSH, FTP (vsftpd) e due varianti di rete **P2P** |
| Livello trasporto / rete | [Livello_Rete/](Livello_Rete/) | socket TCP e UDP in Python, multiplexing/demultiplexing, simulazione di TCP (controllo di congestione), calcolo dell'RTO, confronto TCP vs UDP nel trasferimento di file |
| Analisi del traffico | [Wireshark/](Wireshark/) | laboratori guidati su TCP e UDP con catture `.pcap` |
| Emulazione di reti | [mininet/](mininet/) | topologie Mininet, regole `iptables`, NAT e port forwarding |
| Configurazione servizi | [DHCP/](DHCP/) | guida a un server DHCP su interfaccia virtuale |
| Materiale | [docs/](docs/) | slide Mininet e traccia del laboratorio sul livello applicazione |

## Requisiti

- Linux (Ubuntu/Debian consigliato), Python 3.8+
- Docker e Docker Compose per le demo del livello applicazione
- Mininet e Wireshark per le sezioni omonime

## Come usarle

```bash
git clone https://github.com/lucadagati/computer-networks-labs.git
cd computer-networks-labs/Livello_Rete/TCP_1     # esempio
python3 serverTCP.py &  python3 clientTCP.py
```

Le demo Docker si avviano con `docker compose up --build` dalla cartella della demo (es. `Livello_Applicazione/P2P`).

Nota: nel laboratorio `Livello_Rete/TCP_6` il file di prova da 10 MB viene generato con `python3 createfile.py` e non è versionato.

## Autori

Luca D'Agati, con contributi di Ilenia Ficili.
