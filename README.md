# cgnat-wireguard-vpn-vps-fritzbox
Provides a secure and easy-to-follow solution for setting up a WireGuard VPN to access your home network from anywhere, overcoming network restrictions like CGNAT.

# English Instructions

## Requirements:
- Fritzbox firmware version 7.50 or higher
- Cloud server with own IPv4 address and Linux (e.g., Hetzner CX11 with Debian or Ubuntu)

## Background:
With a Vodafone cable connection or fiber connection from Deutsche Glasfaser, using CGNAT (no own IPv4), only IPv6 works. In foreign cellular networks or WiFi, IPv6 is often not accessible, only IPv4 is. Therefore, it is necessary to use a VPN server as jumphost to access the home network from anywhere. If IPv6 works, access directly to the Fritzbox is possible. In this scenario we describe how traffic is routed through the VPN server to the home network behind the CGNAT.

## Setup Steps:

### 1. Rent a Cloud Server
Rent a cloud server (e.g., Hetzner CX11) and install Debian.

### 2. Install WireGuard on the Linux Server
sudo apt update
sudo apt install wireguard

### 3. WireGuard Configuration on the Linux Server
The key pairs are stored in the directory where you run the command. Our example allows for up to 252 clients to be configured, but you're free to adjust the VPN network and netmask to your needs. Make sure you properly plan and size the VM as well as available bandwidth in that case.

Generate key pairs:
```
wg genkey | tee server_privatekey | wg pubkey > server_publickey
wg genkey | tee fritzbox_privatekey | wg pubkey > fritzbox_publickey
wg genkey | tee client1_privatekey | wg pubkey > client_publickey
wg genkey | tee client2_privatekey | wg pubkey > client2_publickey
wg genkey | tee client3_privatekey | wg pubkey > client3_publickey
```
In this example, 10.0.0.0/24 is used for the VPN address range. This can be adjusted as needed.

WireGuard configuration file (/etc/wireguard/wg0.conf):
```
[Interface]
PrivateKey = <server_privatekey>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <fritzbox_publickey>
AllowedIPs = 192.168.77.0/24

[Peer]
PublicKey = <client1_publickey>
AllowedIPs = 10.0.0.3/24

[Peer]
PublicKey = <client2_publickey>
AllowedIPs = 10.0.0.4/24

[Peer]
PublicKey = <client3_publickey>
AllowedIPs = 10.0.0.5/24
```
Make sure to allow the WireGuard port (51820 in this example) in the Hetzner firewall settings so it is reachable.

Start WireGuard and enable it to start on boot:
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

### 4. Configure the Fritzbox
In this example, 192.168.77.0/24 is used as the Fritzbox address range. It is important to adjust this to the address range configured in your Fritzbox.

Go to Internet > Permit Access > WireGuard and select Connect networks or establish special connections. Answer Yes to the question Has this WireGuard connection already been established on the remote site? and import the configuration file.

Fritzbox WireGuard configuration file:
```
[Interface]
PrivateKey = <fritzbox_privatekey>
Address = 192.168.77.1/24

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip_or_DNS>:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```
When importing to the Fritzbox, it is important to define the correct IP address range of the VPN (in our case 10.0.0.0/24).

If you have already configured WireGuard connections on your Fritzbox, use the existing public key of your Fritzbox. Otherwise, you will get an error message. You can view the public (and private) WireGuard keys of your Fritzbox by selecting Internet > Permit Access > VPN (WireGuard) > Display WireGuard settings.

### 5. Configure the Clients
Create configuration files for each client. Those can be imported into the Wireguard client on the enduser device.

