# Własny VPN: Headscale + Tailscale + WireGuard

Cel: jedna prywatna sieć łącząca **VPS, serwer domowy, telefon i Windows**, z funkcjami:

- **Exit node**: cały ruch idzie przez VPS (zmiana publicznego IP, np. w publicznym Wi-Fi).
- **Dostęp do serwera domowego** z każdego urządzenia, także do całego LAN-u domowego (subnet router).
- **Opcjonalnie czysty WireGuard** dla sprzętu bez Tailscale (router, stare urządzenia).

Skoro chmura Tailscale ci nie przeszkadza, a chcesz „po swojemu", rekomenduję: **Headscale na VPS** jako własny serwer koordynacji. Wszystkie klienty to oficjalne aplikacje Tailscale. Pod spodem działa WireGuard, ale kluczami zarządza twój serwer.

## Architektura

```
                    ┌──────────────────────────────┐
                    │ VPS (publiczne IP)           │
                    │  • Headscale (koordynacja)   │
                    │  • Tailscale: EXIT NODE      │
                    └──────────────┬───────────────┘
                                   │ tailnet 100.64.0.0/10
      ┌───────────────┬────────────┴───┬──────────────────┐
┌─────┴─────┐   ┌─────┴─────┐    ┌─────┴─────┐    ┌───────┴──────────┐
│ Telefon   │   │ Windows   │    │ Serwer    │    │ (opcja) sprzęt   │
│ Tailscale │   │ Tailscale │    │ domowy    │    │ z gołym WireGuard│
│           │   │           │    │ SUBNET    │    │ przez most       │
└───────────┘   └───────────┘    │ ROUTER    │    └──────────────────┘
                                 │192.168.1.0/24
                                 └───────────┘
```

| Urządzenie | Rola |
|---|---|
| VPS | Headscale + exit node |
| Serwer domowy | Węzeł + subnet router (udostępnia LAN `192.168.1.0/24`) |
| Telefon | Klient, może używać exit node |
| Windows | Klient, może używać exit node |

---

## Krok 1: VPS: Headscale

Wymagania: domena (np. `hs.twojadomena.pl`) wskazująca na VPS, otwarte porty `443/tcp`, `80/tcp` (certyfikat), `3478/udp` (STUN, opcjonalnie).

```bash
# Pakiet z https://github.com/juanfont/headscale/releases
sudo dpkg -i headscale_*_linux_amd64.deb
sudo nano /etc/headscale/config.yaml
```

Kluczowe ustawienia:

```yaml
server_url: https://hs.twojadomena.pl
listen_addr: 127.0.0.1:8080
prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
dns:
  magic_dns: true
  base_domain: tailnet.lan
  nameservers:
    global:
      - 1.1.1.1
      - 9.9.9.9
```

Reverse proxy z TLS, np. Caddy (`/etc/caddy/Caddyfile`):

```
hs.twojadomena.pl {
    reverse_proxy 127.0.0.1:8080
}
```

```bash
sudo systemctl enable --now headscale caddy
sudo headscale users create moj
```

---

## Krok 2: Dodanie urządzeń

### Linux (VPS i serwer domowy)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --login-server https://hs.twojadomena.pl
```

Komenda wypisze klucz. Zatwierdź węzeł na VPS:

```bash
sudo headscale nodes register --user moj --key <KLUCZ>
```

Lub użyj klucza preauth (wygodne przy automatyzacji):

```bash
sudo headscale preauthkeys create --user moj --expiration 1h
sudo tailscale up --login-server https://hs.twojadomena.pl --authkey <KLUCZ>
```

### Windows

1. Zainstaluj Tailscale z oficjalnej strony.
2. Przed pierwszym logowaniem ustaw serwer w rejestrze (PowerShell jako administrator):

```powershell
reg add "HKLM\SOFTWARE\Tailscale IPN" /v LoginURL /t REG_SZ /d "https://hs.twojadomena.pl" /f
```

3. Zrestartuj Tailscale, zaloguj się i zatwierdź węzeł komendą `headscale nodes register` jak wyżej.

Alternatywa w CLI: `tailscale up --login-server https://hs.twojadomena.pl`.

### Telefon (Android / iOS)

1. Zainstaluj aplikację Tailscale.
2. Android: menu ⋮ → **Use an alternate server** → wpisz `https://hs.twojadomena.pl`.
3. iOS: Ustawienia aplikacji → **Account → Use custom coordination server**.
4. Zaloguj się i zatwierdź węzeł na VPS (`headscale nodes register`).

---

## Krok 3: Exit node (zmiana IP)

Na **VPS**:

```bash
echo 'net.ipv4.ip_forward = 1'          | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

sudo tailscale up --login-server https://hs.twojadomena.pl --advertise-exit-node
```

Zatwierdź trasy na serwerze Headscale:

```bash
headscale nodes list                 # znajdź ID VPS
headscale nodes list-routes -i <ID>
headscale nodes approve-routes -i <ID> -r 0.0.0.0/0,::/0
```

(Na starszych wersjach: `headscale routes list` i `headscale routes enable -r <ID_TRASY>`.)

Używanie exit node:

| Urządzenie | Jak włączyć |
|---|---|
| Linux | `sudo tailscale set --exit-node=<nazwa-vps> --exit-node-allow-lan-access=true` |
| Windows | ikona Tailscale → **Exit nodes** → wybierz VPS |
| Android / iOS | w aplikacji → **Exit node** → wybierz VPS |
| Wyłączenie | `sudo tailscale set --exit-node=` |

