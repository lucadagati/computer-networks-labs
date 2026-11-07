# Mininet demo2: firewall iptables su tre host

Difficoltà: intermedia | Tempo stimato: 60 minuti | **Non verificato in questo ambiente** (Mininet non installato; verificata solo la sintassi Python; i comportamenti sono dedotti dalla lettura degli script)

## Obiettivi didattici

- Applicare politiche di filtraggio diverse a ciascun host con `iptables`.
- Distinguere i target `DROP` (silenzioso) e `REJECT` (risposta ICMP di errore), e usare `LOG` e `-m limit`.
- Capire che ogni host Mininet ha il proprio firewall (network namespace separato).

## Concetti teorici essenziali

- Catena `INPUT`: pacchetti destinati all'host. Le regole sono valutate in ordine; vale la prima che corrisponde.
- `DROP` scarta senza avvisare; `REJECT` risponde (ICMP port/host unreachable o TCP RST).
- `-m limit --limit 1/min`: regola che accetta solo un certo ritmo di pacchetti, con una **raffica iniziale** (`--limit-burst`, default 5) prima del limite.

## Prerequisiti

- Mininet, OVS, controller, `sudo`; `ssh`/`sshd` e un server web (`python3 -m http.server`) solo per i test 1 e 3.

## Struttura dei file

| File | Contenuto |
|---|---|
| `mininet_demo.py` | `setup_network()`: controller `c0`; switch `s1`, `s2`; host `h1` su `s1`, `h2` e `h3` su `s2`; link `s1-s2`. Dopo `net.start()` esegue `hN.cmd('./hN_iptables.sh')` per i tre host e apre il CLI. |
| `host1_iptables.sh` | `iptables -F`; accetta solo TCP/22 in ingresso; `DROP` di tutto il resto in `INPUT`. |
| `host2_iptables.sh` | `iptables -F`; per i pacchetti da `HOST1_IP` fa `LOG` (prefisso `Blocked HOST1: `) e poi `REJECT`; accetta tutto il resto. Contiene `HOST1_IP="192.168.1.1"`. |
| `host3_iptables.sh` | `iptables -F`; accetta TCP/80 con `limit 1/min`, poi `REJECT` per TCP/80; accetta il resto. |

Topologia: `h1 - s1 - s2 - {h2, h3}`. Indirizzi Mininet di default: `h1 = 10.0.0.1`, `h2 = 10.0.0.2`, `h3 = 10.0.0.3`.

## Punti critici da conoscere PRIMA di eseguire (dedotti dal codice)

1. **Permessi e cartella.** Lo script usa percorsi relativi `./host1_iptables.sh` e nel repository i `.sh` **non sono eseguibili**. Esegui `chmod +x *.sh` e lancia lo script **da dentro `mininet/demo2`**. Altrimenti `cmd` restituisce un errore che lo script scarta senza avvisarti e i firewall non vengono applicati.
2. **Indirizzo di Host1 sbagliato.** `host2_iptables.sh` usa `HOST1_IP="192.168.1.1"` mentre `h1` ha `10.0.0.1`: così la regola non corrisponde a nessun pacchetto. Per il test 2 modifica in `HOST1_IP="10.0.0.1"` (esercizio 1).
3. **sshd e web server non sono avviati.** Per provare SSH o HTTP devi avviarli tu negli host (vedi esercizi).
4. **LOG nei namespace.** Nei kernel recenti i messaggi `LOG` generati in un network namespace non-iniziale possono non comparire in `/var/log/syslog` (opzione `net.netfilter.nf_log_all_netns`); controlla anche `dmesg`.

## Procedura passo-passo

```bash
cd mininet/demo2
chmod +x host1_iptables.sh host2_iptables.sh host3_iptables.sh
sed -i 's/HOST1_IP="192.168.1.1"/HOST1_IP="10.0.0.1"/' host2_iptables.sh   # correzione
sudo python3 mininet_demo.py
```

Nel CLI (`mininet>`):

1. Verifica regole: `h1 iptables -L -n -v`, `h2 iptables -L -n -v`, `h3 iptables -L -n -v`.
2. **Host1, solo SSH.** `h2 ping -c 2 h1` deve fallire (100% packet loss: `DROP`, nessun messaggio di errore, solo timeout). Per SSH avvia sshd su `h1` (richiede `openssh-server` installato nell'ospite e la directory `/run/sshd`): `h1 /usr/sbin/sshd`, poi `h2 ssh <utente>@h1`. In alternativa prova una porta TCP qualsiasi con `h2 nc -vz h1 22`.
3. **Host2, rifiuta Host1.** `h1 ping -c 2 h2` deve restituire `Destination Port Unreachable`/`Destination Host Prohibited` (a differenza del `DROP`, il `REJECT` fa rispondere subito). Log: `h2 dmesg | grep "Blocked HOST1"` o `grep "Blocked HOST1" /var/log/syslog`.
4. **Host3, limita HTTP.** Avvia il server: `h3 python3 -m http.server 80 &`. Poi `h1 curl -s -o /dev/null -w "%{http_code}\n" h3:80` ripetuto: le prime richieste (fino a circa 5, per il burst di default) dovrebbero avere successo, poi `curl: (7) Failed to connect ... Connection refused` per il `REJECT` (con ritmo di rientro di 1/min). Il README originale diceva "dopo la prima"; con `limit-burst` di default 5 il comportamento atteso è diverso (da verificare).
5. `pingall` mostrerà molte perdite: è atteso.

## Cosa osservare

- Differenza di comportamento tra `DROP` (timeout) e `REJECT` (errore immediato).
- I contatori di `iptables -L -v -n` crescono sulle regole che scattano.
- `h1` non riesce a ricevere nemmeno le risposte ai propri ping verso altri host: gli echo reply entrano nella catena `INPUT`, dove tutto tranne TCP/22 viene scartato (comportamento dedotto dalle regole, non verificato con Mininet).

## Esercizi proposti

1. **Correggi `HOST1_IP`** e dimostra con `h2 iptables -L -n -v` che il contatore della regola `REJECT` cresce dopo `h1 ping -c 2 h2`.
2. **Consenti il ping verso h1**: aggiungi in `host1_iptables.sh` `iptables -I INPUT -p icmp -j ACCEPT` prima del `DROP` finale. Cosa cambia in `h2 ping h1`?
3. **Stateful firewall**: sostituisci le regole di h1 con `iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT` prima del `DROP`: ora h1 riceve le risposte alle proprie connessioni.
4. **Cambia il limite** di h3 in `--limit 10/min --limit-burst 2` e riprova il test con `curl`.
5. **Cattura**: `h2 tcpdump -n -i h2-eth0 icmp &` mentre h1 pinga h2, e osserva i messaggi ICMP di errore generati dal `REJECT`.

## Troubleshooting

- Nessun firewall applicato: vedi punto 1 (permessi/cartella).
- `h2 ssh h1` fallisce con `Connection refused` o timeout: sshd non è in esecuzione in h1 (punto 3).
- `iptables: command not found`: installa `iptables`.
- Reti residue: `sudo mn -c`.

## Pulizia

`exit` nel CLI; `sudo mn -c`. Le regole `iptables` vivono nei namespace degli host e spariscono con la rete.
