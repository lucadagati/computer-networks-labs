# Laboratorio UDP con Wireshark

## Guida rapida (aggiunta): cosa contiene `udp.pcap` e come analizzarlo

Difficoltà: base | Tempo stimato: 30-45 minuti | Strumenti: Wireshark (GUI) oppure `tshark`

### Prerequisiti

- Wireshark o `tshark` (`sudo apt install tshark`).
- Concetti: porte, intestazione UDP (8 byte), DNS come esempio di protocollo su UDP.
- Apertura: `wireshark Wireshark/UDP/udp.pcap` oppure `tshark -r Wireshark/UDP/udp.pcap`.

### Contenuto del file (verificato con `tshark` e `capinfos`)

| Proprietà | Valore |
|---|---|
| Pacchetti | 37 |
| Durata | circa 40 s (cattura del 23/07/2007) |
| Host | `192.168.1.101` (client) |
| UDP | 16 pacchetti, tutti **DNS** (porta 53) verso/da `68.87.71.226`: ad es. query PTR `226.71.87.68.in-addr.arpa`, query A/NS per `www.mit.edu` e `mit.edu` con alcune risposte `No such name` e `Server failure`; risposta A per `www.mit.edu` = `18.7.22.83` |
| TCP | 21 pacchetti (non UDP): sessione TLS verso la porta 993 (IMAPS) di `128.119.245.93`, una connessione `3796 <-> 37024`, una chiusura HTTP verso `72.14.211.99:80`. Vanno ignorati con il filtro `udp` |
| Lunghezze del campo UDP Length | 33, 37, 48, 51, 52, 53, 57, 103, 133, 137, 142 |
| Numero di protocollo IP | 17 (0x11) |

Nota sulle soluzioni: l'esempio con lunghezza 58 (8 byte di intestazione + 50 di dati) riportato più sotto non corrisponde a un pacchetto di questo file (la lunghezza 58 non compare); nel pcap il primo pacchetto, ad esempio, ha Length = 51 (8 + 43 byte di dati). Il ragionamento della soluzione resta valido: Length = intestazione (8) + dati.

### Filtri utili

| Obiettivo | Filtro |
|---|---|
| Solo UDP | `udp` |
| Solo DNS | `dns` oppure `udp.port == 53` |
| Query o risposte | `dns.flags.response == 0` / `dns.flags.response == 1` |
| Errori DNS | `dns.flags.rcode != 0` |
| Pacchetti dell'host | `ip.addr == 192.168.1.101 && udp` |
| Solo il traffico non UDP | `!udp` |

### Esempi con tshark

```bash
cd Wireshark/UDP
tshark -r udp.pcap -Y udp                                    # 16 righe DNS
tshark -r udp.pcap -Y udp -T fields -e frame.number -e ip.src \
       -e udp.srcport -e udp.dstport -e udp.length -e udp.checksum.status | head -4
# 1  192.168.1.101  4372  53  51  2     <- porta 4372 -> 53, Length 51
# 2  68.87.71.226   53    4372 103 2    <- risposta: porte invertite
tshark -r udp.pcap -q -z io,phs                              # gerarchia: udp 16, tcp 21
tshark -r udp.pcap -Y dns -T fields -e dns.qry.name -e dns.flags.rcode
```

`udp.checksum.status` vale 2 ("non verificato"): tipico di catture con checksum offloading.

### Procedura nella GUI

1. Apri `udp.pcap` e applica `udp`.
2. Seleziona la prima query DNS (frame 1) ed espandi *User Datagram Protocol*: ci sono 4 campi da 2 byte (porta sorgente, porta destinazione, lunghezza, checksum).
3. Seleziona la risposta (frame 2): le porte sono scambiate (4372 diventa destinazione).
4. `Statistiche > Gerarchia dei protocolli` per vedere UDP/DNS contro TCP.

### Cosa osservare

- La porta sorgente è effimera (qui 4372, 4373, ...), la destinazione è la porta nota 53.
- Ogni richiesta ha una risposta con porte invertite; l'identificativo DNS (`0x0001`, `0x0002`, ...) lega query e risposta, perché UDP non ha connessioni.
- Nessun handshake e nessun ACK a livello di trasporto: l'affidabilità, se serve, è compito dell'applicazione.

### Esercizi aggiuntivi

1. Qual è l'intervallo tra la prima query e la sua risposta? *Soluzione:* circa 12 ms (0,012481 s).
2. Perché le query per `www.mit.edu.hsd1.ma.comcast.net` falliscono? *Soluzione:* il resolver prova i suffissi di ricerca; una risposta è `No such name` (rcode 3), un'altra `Server failure` (rcode 2); la query corretta `www.mit.edu` ottiene `18.7.22.83`.
3. Calcola la dimensione massima teorica del payload UDP su IPv4. *Soluzione:* 65535 - 20 - 8 = 65507 byte.
4. Scrivi un filtro per trovare i pacchetti con Length UDP maggiore di 100: `udp.length > 100`.
5. Genera un tuo pacchetto UDP in locale (`python3 Livello_Rete/TCP_1/serverUDP.py` e `clientUDP.py`) catturando su `lo` con `sudo tshark -i lo -f udp` e confronta le intestazioni (non verificato in questo ambiente per la cattura dal vivo).

### Troubleshooting

- Non vedi nulla con `udp`: stai guardando un file diverso, oppure il filtro è scritto male (la barra del filtro diventa rossa).
- Catturare dal vivo richiede privilegi: `sudo` o appartenenza al gruppo `wireshark`.

