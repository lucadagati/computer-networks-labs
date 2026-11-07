# Demo SMTP: server Postfix e dialogo a mano con il protocollo

Difficoltà: intermedia | Tempo stimato: 45 minuti | Verificato eseguendo il container: sessione SMTP completa, consegna nella casella `root` e righe di log di Postfix

## Obiettivi didattici

- Avviare un server SMTP (Postfix) e conversare con esso **a mano** via telnet, riconoscendo i comandi e i codici di risposta.
- Seguire il percorso di un messaggio: accettazione, accodamento, consegna locale.
- Capire il concetto di *open relay* e perché è pericoloso.

## Concetti teorici essenziali

- SMTP è un protocollo testuale su TCP (porta 25) con scambio comando/risposta: `EHLO`, `MAIL FROM`, `RCPT TO`, `DATA`, `QUIT`.
- Codici: `220` (pronto), `250` (ok), `354` (inizia a inviare i dati; termina con una riga contenente solo `.`), `221` (chiusura), `5xx` (errore permanente), `4xx` (temporaneo).
- Il messaggio ha intestazioni (`Subject:`, `From:`...) e corpo, separati da una riga vuota.
- **Relay**: inoltrare posta di terzi verso altri domini. Se qualunque host può farlo, il server è un *open relay* (usato per spam).

## Prerequisiti

- Docker. La porta 25 dell'host può essere occupata: la procedura non pubblica porte e lavora dentro il container.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `Dockerfile` | `ubuntu:20.04` non interattivo; installa `postfix`, `telnet`, `rsyslog`; **aggiunge a `main.cf`** `smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination` e `mynetworks = 0.0.0.0/0` (qualunque indirizzo è considerato rete fidata: *relay aperto*); all'avvio lancia `rsyslog`, `postfix` e `tail -F /var/log/mail.log`, quindi `docker logs` mostra il log di posta. |

Nota: durante l'avvio Postfix può segnalare nel log un avviso `overriding earlier entry: mynetworks=...`, perché la riga aggiunta si somma al valore già presente: vale l'ultima. Questa demo è **solo didattica: non esporre mai il container su Internet**.

## Procedura passo-passo

```bash
cd Livello_Applicazione/demo_smtp
docker build -t smtp-demo .
docker run -d --name smtp-demo smtp-demo
sleep 5
docker logs smtp-demo | tail -5
```

### Dialogo SMTP a mano

```bash
docker exec -it smtp-demo telnet localhost 25
```

Digita (le righe che iniziano con un numero sono le risposte del server):

```text
220 <hostname> ESMTP Postfix (Ubuntu)
EHLO prova.local
250-<hostname> ... 250 CHUNKING
MAIL FROM:<alice@prova.local>
250 2.1.0 Ok
RCPT TO:<root@localhost>
250 2.1.5 Ok
DATA
354 End data with <CR><LF>.<CR><LF>
Subject: Test
(riga vuota)
Ciao Bob
.
250 2.0.0 Ok: queued as 0F3843C0C99
QUIT
221 2.0.0 Bye
```

Scrivi i comandi **a mano e con calma**: se li passi tutti insieme tramite una pipe, `telnet` può chiudere la connessione prima della fine del dialogo (nella verifica la prima prova con una pipe cronometrata si è interrotta dopo `DATA`; la stessa conversazione condotta con un client TCP con pause adeguate ha avuto esito positivo).

### Verifica della consegna

```bash
docker exec smtp-demo grep -E "status=|queued" /var/log/mail.log | tail -3
docker exec smtp-demo cat /var/mail/root
```

Risultati verificati: nel log `to=<root@localhost>, relay=local, ... status=sent (delivered to mailbox)`; la casella `/var/mail/root` contiene `From alice@prova.local ...`, `Subject: Test` e il corpo.

## Cosa osservare

- L'`EHLO` elenca le estensioni supportate (`PIPELINING`, `SIZE 10240000`, `STARTTLS`, `8BITMIME`, `SMTPUTF8`, ...).
- Il messaggio riceve un identificativo di coda (`queued as ...`): lo stesso compare nei log.
- Postfix aggiunge l'intestazione `Received:` e `Message-Id`.
- Il mittente è dichiarato dal client e **non viene verificato** (nessun controllo SPF/autenticazione): è la base del *mail spoofing*.

## Esercizi proposti

I risultati attesi di questa sezione sono quelli standard di Postfix e non sono stati tutti eseguiti.

1. **Errore di sintassi.** Invia `RCPT TO` prima di `MAIL FROM`: il server risponde `503 5.5.1 Error: need MAIL command`.
2. **Destinatario inesistente.** `RCPT TO:<nessuno@localhost>`: risposta `550 ... User unknown`.
3. **Relay verso un dominio esterno.** `RCPT TO:<qualcuno@example.org>`: con `mynetworks = 0.0.0.0/0` viene accettato (relay aperto), anche se la consegna esterna dipende dalla rete del container. Come si risolve? Restringendo `mynetworks` a `127.0.0.0/8`.
4. **Cattura.** Con `docker exec smtp-demo apt-get install -y tcpdump` e `tcpdump -A -i lo port 25` vedi l'intero dialogo in chiaro.
5. **Collega al DNS.** Con la demo `../demo_DNS` aggiungi un record `MX` e capisci come un server sceglie la destinazione.
6. **Crittografia.** Prova `STARTTLS` dal client `openssl s_client -starttls smtp -connect localhost:25` (il certificato è quello di default di Postfix).

## Troubleshooting

- `Connection refused` su `localhost 25`: Postfix non è ancora partito (attendi 5 s) oppure non è in esecuzione (`docker logs smtp-demo`).
- Il log `docker logs` è vuoto: `rsyslog` non ha ancora creato `/var/log/mail.log`.
- `telnet` chiude subito dopo l'invio: scrivi i comandi a mano, uno per volta.
- Nessuna mail in `/var/mail/root`: controlla il log per `status=bounced`/`deferred`.

## Pulizia

```bash
docker rm -f smtp-demo
docker rmi smtp-demo
```
