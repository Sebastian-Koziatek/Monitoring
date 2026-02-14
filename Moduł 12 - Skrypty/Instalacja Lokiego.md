# Skrypt instalacji i deinstalacji Lokiego

Automatyczny skrypt do instalacji i usuwania Grafana Loki na systemach Ubuntu/Debian. Skrypt obsługuje pełny cykl życia aplikacji - od instalacji przez konfigurację, aż po całkowite usunięcie wraz z plikami konfiguracyjnymi.

---

## Opis skryptu

Skrypt zawiera dwie główne funkcje:
- **Instalacja Lokiego** - automatyczna instalacja wersji 2.9.3 z oficjalnego repozytorium GitHub
- **Deinstalacja Lokiego** - całkowite usunięcie aplikacji wraz z danymi i konfiguracją

---

## Pełny skrypt

```bash
#!/bin/bash
set -e

LOKI_VERSION="2.9.3"

function install_loki() {
	echo "Rozpoczynam instalację Lokiego w wersji ${LOKI_VERSION}..."
	
	# Krok 1: Aktualizacja systemu i instalacja wymaganych narzędzi
	sudo apt update
	sudo apt upgrade -y
	sudo apt install -y wget unzip curl
	
	# Krok 2: Pobranie Lokiego
	cd /tmp
	wget https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip
	unzip loki-linux-amd64.zip
	sudo mv loki-linux-amd64 /usr/local/bin/loki
	sudo chmod +x /usr/local/bin/loki
	rm -f loki-linux-amd64.zip
	
	# Krok 3: Weryfikacja instalacji
	echo "Sprawdzanie wersji Lokiego:"
	loki --version
	
	# Krok 4: Utworzenie użytkownika systemowego
	sudo useradd --system --no-create-home --shell /bin/false loki || true
	
	# Krok 5: Utworzenie katalogów danych (z katalogiem WAL!)
	sudo mkdir -p /var/lib/loki/{index,cache,chunks,wal,compactor}
	sudo mkdir -p /etc/loki
	sudo chown -R loki:loki /var/lib/loki
	sudo chown -R loki:loki /etc/loki
	
	# Krok 6: Utworzenie pliku konfiguracyjnego
	sudo tee /etc/loki/loki-config.yaml > /dev/null << 'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: info

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  max_transfer_retries: 0
  wal:
    enabled: true
    dir: /var/lib/loki/wal

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /var/lib/loki/index
    cache_location: /var/lib/loki/cache
    shared_store: filesystem
  filesystem:
    directory: /var/lib/loki/chunks

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20

compactor:
  working_directory: /var/lib/loki/compactor
  shared_store: filesystem

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
EOF
	
	# Krok 7: Utworzenie usługi systemd
	sudo tee /etc/systemd/system/loki.service > /dev/null << 'EOF'
[Unit]
Description=Loki Log Aggregation System
Documentation=https://grafana.com/docs/loki/latest/
After=network.target

[Service]
Type=simple
User=loki
Group=loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
	
	# Krok 8: Uruchomienie usługi Loki
	sudo systemctl daemon-reload
	sudo systemctl enable loki
	sudo systemctl start loki
	
	echo ""
	echo "Loki został zainstalowany i uruchomiony!"
	echo "Sprawdzanie statusu usługi..."
	sleep 3
	sudo systemctl status loki --no-pager
	
	echo ""
	echo "Weryfikacja działania:"
	curl -s http://localhost:3100/ready && echo "Loki jest gotowy!"
}

function remove_loki() {
	echo "Rozpoczynam deinstalację Lokiego..."
	
	# Zatrzymanie usługi Loki
	sudo systemctl stop loki || true
	sudo systemctl disable loki || true
	sudo systemctl daemon-reload
	
	# Usunięcie pliku usługi systemd
	sudo rm -f /etc/systemd/system/loki.service
	sudo systemctl daemon-reload
	
	# Usunięcie binarki
	sudo rm -f /usr/local/bin/loki
	
	# Usunięcie katalogów i plików konfiguracyjnych
	sudo rm -rf /etc/loki
	sudo rm -rf /var/lib/loki
	
	# Usunięcie użytkownika systemowego
	sudo userdel loki || true
	
	echo "Loki został całkowicie usunięty!"
}

case "$1" in
	--install)
		install_loki
		;;
	--remove)
		remove_loki
		;;
	*)
		echo "Użycie: $0 --install | --remove"
		exit 1
		;;
esac
```