Client 1 configuration file:
```
[Interface]
PrivateKey = <client1_privatekey>
Address = 10.0.0.3/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 192.168.77.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```
Client 2 configuration file:
```
[Interface]
PrivateKey = <client2_privatekey>
Address = 10.0.0.4/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 192.168.77.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```
Client 3 configuration file:
```
[Interface]
PrivateKey = <client3_privatekey>
Address = 10.0.0.5/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 192.168.77.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```
### 6. Network Routing and NAT on the Linux Server
Enable IP forwarding:
```
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

Add NAT settings:
```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -A FORWARD -o wg0 -j ACCEPT
```

# Deutsche Anleitung
WireGuard VPN Setup mit Fritzbox und einem Cloud Server

## Voraussetzungen:
- Fritzbox Firmware-Version 7.50 oder höher
- Cloud Server mit eigener IPv4-Adresse und Linux (im Beispiel ein Hetzner CX11 Server mit Debian oder Ubuntu)

## Hintergrund:
Bei einem Vodafone Kabelanschluss oder Glasfaseranschluss mit CGNAT (ohne eigene IPv4) funktioniert nur IPv6. In ausländischen Mobilfunknetzen oder WLANs hat man oftmals keinen Zugriff auf IPv6, sondern nur IPv4. Daher ist es notwendig, einen VPN-Server als Jumphost zu nutzen, um von überall aus auf das Heimnetzwerk zugreifen zu können. Wenn IPv6 funktioniert, kann der Zugriff direkt auf die Fritzbox erfolgen. In diesem Szenario wird die Lösung über den VPN-Server mit IPv4 erklärt.

## Schritte zur Einrichtung:

### 1. Cloud Server mieten
Mieten Sie einen Cloud Server (z.B. Hetzner CX11) und installieren Sie Debian.

### 2. WireGuard auf dem Linux Server installieren
```
sudo apt update
sudo apt install wireguard
```

### 3. WireGuard Konfiguration auf dem Linux Server
Die Schlüsselpaare werden in dem Verzeichnis gespeichert, in dem man sich bei der Ausführung des Befehls befindet. Unser Beispiel ermöglicht die Konfiguration von bis zu 252 Clients, aber Sie können das VPN-Netzwerk und die Netzmaske nach Ihren Bedürfnissen anpassen. Stellen Sie sicher, dass Sie in diesem Fall die VM sowie die verfügbare Bandbreite richtig planen und dimensionieren.

Schlüsselpaare generieren:
```
wg genkey | tee server_privatekey | wg pubkey > server_publickey
wg genkey | tee fritzbox_privatekey | wg pubkey > fritzbox_publickey
wg genkey | tee client1_privatekey | wg pubkey > client_publickey
wg genkey | tee client2_privatekey | wg pubkey > client2_publickey
wg genkey | tee client3_privatekey | wg pubkey > client3_publickey
```

Im Beispiel wird 10.0.0.0/24 für den VPN Adressbereich verwendet. Das kann beliebig angepasst werden.

WireGuard Konfigurationsdatei (/etc/wireguard/wg0.conf):
```
[Interface]
PrivateKey = <server_privatekey>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <fritzbox_publickey>
AllowedIPs = 192.168.77.0/24

[Peer]
PublicKey = <client1_publickey>
AllowedIPs = 10.0.0.3/24

[Peer]
PublicKey = <client2_publickey>
AllowedIPs = 10.0.0.4/24

[Peer]
PublicKey = <client3_publickey>
AllowedIPs = 10.0.0.5/24
```
Achten Sie darauf, bei Hetzner die Firewall für den WireGuard Port (im Beispiel 51820) freizuschalten, damit dieser erreichbar ist.

WireGuard starten und beim Neustart aktivieren:
```
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### 4. Fritzbox konfigurieren
Im Beispiel wird 192.168.77.0/24 als Fritzbox Adressbereich verwendet. Das ist wichtig, den anzupassen auf den Adressbereich, den ihr in eurer Fritzbox eingestellt habt.

Gehen Sie auf Internet > Freigaben > WireGuard und wählen Sie Netzwerke koppeln oder spezielle Verbindungen herstellen. Beantworten Sie die Frage Wurde diese WireGuard-Verbindung bereits auf der Gegenstelle erstellt? mit Ja und importieren Sie die Konfigurationsdatei.

Fritzbox WireGuard Konfigurationsdatei:
```
[Interface]
PrivateKey = <fritzbox_privatekey>
Address = 192.168.77.1/24

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```
Beim Import der Fritzbox ist wichtig, den richtigen IP-Adressbereich des VPN-Bereichs (in unserem Fall 10.0.0.0/24) zu definieren.

Wenn ihr bereits WireGuard Verbindungen konfiguriert habt, verwendet den bestehenden öffentlichen Schlüssel eurer Fritzbox. Andernfalls kommt es zu einer Fehlermeldung. Ihr könnt den öffentlichen (und auch den privaten) WireGuard Schlüssel eurer Fritzbox auslesen, indem ihr unter Internet > Freigaben > VPN (WireGuard) > WireGuard-Einstellungen anzeigen auswählt.

### 5. Clients konfigurieren
Erstellen Sie Konfigurationsdateien für jeden Client. Diese können auf den Clients in Wireguard importiert werden.

Client 1 Konfigurationsdatei:
```
[Interface]
PrivateKey = <client_privatekey>
Address = 10.0.0.3/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 192.168.77.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```
Client 2 Konfigurationsdatei:
```
[Interface]
PrivateKey = <client2_privatekey>
Address = 10.0.0.4/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 192.168.77.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```
Client 3 Konfigurationsdatei:
```
[Interface]
PrivateKey = <client3_privatekey>
Address = 10.0.0.5/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server_publickey>
Endpoint = <server_ip>:51820
AllowedIPs = 192.168.77.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```

### 6. Netzwerk-Routing und NAT auf dem Linux Server
IP-Forwarding aktivieren:
```
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```
NAT-Einstellungen hinzufügen:
```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -A FORWARD -o wg0 -j ACCEPT
```
