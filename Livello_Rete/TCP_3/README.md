# Simulazione del controllo di congestione TCP (Tahoe, Reno, NewReno)

Difficoltà: intermedia | Tempo stimato: 45 minuti | Linguaggio: Python 3 + Matplotlib

## Obiettivi didattici

- Visualizzare l'evoluzione della finestra di congestione (`cwnd`) nelle fasi di **slow start** e **congestion avoidance**.
- Confrontare come Tahoe, Reno e NewReno reagiscono a una perdita.
- Capire i limiti di un modello didattico semplificato rispetto al TCP reale.

## Concetti teorici essenziali

- **Slow start**: `cwnd` raddoppia a ogni RTT finché non raggiunge `ssthresh`.
- **Congestion avoidance**: crescita lineare (circa +1 MSS per RTT).
- **Tahoe**: a ogni perdita `ssthresh = cwnd/2` e `cwnd = 1` (si riparte da slow start).
- **Reno**: con 3 ACK duplicati (fast retransmit/fast recovery) `cwnd` si dimezza e non si riparte da 1.
- **NewReno**: migliora la fast recovery in presenza di perdite multiple nella stessa finestra.

## Prerequisiti

- Python 3.8+ e Matplotlib (`pip install matplotlib`).
- Per vedere la finestra del grafico serve un ambiente grafico. Su server senza display usa il backend `Agg` e salva su file (vedi sotto).

## Struttura dei file

| File | Contenuto |
|---|---|
| `sim.py` | Funzione `simulate_tcp_congestion_control(algorithm)` e, in fondo, il codice che traccia le tre curve e chiama `plt.show()`. (Il file si chiama `sim.py`, non `tcp_congestion_control_simulation.py` come scritto nella vecchia versione del README.) |

Come funziona la funzione (100 iterazioni, "cicli"; un ciclo equivale a un RTT semplificato):

- Parte con `cwnd = 1`, `threshold = 10`.
- Se `cwnd < threshold`: `cwnd *= 2` (slow start). Altrimenti: Tahoe fa `cwnd += 1`; Reno e NewReno fanno `cwnd += 1/cwnd`.
- Ogni 20 cicli (`i % 20 == 0`) simula una perdita: Tahoe `cwnd = 1`; Reno `cwnd /= 2`; NewReno `cwnd = cwnd/2 + 3`. Poi `threshold = cwnd` e riparte.

## Procedura passo-passo

1. Installa le dipendenze e lancia lo script:

   ```bash
   cd Livello_Rete/TCP_3
   pip install matplotlib
   python3 sim.py
   ```

2. Si apre una finestra con tre curve (Tahoe, Reno, NewReno) su 100 cicli. (Verificato che lo script esegue con backend `Agg`; l'apertura della finestra interattiva non è stata verificata in questo ambiente senza display.)

3. Senza display, per ottenere il PNG:

   ```bash
   MPLBACKEND=Agg python3 -c "exec(open('sim.py').read().replace('plt.show()', 'plt.savefig(\"cwnd.png\")'))"
   ```

4. Valori che dovresti ottenere (calcolati eseguendo la funzione):

   | Algoritmo | primi valori di `cwnd` | ciclo 18-24 | massimo |
   |---|---|---|---|
   | Tahoe | 1, 2, 4, 8, 16, 17 | 30, 31, **1**, 2, 3, 4, 5 | 31 |
   | Reno | 1, 2, 4, 8, 16, 16.06 | 16.85, 16.91, **8.49**, 8.6, 8.72, 8.83, 8.95 | 16.91 |
   | NewReno | 1, 2, 4, 8, 16, 16.06 | 16.85, 16.91, **11.49**, 11.57, 11.66, 11.75, 11.83 | 16.91 |

## Cosa osservare

- Tutti partono con crescita esponenziale (1, 2, 4, 8, 16) e superano la soglia 10.
- Tahoe cresce di 1 a ciclo (più veloce di Reno/NewReno che crescono di `1/cwnd` per ciclo) e al ciclo 20 crolla a 1.
- Reno si dimezza; NewReno si dimezza e aggiunge 3, quindi ricomincia più in alto.

## Limiti del modello (da discutere)

- Dopo la perdita il codice pone `threshold = cwnd` **già ridotta**; per Tahoe quindi `threshold = 1` e non riparte con slow start (cresce linearmente da 1, mentre il TCP reale userebbe `ssthresh = cwnd_prima_della_perdita / 2`).
- Tahoe usa `+1` per ciclo, Reno e NewReno `+1/cwnd` per ciclo: unità di misura diverse, perciò il confronto tra le curve non è omogeneo (in TCP reale `+1/cwnd` è per ACK, cioè circa +1 MSS per RTT per tutti).
- Le perdite sono periodiche e deterministiche, senza ACK duplicati né timeout.

## Esercizi proposti

1. **Cambia la soglia.** Metti `threshold = 32`: quando termina lo slow start? *Soluzione:* `cwnd` arriva a 32 dopo 5 cicli (1, 2, 4, 8, 16, 32).
2. **Cambia il periodo delle perdite.** Usa `i % 10 == 0` e `i % 50 == 0`. Quale algoritmo risente di più delle perdite frequenti? *Soluzione:* Tahoe, che azzera la finestra.
3. **Correggi il modello.** Memorizza `threshold = cwnd_prima / 2` e fai crescere tutti in modo omogeneo (`cwnd += 1/cwnd`). Come cambiano le curve?
4. **Aggiungi un grafico di `ssthresh`.** Restituisci anche la storia della soglia e disegnala.
5. **Confronta col pcap.** Apri `../../Wireshark/TCP/tcp.pcap` e usa il grafico Stevens: i dati inviati per RTT assomigliano alla fase di slow start?

## Troubleshooting

- `ModuleNotFoundError: No module named 'matplotlib'`: `pip install matplotlib`.
- Nessuna finestra / errore `Cannot connect to display`: usa `MPLBACKEND=Agg` e salva il PNG.
- Su WSL serve un server X o WSLg.

## Pulizia

Rimuovi eventuale `cwnd.png` generato e la cartella `__pycache__` se presente.
