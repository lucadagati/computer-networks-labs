# Mininet demo1: definire una topologia con `Topo`

Difficoltà: base | Tempo stimato: 30 minuti | **Non verificato in questo ambiente** (Mininet non installato; solo `python3 -I -m py_compile` su `topo2.py` e `topo3.py`)

## Obiettivi didattici

- Descrivere una topologia con la classe `Topo` (host, switch, link).
- Usarla in due modi: da un programma Python (`Mininet(topo=...)`) e dalla riga di comando (`mn --custom ... --topo ...`).
- Capire cosa fa il salvataggio della topologia con `pickle`.

## Concetti teorici essenziali

- Una `Topo` è una descrizione astratta del grafo di rete; `Mininet(topo=...)` la istanzia come processi e link reali.
- La topologia è quella classica di due LAN collegate: due switch, ciascuno con due host, e un link tra gli switch.

```text
 h1      h3
  \      /
   s1---s2
  /      \
 h2      h4
```

## Prerequisiti

- Mininet + Open vSwitch + controller, `sudo` (vedi [`../README.md`](../README.md)).

## Struttura dei file

| File | Contenuto |
|---|---|
| `topo2.py` | Classe `MyTopology(Topo)` con `h1..h4`, `s1`, `s2` e i cinque link (h1-s1, h2-s1, h3-s2, h4-s2, s1-s2). Nel `__main__` crea `Mininet(topo=mytopo)`, avvia la rete, **salva la topologia in `topo2.pkl`** con `pickle.dump`, esegue `h1.cmd('ping -c 3', h2.IP())` e `h3.cmd('ping -c 3', h4.IP())` (l'output dei ping **non viene stampato**: `cmd` lo restituisce come stringa che lo script scarta) e ferma la rete. Non apre il CLI. |
| `topo3.py` | Stessa topologia (classe `MyTopo`), senza codice di avvio. Espone il dizionario `topos = {'mytopo': (lambda: MyTopo())}` che permette a `mn` di caricarla con `--custom`. |

## Procedura passo-passo

### A) Topologia da riga di comando (`topo3.py`)

```bash
cd mininet/demo1
sudo mn --custom topo3.py --topo mytopo --test pingall
```

Atteso (da verificare): `*** Ping: testing ping reachability`, 12 coppie, `*** Results: 0% dropped (12/12 received)`.

Per il CLI interattivo: `sudo mn --custom topo3.py --topo mytopo` e poi `nodes`, `net`, `h1 ping -c 2 h4`, `exit`.

### B) Topologia da script (`topo2.py`)

```bash
sudo python3 topo2.py
ls -l topo2.pkl      # creato (di proprietà di root perché lanciato con sudo)
```

Lo script termina da solo dopo qualche secondo senza stampare i risultati dei ping. Per vederli modifica, ad esempio, `print(h1.cmd('ping -c 3', h2.IP()))`.

## Cosa osservare

- `h1` e `h2` si raggiungono tramite `s1`; `h1` e `h3` attraverso il link `s1-s2`.
- Con `net` nel CLI vedi le interfacce: `h1-eth0`, `s1-eth1`, `s1-eth3` (verso `s2`), ecc.

## Esercizi proposti

1. **Aggiungi un host `h5` a `s2`** in `topo3.py` e verifica con `pingall` che le coppie diventano 20.
2. **Interrompi il link tra gli switch**: nel CLI `link s1 s2 down`, poi `pingall`. *Soluzione attesa:* h1/h2 raggiungono solo tra loro, h3/h4 solo tra loro; le coppie tra le due LAN falliscono.
3. **Mostra l'output dei ping** in `topo2.py` con `print(...)`.
4. **Ricarica il pickle**: scrivi un secondo script che fa `pickle.load(open('topo2.pkl','rb'))` e `Mininet(topo=topo)`. Nota: per deserializzare serve che la classe `MyTopology` sia importabile.
5. **Link con banda e ritardo**: usa `TCLink` (`--link tc,bw=10,delay=20ms` da riga di comando) e misura l'RTT con `h1 ping h3`.

## Troubleshooting

- `mn: command not found`: Mininet non installato (`sudo apt install mininet`).
- `Cannot find required executable controller`: installa `openvswitch-testcontroller` o usa `--controller=none` con `--switch ovsbr` (non verificato).
- `ImportError: No module named mininet`: esegui con il Python di sistema (`/usr/bin/python3`) e non con quello di un ambiente conda/venv.
- Reti residue dopo un errore: `sudo mn -c`.

## Pulizia

```bash
sudo mn -c
rm -f topo2.pkl
rm -rf __pycache__
```
