# Tailscale + WireGuard po swojemu

Tailscale to w środku WireGuard plus warstwa koordynacji (klucze, NAT traversal, ACL, DNS). Skoro chcesz to zrobić „po swojemu", są trzy sensowne drogi. Wybierz jedną.

| Wariant | Co dostajesz | Trudność |
|---|---|---|
| **A. Headscale** (własny serwer koordynacji) + klienty Tailscale | Pełna kontrola, brak zależności od chmury Tailscale | średnia |
| **B. Tailscale + czysty WireGuard** (subnet router jako most) | Urządzenia bez Tailscale (router, stary sprzęt) widzą tailnet | niska |
| **C. Własny mesh na gołym WireGuard** (hub-and-spoke + skrypty) | Zero zewnętrznych komponentów | niska, ale ręczne zarządzanie |

Rekomendacja: **A** jeśli chcesz mieć wszystko u siebie, **B** jeśli chcesz tylko połączyć istniejące WireGuard-owe urządzenia z Tailscale.

---

## Wariant A: Headscale + klienty Tailscale

### 1. Serwer (VPS z publicznym IP)

```bash
# Pobierz najnowszy pakiet z https://github.com/juanfont/headscale/releases
sudo dpkg -i headscale_*_linux_amd64.deb
sudo nano /etc/headscale/config.yaml
```

Kluczowe ustawienia w `config.yaml`:

```yaml
server_url: https://hs.twojadomena.pl
listen_addr: 0.0.0.0:8080
prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
dns:
  magic_dns: true
  base_domain: tailnet.lan
```

```bash
sudo systemctl enable --now headscale
sudo headscale users create moj
```

Postaw reverse proxy (Caddy/nginx) z TLS na `hs.twojadomena.pl` → `127.0.0.1:8080`.

### 2. Klienci

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --login-server https://hs.twojadomena.pl
# na serwerze zatwierdź węzeł:
sudo headscale nodes register --user moj --key <KLUCZ_Z_KLIENTA>
```

Klucz preauth (dla automatyzacji):

```bash
headscale preauthkeys create --user moj --reusable --expiration 24h
sudo tailscale up --login-server https://hs.twojadomena.pl --authkey <KLUCZ>
```

### 3. ACL (plik `/etc/headscale/acl.json`)

```json
{
  "acls": [
    { "action": "accept", "src": ["*"], "dst": ["*:*"] }
  ]
}
```

Zacznij od „wszystko wszędzie", potem zawężaj per użytkownik/tag.

---

## Wariant B: Tailscale + czysty WireGuard (most)

Cel: urządzenie z gołym WireGuardem (np. router, telefon bez Tailscale) ma dostęp do tailnetu i odwrotnie.

```
[urządzenie WG] ──WG──> [BRAMA: Tailscale + WireGuard] ──Tailscale──> [reszta tailnetu]
 10.10.0.2/24             wg0: 10.10.0.1/24 | tailscale0: 100.x.y.z
```

### 1. Na bramie (Linux)

```bash
sudo apt install wireguard
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
chmod 600 /etc/wireguard/server.key

echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

`/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <ZAWARTOŚĆ server.key>
# NAT: ruch z WG wychodzi do tailnetu jako adres bramy
PostUp   = iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o tailscale0 -j MASQUERADE; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o tailscale0 -j MASQUERADE; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT

[Peer]
# urządzenie z czystym WireGuard
PublicKey = <KLUCZ_PUBLICZNY_URZĄDZENIA>
AllowedIPs = 10.10.0.2/32
```

```bash
sudo systemctl enable --now wg-quick@wg0
```

### 2. Udostępnij podsieć WG w tailnecie

```bash
sudo tailscale up --advertise-routes=10.10.0.0/24
# zatwierdź trasę w panelu Tailscale (albo: headscale routes enable -r <ID>)
```

Dzięki temu węzły Tailscale trafiają do `10.10.0.0/24` przez bramę.

### 3. Na urządzeniu z czystym WireGuard

```ini
[Interface]
Address = 10.10.0.2/24
PrivateKey = <KLUCZ_PRYWATNY_URZĄDZENIA>
DNS = 100.100.100.100        # MagicDNS (opcjonalnie)

[Peer]
PublicKey = <server.pub>
Endpoint = brama.twojadomena.pl:51820
# 100.64.0.0/10 = cały tailnet
AllowedIPs = 10.10.0.0/24, 100.64.0.0/10
PersistentKeepalive = 25
```

Test: `ping 100.x.y.z` (dowolny węzeł tailnetu) z urządzenia WG.

---

## Wariant C: Własny mesh na gołym WireGuard

Jeśli Tailscale jest Ci zbędny: hub-and-spoke, hub ma publiczny IP i przekazuje ruch między spoke'ami.

**Hub** (`wg0.conf`):

```ini
[Interface]
Address = 10.20.0.1/24
ListenPort = 51820
PrivateKey = <HUB_PRIV>

[Peer]   # laptop
PublicKey = <LAPTOP_PUB>
AllowedIPs = 10.20.0.2/32

[Peer]   # serwer domowy
PublicKey = <DOM_PUB>
AllowedIPs = 10.20.0.3/32, 192.168.1.0/24
```

**Spoke**:

```ini
[Interface]
Address = 10.20.0.2/24
PrivateKey = <LAPTOP_PRIV>

[Peer]
PublicKey = <HUB_PUB>
Endpoint = hub.twojadomena.pl:51820
AllowedIPs = 10.20.0.0/24
PersistentKeepalive = 25
```

Na hubie włącz `ip_forward`. Żeby spoke'i gadały bezpośrednio (P2P), dopisz wzajemnie `[Peer]` z `Endpoint` — to właśnie robi Tailscale automatycznie (NAT traversal), a tu robisz to ręcznie lub skryptem.

---

## Dobre praktyki

- Nie koliduj adresacji: tailnet `100.64.0.0/10`, WG `10.x`, LAN `192.168.x`.
- Klucze prywatne: `chmod 600`, nigdy w repo (dodaj `*.key` do `.gitignore`).
- Firewall: otwórz tylko `51820/udp` (WG) i `443/tcp` (Headscale).
- Jeśli nie działa: `sudo wg show`, `tailscale status`, `tailscale netcheck`, `ip route`, `sysctl net.ipv4.ip_forward`.
- Wersje i flagi sprawdź w aktualnych dokumentacjach Headscale i Tailscale — konfiguracje zmieniają się między wydaniami.

---

## Co dalej — doprecyzuj „po swojemu"

Żebym dopasował plik, napisz:

1. Czy chcesz **własny serwer** (Headscale), czy zostać przy chmurze Tailscale?
2. Jakie urządzenia łączysz (router, telefon, VPS, Windows)?
3. Czy to ma być dostęp do całej sieci domowej, czy tylko pojedyncze usługi?
