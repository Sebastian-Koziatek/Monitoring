![Grafana](/grafiki/m1-pasted-image-20260608081708.png)

Grafana CLI (`grafana cli`) to narzędzie wiersza poleceń instalowane razem z Grafaną. Umożliwia zarządzanie pluginami, operacje administracyjne i zadania serwisowe — bez dostępu do interfejsu webowego.

## Struktura polecenia

```bash
grafana cli [opcje globalne] <grupa> <komenda> [argumenty]
```

Dostępne grupy komend:

| Grupa | Opis |
|-------|------|
| `plugins` | Zarządzanie pluginami |
| `admin` | Operacje administracyjne |

---

## Zarządzanie pluginami

### Listowanie pluginów

**Zainstalowane pluginy:**

```bash
grafana cli plugins ls
```

**Dostępne pluginy w oficjalnym repozytorium:**

```bash
grafana cli plugins list-available
```

**Filtrowanie listy:**

```bash
grafana cli plugins list-available | grep zabbix
```

### Instalacja pluginu

```bash
sudo grafana cli plugins install <plugin-id>
```

**Przykłady:**

```bash
# Plugin Zabbix (data source + app)
sudo grafana cli plugins install alexanderzobnin-zabbix-app

# Panel - zegar
sudo grafana cli plugins install grafana-clock-panel

# Worldmap Panel - mapa geograficzna
sudo grafana cli plugins install grafana-worldmap-panel
```

> **Wymagane po każdej zmianie pluginów:**
> ```bash
> sudo systemctl restart grafana-server
> ```

### Instalacja konkretnej wersji

```bash
sudo grafana cli plugins install <plugin-id> <wersja>
```

Przykład:

```bash
sudo grafana cli plugins install alexanderzobnin-zabbix-app 4.4.0
```

### Instalacja z URL lub pliku lokalnego

Przydatne w środowiskach bez dostępu do internetu lub przy własnych pluginach:

```bash
sudo grafana cli \
  --pluginUrl https://example.com/myplugin-1.0.0.zip \
  plugins install my-plugin-id
```

Z pliku lokalnego:

```bash
sudo grafana cli \
  --pluginsDir /var/lib/grafana/plugins \
  --pluginUrl file:///tmp/myplugin-1.0.0.zip \
  plugins install my-plugin-id
```

### Aktualizacja pluginów

**Wybrany plugin:**

```bash
sudo grafana cli plugins update <plugin-id>
```

**Wszystkie pluginy jednocześnie:**

```bash
sudo grafana cli plugins update-all
```

### Usuwanie pluginu

```bash
sudo grafana cli plugins remove <plugin-id>
```

---

## Komendy administracyjne

### Reset hasła administratora

Jeśli zapomniano hasła do konta `admin`:

```bash
sudo grafana cli admin reset-admin-password <nowe-haslo>
```

Komenda modyfikuje bazę danych Grafany bezpośrednio. Działa niezależnie od tego, czy serwer jest uruchomiony.

### Sprawdzenie wersji Grafany

```bash
grafana --version
```

---

## Opcje globalne

| Opcja | Opis |
|-------|------|
| `--pluginsDir <ścieżka>` | Katalog z pluginami (domyślnie `/var/lib/grafana/plugins`) |
| `--pluginUrl <url>` | Alternatywne URL repozytorium lub plik lokalny |
| `--insecure` | Pomiń weryfikację certyfikatu TLS |
| `--debug` | Tryb debugowania (szczegółowe logi) |
| `--homepath <ścieżka>` | Ścieżka do katalogu głównego Grafany |

**Przykład — instalacja z niestandardowym katalogiem:**

```bash
sudo grafana cli \
  --pluginsDir /opt/grafana-plugins \
  plugins install grafana-clock-panel
```

---

## Typowe scenariusze

### Instalacja i aktywacja pluginu Zabbix

```bash
# 1. Zainstaluj plugin
sudo grafana cli plugins install alexanderzobnin-zabbix-app

# 2. Zrestartuj Grafanę
sudo systemctl restart grafana-server

# 3. Sprawdź status usługi
sudo systemctl status grafana-server

# 4. W UI: Administration > Plugins > wyszukaj "Zabbix" > Enable
# 5. W UI: Connections > Data Sources > Add data source > Zabbix
```

### Aktualizacja wszystkich pluginów

```bash
# Zaktualizuj wszystkie pluginy
sudo grafana cli plugins update-all

# Zrestartuj Grafanę
sudo systemctl restart grafana-server

# Sprawdź logi pod kątem błędów
sudo journalctl -u grafana-server -n 50
```

### Dokumentacja zainstalowanych pluginów

```bash
# Wylistuj zainstalowane pluginy i zapisz do pliku
grafana cli plugins ls > zainstalowane_pluginy.txt
cat zainstalowane_pluginy.txt
```

---

## Lokalizacja plików Grafany

| Plik / Katalog | Opis |
|----------------|------|
| `/etc/grafana/grafana.ini` | Główny plik konfiguracyjny |
| `/var/lib/grafana/plugins/` | Katalog zainstalowanych pluginów |
| `/var/log/grafana/grafana.log` | Logi serwera Grafana |
| `/var/lib/grafana/grafana.db` | Baza danych SQLite (domyślna) |
| `/usr/share/grafana/` | Pliki binarne i frontend Grafany |

---

## Dobre praktyki

- Zawsze restartuj `grafana-server` po instalacji, aktualizacji lub usunięciu pluginu.
- Sprawdzaj kompatybilność pluginu z wersją Grafany przed aktualizacją.
- W środowiskach produkcyjnych testuj nowe wersje pluginów najpierw w staging.
- Używaj opcji `--pluginUrl` do dystrybuowania pluginów z wewnętrznego serwera w sieciach bez dostępu do internetu.
- Prowadź listę zainstalowanych pluginów w systemie kontroli wersji (`grafana cli plugins ls`).

---

## Podsumowanie

Grafana CLI to narzędzie do zarządzania pluginami i administracji serwerem z poziomu terminala. Kluczowe operacje to: instalacja pluginów (`plugins install`), ich aktualizacja (`plugins update-all`) oraz reset hasła administratora (`admin reset-admin-password`). Po każdej zmianie pluginów wymagany jest restart usługi `grafana-server`.
