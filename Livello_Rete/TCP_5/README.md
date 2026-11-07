# Simulazione del Retransmission Timeout (RTO) con backoff esponenziale

Difficoltà: base | Tempo stimato: 20 minuti | Linguaggio: Python 3 (libreria standard)

## Obiettivi didattici

- Capire perché TCP ha un timer di ritrasmissione e perché lo raddoppia dopo ogni perdita (*exponential backoff*).
- Osservare un modello semplificato di adattamento del timeout alle condizioni di rete.
- Confrontare il modello didattico con l'algoritmo reale di stima dell'RTO (RFC 6298).

## Concetti teorici essenziali

- Il mittente avvia un timer per ogni segmento non confermato; allo scadere (RTO) ritrasmette.
- Dopo una scadenza TCP **raddoppia** l'RTO (backoff esponenziale) per non peggiorare una congestione; ha un limite massimo.
- Nel TCP reale l'RTO si calcola da `SRTT` e `RTTVAR` (`RTO = SRTT + 4·RTTVAR`, minimo 1 s), non con un semplice dimezzamento.

## Prerequisiti

- Python 3.8+. Nessuna libreria esterna e nessuna rete.

## Struttura dei file

| File | Contenuto |
|---|---|
| `RTO.py` | La funzione `simulate_tcp_rto()` (chiamata alla fine del file). Il file si chiama `RTO.py` (il vecchio README citava `simulate_tcp_rto.py`). |

Logica del ciclo (10 tentativi):

1. Stampa `Tentativo N: RTO corrente = Xs`.
2. Con probabilità 30% (`random.random() < 0.3`) il pacchetto è "perso", altrimenti "ricevuto".
3. Se perso: `rto = min(rto * 2, 10)`. Se ricevuto: `rto = max(1, rto / 2)`.
4. `time.sleep(rto)`: attende il nuovo RTO prima del tentativo successivo.

Valori iniziali: `rto = 1` s, massimo 10 s. Non c'è un seme casuale: ogni esecuzione è diversa.

## Procedura passo-passo

```bash
cd Livello_Rete/TCP_5
python3 RTO.py
```

Durata: da 10 s (nessuna perdita) fino a oltre 50 s (molte perdite consecutive).

Output di una esecuzione reale (la tua sarà diversa; durata 20 s):

```text
Tentativo 1: RTO corrente = 1s
  Pacchetto perso. Attendere per RTO prima della ritrasmissione...
Tentativo 2: RTO corrente = 2s
  Pacchetto ricevuto correttamente.
Tentativo 3: RTO corrente = 1s
  Pacchetto ricevuto correttamente.
...
Tentativo 6: RTO corrente = 1s
  Pacchetto perso. Attendere per RTO prima della ritrasmissione...
Tentativo 7: RTO corrente = 2s
  Pacchetto perso. Attendere per RTO prima della ritrasmissione...
Tentativo 8: RTO corrente = 4s
  Pacchetto ricevuto correttamente.
Tentativo 9: RTO corrente = 2.0s
```

Nota: il valore stampato è quello **prima** dell'aggiornamento di quel tentativo; dopo un ricevimento corretto il valore diventa un `float` (`2.0s`).

## Cosa osservare

- Dopo ogni perdita il tentativo successivo ha RTO doppio: 1, 2, 4, 8, 10, 10, ...
- Dopo un successo l'RTO si dimezza ma non scende sotto 1 s.
- Il programma non ritrasmette davvero: dopo una perdita passa semplicemente al tentativo successivo (ritrasmissione non modellata).

## Esercizi proposti

1. **Seed fisso.** Aggiungi `random.seed(42)` all'inizio per rendere riproducibile la simulazione e confrontare due versioni del codice.
2. **Probabilità di perdita.** Porta la perdita a 0.7 (e a 0.0). Come si comporta l'RTO? *Soluzione:* con 0.7 satura rapidamente al massimo (10 s); con 0.0 resta sempre 1 s.
3. **Tempo totale.** Somma gli `rto` attesi e stampa il tempo totale della simulazione. Confronta perdita 0.1 e 0.5.
4. **RTO reale.** Implementa la stima RFC 6298: `SRTT = 0.875·SRTT + 0.125·R`, `RTTVAR = 0.75·RTTVAR + 0.25·|SRTT-R|`, `RTO = SRTT + 4·RTTVAR`, con RTT campione `R` casuali tra 20 e 80 ms.
5. **Collegamento al pcap.** Nel file `../../Wireshark/TCP/tcp.pcap` si rilevano ritrasmissioni (`tcp.analysis.retransmission`)? *Soluzione:* nessuna (verificato con tshark).

## Troubleshooting

- Sembra "bloccato": è la `time.sleep`; una esecuzione può durare oltre mezzo minuto.
- Per interrompere: `Ctrl+C`.

## Pulizia

Nessun file creato (rimuovi `__pycache__` se presente).
