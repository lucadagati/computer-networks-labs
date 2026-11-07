# Analisi del Comportamento TCP con Wireshark

## Guida rapida (aggiunta): cosa contiene `tcp.pcap` e come analizzarlo

Difficoltà: intermedia | Tempo stimato: 60-90 minuti | Strumenti: Wireshark (GUI) oppure `tshark`

### Prerequisiti

- Wireshark 3.x o superiore (`sudo apt install wireshark`) oppure solo `tshark` (`sudo apt install tshark`).
- Concetti: handshake a tre vie, numeri di sequenza/ACK, finestra del ricevente (`Win`), MSS, RTT, slow start.
- Apertura: `wireshark Wireshark/TCP/tcp.pcap` oppure `tshark -r Wireshark/TCP/tcp.pcap`.

### Contenuto del file (verificato con `tshark` e `capinfos`)

| Proprietà | Valore |
|---|---|
| Pacchetti | 213 |
| Durata | 7,6 s (cattura del 21/08/2004, Ethernet) |
| Connessione principale | `192.168.1.102:1161` verso `128.119.245.12:80` (gaia.cs.umass.edu), 201 frame |
| Operazione | un `POST /ethereal-labs/lab3-1-reply.htm` che carica un file da circa 150 KB (164090 byte di dati dal client) e la risposta `HTTP/1.1 200 OK` (frame 203) |
| Altro traffico | un SYN isolato (frame 213, `192.168.1.102:1162` verso `199.2.53.206:631`, porta di stampa IPP) e 9 pacchetti SSDP (UDP 1900) da `192.168.1.100`: rumore da ignorare |
| Handshake | frame 1 `SYN` (Seq=0, MSS 1460), frame 2 `SYN,ACK` (Seq=0, Ack=1, Win=5840), frame 3 `ACK` |
| Primi segmenti dati del client | frame 4 (Seq=1, 565 byte, contiene le intestazioni HTTP del POST), poi Seq=566, 2026, 3486, 4946, 6406 con 1460 byte ciascuno (MSS) |
| Ritrasmissioni | nessuna (`tcp.analysis.retransmission` non seleziona alcun frame) |
| Finestra annunciata dal server | minimo 5840 byte (all'handshake) |
| Chiusura | nessun FIN/RST nella traccia: la cattura termina dopo la risposta HTTP |

I numeri coincidono con le soluzioni riportate più sotto.

### Filtri di visualizzazione utili (Wireshark e `tshark -Y`)

| Obiettivo | Filtro |
|---|---|
| Solo la connessione del lab | `tcp.port == 1161` oppure `tcp.stream == 0` |
| Handshake | `tcp.flags.syn == 1` |
| Solo dati dal client | `ip.src == 192.168.1.102 && tcp.len > 0` |
| Solo ACK dal server | `ip.src == 128.119.245.12 && tcp.len == 0` |
| Ritrasmissioni, ACK duplicati | `tcp.analysis.retransmission`, `tcp.analysis.duplicate_ack` |
| RTT di ciascun ACK | `tcp.analysis.ack_rtt` (colonna: `Statistica > Grafici`, o campo `tcp.analysis.ack_rtt`) |
| Messaggi HTTP | `http` (il lab invita a disabilitarlo per vedere i soli segmenti TCP) |

### Esempi con tshark

```bash
cd Wireshark/TCP
tshark -r tcp.pcap -c 12                          # primi 12 pacchetti
tshark -r tcp.pcap -q -z conv,tcp                 # conversazioni TCP
tshark -r tcp.pcap -Y 'tcp.flags.syn==1'          # handshake
tshark -r tcp.pcap -Y 'tcp.analysis.retransmission' | wc -l     # restituisce 0
tshark -r tcp.pcap -Y 'ip.src==192.168.1.102 && tcp.dstport==80 && tcp.len>0' \
       -T fields -e frame.number -e tcp.seq -e tcp.len | head -7
tshark -r tcp.pcap -Y 'tcp.analysis.ack_rtt' -T fields -e frame.number -e tcp.analysis.ack_rtt | head
tshark -r tcp.pcap -Y 'ip.src==128.119.245.12 && tcp.srcport==80' \
       -T fields -e tcp.window_size_value | sort -n | head -1       # 5840
```

Output verificato del primo comando (estratto): `1 ... [SYN] Seq=0 Win=16384 Len=0 MSS=1460`, `2 ... [SYN, ACK] Seq=0 Ack=1 Win=5840 ...`, `4 ... POST /ethereal-labs/lab3-1-reply.htm HTTP/1.1`.

### Procedura nella GUI

1. `File > Apri` e scegli `tcp.pcap`.
2. Applica il filtro `tcp.port == 1161` per isolare la connessione.
3. Seleziona il frame 1 e nel pannello dettagli espandi *Transmission Control Protocol*: osserva `Sequence Number`, `Flags`, `Window`, `Options`.
4. Per i grafici: scegli un pacchetto del client e apri `Statistiche > Grafici TCP > Time-Sequence (Stevens)`; poi anche `Throughput` e `Round Trip Time`.
5. Per vedere i numeri di sequenza relativi o assoluti: `Modifica > Preferenze > Protocolli > TCP > Relative sequence numbers`.

### Cosa osservare

- Il primo segmento (565 byte) è più corto degli altri perché contiene solo le intestazioni HTTP; poi il client invia a raffiche di due segmenti da 1460 byte, compatibile con una finestra iniziale piccola (slow start).
- Il server invia ACK cumulativi (`Ack` = 566, 2026, 3486, ... nei primi frame) e la finestra annunciata cresce (6780, 8760, 11680, ...). Verifica tu quanti segmenti conferma ciascun ACK (esercizio 3 della traccia originale).
- Il throughput stimato dalla soluzione (circa 30 KByte/s) si ottiene come (ultimo ACK - primo Seq)/(tempo ultimo ACK - tempo primo segmento).
- Il grafico di Stevens mostra una crescita a "gradini" ma, come indicato nelle soluzioni, la distinzione slow start / congestion avoidance non è netta in questa traccia.

### Esercizi aggiuntivi

1. Calcola l'RTT del primo segmento: filtro `tcp.analysis.ack_rtt`; verifica che per il frame 2 è circa 0,023 s (tempo SYN → SYN,ACK).
2. Con `tshark -z io,stat,1` conta frame e byte per secondo: *soluzione:* nel primo secondo 50 frame, nel secondo 36 (verificato).
3. Riconosci i 9 pacchetti SSDP con il filtro `udp`: *soluzione:* sono verso `192.168.1.1:1900`, non c'entrano con il lab.
4. Scrivi in un file CSV (`-T fields -E separator=,`) tempo e numero di sequenza e traccia il grafico con Python/Matplotlib.
5. Calcola la percentuale di segmenti da 1460 byte sul totale dei segmenti di dati.

### Troubleshooting

- Wireshark chiede permessi per catturare: non servono per aprire un `.pcap`; servono solo per catturare dal vivo (`sudo` o gruppo `wireshark`).
- `tshark` stampa "Running as user root ... dangerous": è un avviso, non un errore.
- Se non vedi i campi `tcp.analysis.*`: attiva `Analizza > Protocolli abilitati` TCP e l'opzione *Analyze TCP sequence numbers*.

### Pulizia

Il laboratorio non modifica nulla; non salvare file di esportazione dentro il repository.

---

## Traccia originale e soluzioni

## Descrizione del Laboratorio
In questo laboratorio, ci concentreremo sull'analisi approfondita del comportamento del protocollo TCP (Transmission Control Protocol). Attraverso l'esame di una traccia di segmenti TCP inviati e ricevuti durante il trasferimento di un file di 150KB (contenente "Le avventure di Alice nel Paese delle Meraviglie" di Lewis Carroll) dal tuo computer a un server remoto, esploreremo vari aspetti chiave del protocollo TCP.

### Obiettivi Principali
- **Numeri di Sequenza e Conferma:** Studieremo come TCP utilizza questi numeri per garantire un trasferimento dati affidabile.
- **Controllo della Congestione:** Analizzeremo l'algoritmo di controllo della congestione di TCP, osservando in particolare le fasi di slow start ed evitamento della congestione.
- **Controllo del Flusso:** Esamineremo il meccanismo di controllo del flusso pubblicizzato dal ricevitore di TCP.
- **Prestazioni di Connessione:** Valuteremo le prestazioni della connessione TCP, includendo throughput e tempo di andata e ritorno (RTT).

Il laboratorio include anche una breve considerazione sull'allestimento della connessione TCP.

### File pcap
Nel repository, troverai il file pcap con il flusso TCP, già pulito e pronto per l'analisi.

## Guida all'Analisi con Wireshark

### Analisi dei Segmenti TCP
1. **Indirizzo IP e Numero di Porta TCP:** Determina l'indirizzo IP e il numero di porta TCP utilizzati dal computer client durante il trasferimento del file a gaia.cs.umass.edu.
2. **Indirizzo IP di gaia.cs.umass.edu:** Identifica l'indirizzo IP del server e su quale numero di porta sta inviando e ricevendo segmenti TCP.

### Comandi Specifici
- Per cambiare la visualizzazione in Wireshark e concentrarti sui segmenti TCP anziché sui messaggi HTTP, vai su `Analizza -> Protocolli Abilitati`, deseleziona la casella HTTP e premi OK.

### Domande di Laboratorio
Rispondi alle seguenti domande basandoti sulla traccia tcp-ethereal-trace-1 disponibile in [Wireshark Labs](http://gaia.cs.umass.edu/wireshark-labs/wireshark-traces.zip):
- Quali sono i numeri di sequenza dei primi sei segmenti TCP nella connessione, includendo il segmento contenente il comando HTTP POST?
- Qual è il valore RTT per ciascuno dei sei segmenti?
- Analizza la lunghezza di ciascuno dei primi sei segmenti TCP.
- Qual è la quantità minima di spazio buffer disponibile annunciato al ricevente per l'intera traccia?
- Ci sono stati dei segmenti ritrasmessi?
- Qual è la velocità di trasferimento per la connessione TCP?

### Controllo della Congestione TCP in Azione
Utilizza lo strumento di graficazione Grafico Sequenza-Tempo (Stevens) in Wireshark per analizzare la quantità di dati inviati per unità di tempo dal client al server. Questo ti aiuterà a identificare le fasi di slow start di TCP e di evitamento della congestione.

## Risorse Aggiuntive
- [Introduzione a Wireshark](https://www.wireshark.org)
- [Documentazione TCP](https://www.ietf.org/rfc/rfc793.txt)

## Contributi e Feedback
I tuoi contributi e feedback su questo laboratorio sono sempre ben accetti. Sentiti libero di aprire una issue o una pull request per qualsiasi suggerimento o correzione.


# Soluzioni - Analisi TCP con Wireshark

## TCP Basics

### 1. Indirizzo IP e Porta TCP del Computer Client
- **IP address**: `192.168.1.102`
- **TCP port number**: `1161`

### 2. Indirizzo IP e Porta TCP di gaia.cs.umass.edu
- **IP address**: `128.119.245.12`
- **TCP port number**: `80`

### 3. Numero di Sequenza del Segmento TCP SYN
- **Sequence number**: `0`
- **SYN flag**: Set to `1` (identifies the segment as a SYN segment)

### 4. Numero di Sequenza del Segmento SYNACK
- **Sequence number**: `0`
- **ACK field value**: `1` (calculated by adding `1` to the initial sequence number of the client's SYN segment)
- **SYN and ACK flags**: Both set to `1` (identifies the segment as a SYNACK segment)

### 5. Numero di Sequenza del Segmento TCP con il Comando HTTP POST
- **Sequence number**: `1`

### 6. Numeri di Sequenza dei Primi Sei Segmenti e RTT
- **Segment numbers in trace**: 1 – 6 as No. 4, 5, 7, 8, 10, and 11.
- **Sequence numbers**: `1`, `566`, `2026`, `3486`, `4946`, `6406`.
- **Send times, ACK receive times, and RTT values** detailed for each segment.
- **EstimatedRTT values** calculated using the formula: `EstimatedRTT = 0.875 * EstimatedRTT + 0.125 * SampleRTT`.

### 7. Lunghezza dei Primi Sei Segmenti TCP
- **First TCP segment (HTTP POST)**: `565 bytes`.
- **Other five segments**: `1460 bytes` each (MSS).

### 8. Spazio Buffer Minimo Disponibile e Controllo del Mittente
- **Minimum buffer space (receiver window)**: `5840 bytes`.
- **No sender throttling** due to lack of receiver buffer space.

### 9. Segmenti Ritrasmessi nel File di Traccia
- **No retransmitted segments**, verified by sequence number analysis.

### 10. Dati Riconosciuti in un ACK
- **Acknowledged sequence numbers and acknowledged data** listed for each ACK.
- **Identification of cases** where the receiver is ACKing every other received segment.

### 11. Throughput della Connessione TCP
- **Throughput**: Approximately `30.222 KByte/sec`.

### 12. Fasi di Slow Start e Congestion Avoidance di TCP
- **Start of TCP Slow Start**: At the beginning of the connection.
- **Identification of TCP slow start and congestion avoidance phases** is not precisely possible in this trace.
