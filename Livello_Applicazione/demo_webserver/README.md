# Demo web server: Apache (httpd) in un container

Difficoltà: base | Tempo stimato: 20 minuti | Verificato eseguendo il container (risposta `HTTP/1.1 200 OK`)

## Obiettivi didattici

- Avviare un web server in un container e servire una pagina statica.
- Vedere una richiesta/risposta HTTP completa (riga di stato, intestazioni, corpo).
- Capire il mapping delle porte (`-p host:container`).

## Concetti teorici essenziali

- HTTP è un protocollo testuale richiesta/risposta su TCP (porta 80 di default).
- Intestazioni utili: `Server`, `Content-Type`, `Content-Length`, `Last-Modified`, `ETag`.
- Un container ha una rete propria: per raggiungerlo dall'host serve pubblicare la porta (`-p`) oppure usare l'IP del container/la rete Docker.

## Prerequisiti

- Docker; accesso a Internet per scaricare `httpd` e (nella build) `curl`/`wget`.

## Struttura dei file

| File | Cosa fa |
|---|---|
| `Dockerfile` | Parte da `httpd` (Apache ufficiale), copia `index.html` in `/usr/local/apache2/htdocs/` e installa `curl` e `wget` per poter fare richieste dall'interno del container. |
| `index.html` | Pagina HTML minima in italiano: titolo "Benvenuto!", `h1` "Ciao Mondo!" e un paragrafo. |

## Procedura passo-passo

```bash
cd Livello_Applicazione/demo_webserver
docker build -t web-demo .
docker run -d --name web-demo -p 8080:80 web-demo
curl -i http://localhost:8080/
```

Output atteso (verificato, i valori di data/versione cambiano):

```text
HTTP/1.1 200 OK
Date: ...
Server: Apache/2.4.xx (Unix)
Last-Modified: ...
ETag: "cb-..."
Accept-Ranges: bytes
Content-Length: 203
Content-Type: text/html

<!DOCTYPE html>
<html lang="it">
...
```

Se la porta 8080 è occupata scegli un'altra porta. Verifica interna (curl installato nell'immagine):

```bash
docker exec web-demo curl -s -i http://localhost/ | head -3
docker logs web-demo        # log di accesso/errore di Apache
```

## Cosa osservare

- `Content-Length: 203` coincide con la dimensione di `index.html` (`wc -c index.html`).
- La seconda richiesta con `curl -i -H "If-None-Match: <ETag>"` restituisce `304 Not Modified` (comportamento standard HTTP, non eseguito qui).
- `docker logs` mostra ogni richiesta ricevuta.

## Esercizi proposti

1. **Richiesta a mano con telnet/netcat.** `printf 'GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | nc localhost 8080`: ottieni la stessa risposta. Cosa succede senza la riga `Host`? (HTTP/1.1 la richiede.)
2. **Risorsa inesistente.** `curl -i http://localhost:8080/non-esiste`: codice 404.
3. **Modifica la pagina** senza ricostruire l'immagine: `docker cp index.html web-demo:/usr/local/apache2/htdocs/` e ricarica.
4. **Cattura con Wireshark** su `docker0` o sul loopback: individua handshake, richiesta GET e risposta 200.
5. **Più server.** Avvia un secondo container su `-p 8081:80` e confronta le risposte.

## Troubleshooting

- `port is already allocated`: cambia la porta host (`-p 8090:80`).
- `curl: (7) Failed to connect`: container non partito (`docker ps -a`, `docker logs web-demo`).
- Pagina vecchia: ricostruisci l'immagine (`docker build`) e ricrea il container.

## Pulizia

```bash
docker rm -f web-demo
docker rmi web-demo
```