---

## Wyjaśnienie funkcji install_loki()

### Krok 1: Aktualizacja systemu i instalacja wymaganych narzędzi

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y wget unzip curl
```

Instalowane pakiety:
- **wget**: Program do pobierania plików z serwerów HTTP/HTTPS
- **unzip**: Narzędzie do rozpakowywania archiwów ZIP
- **curl**: Narzędzie do transferu danych z/do serwera, używane do testowania API

### Krok 2: Pobranie i instalacja binarki Lokiego

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki
rm -f loki-linux-amd64.zip
```

Operacje:
- Przejście do katalogu tymczasowego `/tmp`
- Pobranie oficjalnej binarki Lokiego z GitHub Releases
- Rozpakowanie archiwum ZIP
- Przeniesienie binarki do `/usr/local/bin/` (standardowa lokalizacja dla aplikacji instalowanych ręcznie)
- Nadanie uprawnień wykonywania
- Usunięcie pliku ZIP po instalacji

### Krok 3: Weryfikacja instalacji

```bash
loki --version
```

Sprawdzenie poprawności instalacji poprzez wyświetlenie wersji Lokiego. Komenda powinna zwrócić informacje o wersji, dacie kompilacji i wersji Go.

### Krok 4: Utworzenie użytkownika systemowego

```bash
sudo useradd --system --no-create-home --shell /bin/false loki || true
```

Parametry:
- **--system**: Tworzy użytkownika systemowego (UID < 1000)
- **--no-create-home**: Nie tworzy katalogu domowego dla użytkownika
- **--shell /bin/false**: Użytkownik nie może się logować interaktywnie
- **|| true**: Kontynuuj nawet jeśli użytkownik już istnieje

### Krok 5: Utworzenie katalogów danych

```bash
sudo mkdir -p /var/lib/loki/{index,cache,chunks,wal,compactor}
sudo mkdir -p /etc/loki
sudo chown -R loki:loki /var/lib/loki
sudo chown -R loki:loki /etc/loki
```

Tworzone katalogi:
- `/etc/loki/` - pliki konfiguracyjne
- `/var/lib/loki/index` - indeksy danych
- `/var/lib/loki/cache` - cache dla indeksów
- `/var/lib/loki/chunks` - dane logów (chunki)
- `/var/lib/loki/wal` - Write-Ahead Log (WAL) dla ingester - **KRYTYCZNE dla niezawodności**
- `/var/lib/loki/compactor` - katalog roboczy kompaktora

Ustawienie właściciela katalogów na użytkownika `loki` zapewnia odpowiednie uprawnienia.

### Krok 6: Utworzenie pliku konfiguracyjnego

Tworzony jest plik `/etc/loki/loki-config.yaml` z podstawową konfiguracją.

#### Kluczowe sekcje konfiguracji:

**Server:**
```yaml
server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: info
```
- Port HTTP: 3100 - API Lokiego
- Port gRPC: 9096 - komunikacja z agentami
- Poziom logowania: info

**Ingester z WAL:**
```yaml
ingester:
  wal:
    enabled: true
    dir: /var/lib/loki/wal
```
- **WAL (Write-Ahead Log)** zapewnia trwałość danych przed zapisem do chunków
- Chroni przed utratą danych w przypadku awarii
- **WAŻNE**: Bez WAL dane mogą zostać utracone podczas restartu