Sprawdź: `curl ifconfig.me` powinno pokazać IP VPS-a.

Exit node w domu (powrót do polskiego IP, gdy jesteś za granicą): to samo na serwerze domowym (`--advertise-exit-node` plus zatwierdzenie tras). Możesz mieć kilka exit nodów i przełączać je jak lokalizacje.

---

## Krok 4: Dostęp do serwera domowego

Najprostszy wariant: serwer domowy jest zwykłym węzłem, więc łączysz się przez jego adres tailnetu lub nazwę MagicDNS:

```bash
ssh uzytkownik@serwer-domowy      # albo 100.64.0.x
```

Żeby dostać się do **całego LAN-u** (drukarka, NAS, kamery, router), zrób z serwera subnet router:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

sudo tailscale up --login-server https://hs.twojadomena.pl --advertise-routes=192.168.1.0/24
```

```bash
# na VPS:
headscale nodes approve-routes -i <ID_SERWERA_DOMOWEGO> -r 192.168.1.0/24
```

Na Linuksie (klient) dodaj `--accept-routes`:

```bash
sudo tailscale set --accept-routes
```

Windows i telefon przyjmują trasy automatycznie. Po tym działa np. `http://192.168.1.50` z telefonu poza domem.

Wskazówka: jeśli na wyjeździe trafisz do sieci z tą samą podsiecią `192.168.1.0/24`, trasy będą się gryzły. Zmień domową podsieć na coś rzadszego, np. `192.168.77.0/24`.

---

## Krok 5 (opcjonalny): czysty WireGuard przez most

Gdy masz urządzenie, które umie tylko WireGuard (router, kamera, stary telefon), podłącz je do węzła-bramy (np. serwera domowego z Tailscale).

Na bramie `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <KLUCZ_PRYWATNY_BRAMY>
PostUp   = iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o tailscale0 -j MASQUERADE; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o tailscale0 -j MASQUERADE; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT

[Peer]
PublicKey = <KLUCZ_PUBLICZNY_URZADZENIA>
AllowedIPs = 10.10.0.2/32
```

```bash
sudo apt install wireguard
sudo systemctl enable --now wg-quick@wg0
sudo tailscale up --login-server https://hs.twojadomena.pl --advertise-routes=192.168.1.0/24,10.10.0.0/24
```

Zatwierdź trasę `10.10.0.0/24` w Headscale. Urządzenie WireGuard:

```ini
[Interface]
Address = 10.10.0.2/24
PrivateKey = <KLUCZ_PRYWATNY_URZADZENIA>

[Peer]
PublicKey = <KLUCZ_PUBLICZNY_BRAMY>
Endpoint = <PUBLICZNY_ADRES_BRAMY>:51820
AllowedIPs = 10.10.0.0/24, 100.64.0.0/10
PersistentKeepalive = 25
```

Brama musi być osiągalna z zewnątrz: przekierowanie portu `51820/udp` na routerze domowym albo postawienie bramy na VPS.

---

## Krok 6: ACL (kto co widzi)

Plik `/etc/headscale/acl.hujson` (ścieżka w `config.yaml` → `policy.path`). Tagi wymagają przypisania węzłom (`headscale nodes tag -i <ID> -t tag:vps`).

```json
{
  "tagOwners": {
    "tag:vps":  ["moj"],
    "tag:home": ["moj"]
  },
  "acls": [
    { "action": "accept", "src": ["moj"], "dst": ["*:*"] }
  ],
  "autoApprovers": {
    "exitNode": ["tag:vps"],
    "routes": { "192.168.1.0/24": ["tag:home"] }
  }
}
```

Zacznij od „wszystko dla właściciela", potem zawężaj (np. telefon tylko do portów serwera domowego).

---

## Bezpieczeństwo i diagnostyka

- Otwórz na VPS tylko `443/tcp`, `80/tcp`, `3478/udp` (i `51820/udp` jeśli używasz kroku 5). SSH ogranicz do tailnetu lub kluczy.
- Klucze prywatne: `chmod 600`, nigdy w repo (`*.key` do `.gitignore`).
- Zrób backup `/var/lib/headscale/` (baza z węzłami i kluczami).
- Diagnostyka:
  - `tailscale status`
  - `tailscale netcheck`
  - `tailscale ping <węzeł>` (pokaże, czy połączenie jest bezpośrednie czy przez relay DERP)
  - `sudo wg show`
  - `ip route`
  - `sysctl net.ipv4.ip_forward`
- Jeśli exit node nie przepuszcza ruchu, zwykle brakuje `ip_forward` albo zatwierdzenia tras w Headscale.
- Nazwy komend Headscale zmieniają się między wersjami (`routes` → `approve-routes`). Sprawdź `headscale --help` na swojej wersji.

## Kolejność wdrożenia (checklista)

- [ ] Domena wskazuje na VPS
- [ ] Headscale + Caddy działa (`https://hs.twojadomena.pl`)
- [ ] VPS dodany jako węzeł
- [ ] Exit node zatwierdzony
- [ ] Serwer domowy dodany + subnet router zatwierdzony
- [ ] Windows i telefon dodane
- [ ] Test: `curl ifconfig.me` przez exit node pokazuje IP VPS
- [ ] Test: dostęp do `192.168.1.x` z telefonu poza domem
- [ ] ACL i backup
