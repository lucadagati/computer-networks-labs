FROM ubuntu:22.04

# Aggiorna la lista dei pacchetti e installa telnetd, curl, wget e dnsutils (per dig e nslookup)
RUN apt-get update && apt-get install -y telnetd curl wget dnsutils && apt-get clean && rm -rf /var/lib/apt/lists/*

# Crea un nuovo utente e impostala password
RUN useradd demo && echo "demo:demo123" | chpasswd

# Avvia il server telnet al boot del container
CMD ["/usr/sbin/in.telnetd", "-debug", "23"]