**Schema Config:**
```yaml
schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
```
- **boltdb-shipper**: Wbudowana baza danych dla indeksów
- **filesystem**: Przechowywanie danych na dysku lokalnym
- **schema v11**: Aktualny schemat danych

**Storage Config:**
```yaml
storage_config:
  boltdb_shipper:
    active_index_directory: /var/lib/loki/index
    cache_location: /var/lib/loki/cache
  filesystem:
    directory: /var/lib/loki/chunks
```
Określa lokalizacje przechowywania danych na dysku.

**Limits Config:**
```yaml
limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
```
- Odrzucanie logów starszych niż 7 dni (168h)
- Limit przyjmowania logów: 10 MB/s
- Maksymalny burst: 20 MB

### Krok 7: Utworzenie usługi systemd

```bash
sudo tee /etc/systemd/system/loki.service > /dev/null << 'EOF'
[Unit]
Description=Loki Log Aggregation System
Documentation=https://grafana.com/docs/loki/latest/
After=network.target

[Service]
Type=simple
User=loki
Group=loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

Parametry usługi:
- **Type=simple**: Proces działający w foreground
- **User/Group**: Uruchamianie jako użytkownik `loki`
- **ExecStart**: Komenda uruchamiająca Lokiego z plikiem konfiguracyjnym
- **Restart=on-failure**: Automatyczny restart po awarii
- **RestartSec=10**: Oczekiwanie 10 sekund przed restartem

### Krok 8: Uruchomienie usługi

```bash
sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
```

Operacje:
- **daemon-reload**: Przeładowanie konfiguracji systemd (rozpoznanie nowej usługi)
- **enable**: Włączenie autostartu Lokiego przy uruchomieniu systemu
- **start**: Uruchomienie usługi Loki

Weryfikacja:
- `systemctl status loki` - sprawdzenie statusu usługi
- `curl http://localhost:3100/ready` - sprawdzenie gotowości Lokiego

---

## Wyjaśnienie funkcji remove_loki()

### Zatrzymanie i wyłączenie usługi

```bash
sudo systemctl stop loki || true
sudo systemctl disable loki || true
sudo systemctl daemon-reload
```

Zatrzymanie usługi Loki, wyłączenie autostartu i przeładowanie konfiguracji systemd.

### Usunięcie plików systemowych

```bash
sudo rm -f /etc/systemd/system/loki.service
sudo systemctl daemon-reload
```

Usunięcie pliku usługi systemd i ponowne przeładowanie konfiguracji.

### Usunięcie binarki i danych

```bash
sudo rm -f /usr/local/bin/loki
sudo rm -rf /etc/loki
sudo rm -rf /var/lib/loki
```

Usunięcie:
- Binarki Lokiego z `/usr/local/bin/`
- Wszystkich plików konfiguracyjnych z `/etc/loki/`
- Wszystkich danych z `/var/lib/loki/` (indeksy, chunki, WAL, cache)

### Usunięcie użytkownika systemowego

```bash
sudo userdel loki || true
```

Usunięcie użytkownika systemowego `loki`. Parametr `|| true` zapewnia, że skrypt nie zakończy się błędem, jeśli użytkownik nie istnieje.

---

## Użycie skryptu

### Instalacja Lokiego

```bash
# Nadanie uprawnień wykonywania
chmod +x install_loki.sh

# Uruchomienie instalacji
sudo ./install_loki.sh --install
```

### Weryfikacja instalacji

Po instalacji można zweryfikować działanie Lokiego:

```bash
# Sprawdzenie statusu usługi
sudo systemctl status loki

# Sprawdzenie gotowości Loki
curl http://localhost:3100/ready

# Sprawdzenie metryk
curl http://localhost:3100/metrics | grep loki_build_info

# Przegląd logów
sudo journalctl -u loki -f
```