### Pulizia

Nessun file generato.

---

## Traccia originale e soluzioni

## Introduzione
In questo laboratorio, esploreremo il protocollo di trasporto UDP, un protocollo essenziale e senza fronzoli. L'UDP è semplice e diretto, il che ci permette di coprirlo rapidamente in questo laboratorio. Questo è ideale per chi ha limitazioni di tempo.

## Obiettivo
Diventare esperti nell'uso di Wireshark per analizzare il traffico UDP.

## Requisiti
- Conoscenza base di Wireshark.
- Connessione di rete attiva (o una traccia di pacchetti pre-registrata contenente pacchetti UDP).

## Attività
1. **Cattura di Pacchetti**
   - Avviare Wireshark e iniziare la cattura dei pacchetti.
   - Eseguire azioni che portino il tuo host a inviare e ricevere pacchetti UDP. Anche restando inattivi, è possibile catturare pacchetti UDP inviati da altri dispositivi sulla rete.

2. **Analisi dei Pacchetti UDP**
   - Interrompere la cattura dei pacchetti.
   - Impostare un filtro in Wireshark per mostrare solo i pacchetti UDP inviati e ricevuti dal tuo host.
   - Scegliere un pacchetto UDP e analizzarne i campi nell'intestazione UDP nella finestra dei dettagli.

## Esercizi
Dopo aver catturato e filtrato i pacchetti UDP, completare le seguenti attività:
1. **Analisi dell'Intestazione UDP**: Selezionare un pacchetto UDP e determinare il numero e il nome dei campi nell'intestazione UDP.
2. **Lunghezza dei Campi**: Determinare la lunghezza in byte di ciascun campo nell'intestazione UDP, utilizzando le informazioni visualizzate in Wireshark.
3. **Campo Lunghezza**: Capire a cosa si riferisce il valore nel campo Lunghezza e verificare con un pacchetto UDP catturato.
4. **Payload UDP Massimo**: Determinare il numero massimo di byte che possono essere inclusi in un payload UDP.
5. **Numero di Porta Sorgente Massimo**: Identificare il numero di porta sorgente più grande possibile.
6. **Numero di Protocollo per l'UDP**: Fornire il numero di protocollo per l'UDP in notazione esadecimale e decimale.
7. **Analisi di Coppie di Pacchetti UDP**: Esaminare una coppia di pacchetti UDP dove il primo è inviato dal tuo host e il secondo è una risposta al primo. Descrivere la relazione tra i numeri di porta nei due pacchetti.

## Suggerimenti per la Stampa dei Pacchetti
Per ogni domanda, è consigliato consegnare una stampa del/dei pacchetto/i usato/i per rispondere. Usare la funzione di stampa in Wireshark (File->Stampa) selezionando 'Solo pacchetto selezionato', 'Linea di riassunto del pacchetto', e i dettagli necessari per rispondere alle domande.


















































# Soluzioni - Analisi Pacchetti UDP con Wireshark

## 1. Campi dell'Intestazione UDP
- **Domanda**: Seleziona un pacchetto UDP dalla tua traccia e determina quanti e quali sono i campi nell'intestazione UDP.
- **Risposta**: L'intestazione UDP ha quattro campi: porta sorgente, porta destinazione, lunghezza e checksum.

## 2. Lunghezza dei Campi dell'Intestazione UDP
- **Domanda**: Determina la lunghezza (in byte) di ciascuno dei campi dell'intestazione UDP.
- **Risposta**: Ogni campo dell'intestazione UDP è lungo due byte.

## 3. Valore del Campo Lunghezza
- **Domanda**: Il valore nel campo Lunghezza si riferisce a cosa?
- **Risposta**: Il campo Lunghezza indica la lunghezza totale del segmento UDP, incluse intestazione e dati, misurata in byte. Nel pacchetto esaminato, la lunghezza è di 58 byte, con 8 byte di intestazione e 50 byte di dati.

## 4. Numero Massimo di Byte nel Payload UDP
- **Domanda**: Qual è il numero massimo di byte che possono essere inclusi in un payload UDP?
- **Risposta**: Dato che l'intestazione UDP usa 16 bit, la lunghezza massima di un segmento UDP (intestazione inclusa) è 65535 byte.

## 5. Numero di Porta Sorgente Massimo
- **Domanda**: Qual è il numero di porta sorgente più grande possibile?
- **Risposta**: Il numero massimo per una porta sorgente è 65535, derivante dai 16 bit utilizzati per rappresentare il numero di porta.

## 6. Numero di Protocollo per l'UDP
- **Domanda**: Qual è il numero di protocollo per l'UDP in notazione esadecimale e decimale?
- **Risposta**: Il numero di protocollo per l'UDP è 17, che in esadecimale si rappresenta con 0x11.

## 7. Relazione tra i Numeri di Porta in Coppie di Pacchetti UDP
- **Domanda**: Esamina una coppia di pacchetti UDP in cui il primo pacchetto è inviato dal tuo host e il secondo è una risposta al primo. Descrivi la relazione tra i numeri di porta nei due pacchetti.
- **Risposta**: La porta sorgente del primo pacchetto diventa la porta destinazione del secondo pacchetto e viceversa. Questo indica una comunicazione diretta tra i due host, dove il pacchetto di risposta è indirizzato esattamente all'host che ha inviato la richiesta.