### Deinstalacja Lokiego

```bash
# Uruchomienie deinstalacji
sudo ./install_loki.sh --remove
```

**UWAGA**: Deinstalacja usuwa wszystkie dane i konfigurację!

---

## Rozwiązywanie problemów

### Loki nie uruchamia się

Sprawdzenie logów:
```bash
sudo journalctl -u loki -n 50
```

Najczęstsze przyczyny:
- Błędna składnia w pliku konfiguracyjnym
- Brak uprawnień do katalogów danych
- Port 3100 lub 9096 jest zajęty

### Sprawdzenie zajętości portów

```bash
# Sprawdzenie portu 3100
sudo lsof -i :3100

# Sprawdzenie portu 9096
sudo lsof -i :9096
```

### Ręczne uruchomienie Lokiego (debug)

```bash
# Uruchomienie w trybie foreground
sudo -u loki /usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
```

### Problemy z uprawnieniami

```bash
# Sprawdzenie właściciela katalogów
ls -la /var/lib/loki
ls -la /etc/loki

# Naprawa uprawnień
sudo chown -R loki:loki /var/lib/loki
sudo chown -R loki:loki /etc/loki
```

---

## Uwagi końcowe

### Bezpieczeństwo

W środowisku produkcyjnym należy rozważyć:
- Włączenie autentykacji (`auth_enabled: true`)
- Konfigurację TLS/SSL dla komunikacji szyfrowanej
- Zabezpieczenie portów 3100 i 9096 przez firewall
- Regularne backupy danych z `/var/lib/loki/`

### Wydajność

Dla większych wdrożeń warto:
- Zwiększyć limity w `limits_config`
- Skonfigurować zewnętrzne storage (S3, GCS)
- Włączyć monitoring metryk Lokiego przez Prometheus
- Skonfigurować retention dla automatycznego czyszczenia starych logów

### Monitorowanie

Loki eksportuje metryki Prometheus na porcie 3100:
```bash
curl http://localhost:3100/metrics
```

Kluczowe metryki do monitorowania:
- `loki_ingester_chunks_created_total` - liczba utworzonych chunków
- `loki_ingester_bytes_received_total` - bajty otrzymanych logów
- `loki_request_duration_seconds` - czas odpowiedzi API

---

## Pliki i katalogi

### Lokalizacje plików:

| Plik/katalog | Opis |
|-------------|------|
| `/usr/local/bin/loki` | Binarka Lokiego |
| `/etc/loki/loki-config.yaml` | Plik konfiguracyjny |
| `/var/lib/loki/index` | Indeksy danych |
| `/var/lib/loki/cache` | Cache indeksów |
| `/var/lib/loki/chunks` | Dane logów (chunki) |
| `/var/lib/loki/wal` | Write-Ahead Log |
| `/var/lib/loki/compactor` | Katalog roboczy kompaktora |
| `/etc/systemd/system/loki.service` | Usługa systemd |

---

## Dodatkowe opcje konfiguracji

### Włączenie retencji (automatyczne usuwanie starych logów)

Edycja `/etc/loki/loki-config.yaml`:

```yaml
limits_config:
  retention_period: 744h  # 31 dni

table_manager:
  retention_deletes_enabled: true
  retention_period: 744h
```

Po zmianie restart usługi:
```bash
sudo systemctl restart loki
```

### Zwiększenie limitów dla większych obciążeń

```yaml
limits_config:
  ingestion_rate_mb: 50
  ingestion_burst_size_mb: 100
  max_streams_per_user: 10000
```

---

## Referencje

- [Dokumentacja Grafana Loki](https://grafana.com/docs/loki/latest/)
- [GitHub Releases Loki](https://github.com/grafana/loki/releases)
- [Przykłady konfiguracji](https://grafana.com/docs/loki/latest/configuration/)
- [Best Practices](https://grafana.com/docs/loki/latest/best-practices/)
