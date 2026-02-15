# Skrypt instalacji i deinstalacji Prometheus z Node Exporter

Automatyczny skrypt do instalacji i usuwania Prometheusa wraz z Node Exporter na systemach Ubuntu/Debian. Skrypt instaluje komponenty z oficjalnych źródeł, konfiguruje usługi systemd i uruchamia monitoring. Konfiguracja pozostaje domyślna - gotowa do modyfikacji podczas szkolenia.

---

## Opis skryptu

Skrypt zawiera dwie główne funkcje:
- **Instalacja Prometheus + Node Exporter** - automatyczna instalacja z oficjalnych źródeł GitHub
- **Deinstalacja** - całkowite usunięcie obu komponentów wraz z danymi i konfiguracją

---

## Pełny skrypt

```bash
#!/bin/bash
set -e

PROMETHEUS_VERSION="2.50.1"
NODE_EXPORTER_VERSION="1.7.0"

function install_prometheus() {
	echo "=== Instalacja Prometheusa ==="
	
	# Krok 1: Tworzenie użytkownika systemowego
	sudo useradd --no-create-home --shell /bin/false prometheus || true
	
	# Krok 2: Tworzenie katalogów
	sudo mkdir -p /etc/prometheus
	sudo mkdir -p /var/lib/prometheus
	
	# Krok 3: Pobranie i instalacja Prometheus
	cd /tmp
	wget https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
	tar xvfz prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
	cd prometheus-${PROMETHEUS_VERSION}.linux-amd64
	
	# Kopiowanie plików binarnych
	sudo cp prometheus /usr/local/bin/
	sudo cp promtool /usr/local/bin/
	
	# Kopiowanie plików konfiguracyjnych
	sudo cp prometheus.yml /etc/prometheus/
	sudo cp -r consoles /etc/prometheus/
	sudo cp -r console_libraries /etc/prometheus/
	
	# Ustawienie uprawnień
	sudo chown -R prometheus:prometheus /etc/prometheus
	sudo chown -R prometheus:prometheus /var/lib/prometheus
	sudo chown prometheus:prometheus /usr/local/bin/prometheus
	sudo chown prometheus:prometheus /usr/local/bin/promtool
	
	# Krok 4: Utworzenie usługi systemd
	sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/var/lib/prometheus/ \\
  --storage.tsdb.retention.time=15d \\
  --web.console.templates=/etc/prometheus/consoles \\
  --web.console.libraries=/etc/prometheus/console_libraries \\
  --web.listen-address=0.0.0.0:9090 \\
  --web.enable-lifecycle

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
	
	# Krok 5: Uruchomienie Prometheus
	sudo systemctl daemon-reload
	sudo systemctl enable prometheus
	sudo systemctl start prometheus
	
	# Czyszczenie
	cd /tmp
	rm -rf prometheus-${PROMETHEUS_VERSION}.linux-amd64*
	
	echo "✓ Prometheus zainstalowany i uruchomiony!"
	echo "  URL: http://localhost:9090"
}

function install_node_exporter() {
	echo "=== Instalacja Node Exporter ==="
	
	# Krok 1: Tworzenie użytkownika systemowego
	sudo useradd --no-create-home --shell /bin/false node_exporter || true
	
	# Krok 2: Pobranie i instalacja Node Exporter
	cd /tmp
	wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
	tar xvfz node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
	
	# Kopiowanie pliku binarnego
	sudo cp node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
	sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
	
	# Krok 3: Utworzenie usługi systemd
	sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
After=network-online.target
Wants=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \\
  --web.listen-address=:9100

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
	
	# Krok 4: Uruchomienie Node Exporter
	sudo systemctl daemon-reload
	sudo systemctl enable node_exporter
	sudo systemctl start node_exporter
	
	# Krok 5: Dodanie Node Exporter do konfiguracji Prometheus
	if ! grep -q "node_exporter" /etc/prometheus/prometheus.yml; then
		sudo tee -a /etc/prometheus/prometheus.yml > /dev/null <<EOF

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
EOF
		# Przeładowanie konfiguracji Prometheus
		sudo systemctl restart prometheus
	fi
	
	# Czyszczenie
	cd /tmp
	rm -rf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64*
	
	echo "✓ Node Exporter zainstalowany i uruchomiony!"
	echo "  URL: http://localhost:9100/metrics"
}

function remove_prometheus() {
	echo "=== Deinstalacja Prometheus i Node Exporter ==="
	
	# Zatrzymanie usług
	sudo systemctl stop prometheus || true
	sudo systemctl stop node_exporter || true
	sudo systemctl disable prometheus || true
	sudo systemctl disable node_exporter || true
	
	# Usunięcie plików usług systemd
	sudo rm -f /etc/systemd/system/prometheus.service
	sudo rm -f /etc/systemd/system/node_exporter.service
	sudo systemctl daemon-reload
	
	# Usunięcie plików binarnych
	sudo rm -f /usr/local/bin/prometheus
	sudo rm -f /usr/local/bin/promtool
	sudo rm -f /usr/local/bin/node_exporter
	
	# Usunięcie katalogów konfiguracyjnych i danych
	sudo rm -rf /etc/prometheus
	sudo rm -rf /var/lib/prometheus
	
	# Usunięcie użytkowników systemowych
	sudo userdel prometheus || true
	sudo userdel node_exporter || true
	
	echo "✓ Prometheus i Node Exporter zostały całkowicie usunięte!"
}

case "$1" in
	--install)
		install_prometheus
		install_node_exporter
		echo ""
		echo "=== Status instalacji ==="
		sudo systemctl status prometheus --no-pager -l
		sudo systemctl status node_exporter --no-pager -l
		echo ""
		echo "=== Dostępne endpointy ==="
		echo "Prometheus Web UI: http://localhost:9090"
		echo "Node Exporter:     http://localhost:9100/metrics"
		;;
	--remove)
		remove_prometheus
		;;
	*)
		echo "Użycie: $0 --install | --remove"
		exit 1
		;;
esac
```

---

## Wyjaśnienie funkcji install_prometheus()

### Krok 1: Tworzenie użytkownika systemowego

```bash
sudo useradd --no-create-home --shell /bin/false prometheus || true
```

**Wyjaśnienie:**
- **--no-create-home**: Nie tworzy katalogu domowego dla użytkownika
- **--shell /bin/false**: Blokuje możliwość logowania się tym użytkownikiem
- **|| true**: Kontynuuje wykonanie nawet jeśli użytkownik już istnieje

**Uwaga:** Dedykowany użytkownik systemowy zwiększa bezpieczeństwo - Prometheus działa z ograniczonymi uprawnieniami.

### Krok 2: Tworzenie katalogów

```bash
sudo mkdir -p /etc/prometheus        # Pliki konfiguracyjne
sudo mkdir -p /var/lib/prometheus    # Baza danych TSDB
```

Struktura katalogów:
- **/etc/prometheus**: Konfiguracja, reguły alertów, konsole
- **/var/lib/prometheus**: Dane czasowe (time series database)

### Krok 3: Pobieranie i instalacja

```bash
wget https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
tar xvfz prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
```

Pobiera oficjalną, przetestowaną wersję z GitHub Releases.

**Kopiowanie plików:**
```bash
sudo cp prometheus /usr/local/bin/        # Główny serwer Prometheus
sudo cp promtool /usr/local/bin/          # Narzędzie do walidacji konfiguracji
sudo cp prometheus.yml /etc/prometheus/   # Domyślna konfiguracja
sudo cp -r consoles /etc/prometheus/      # Szablony konsol
sudo cp -r console_libraries /etc/prometheus/  # Biblioteki konsol
```

### Krok 4: Usługa systemd

Plik `/etc/systemd/system/prometheus.service` definiuje jak systemd zarządza Prometheusem:

```ini
[Service]
User=prometheus                          # Uruchamianie jako użytkownik prometheus
Group=prometheus
Type=simple                              # Proces działa na pierwszym planie
ExecStart=/usr/local/bin/prometheus \    # Komenda uruchomienia
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=15d \    # Retencja danych - 15 dni
  --web.listen-address=0.0.0.0:9090 \    # Nasłuchiwanie na wszystkich interfejsach
  --web.enable-lifecycle                  # Włączenie API do przeładowania konfiguracji
```

**Kluczowe parametry:**
- **--config.file**: Ścieżka do pliku konfiguracyjnego
- **--storage.tsdb.path**: Lokalizacja bazy danych metryk
- **--storage.tsdb.retention.time=15d**: Metryki starsze niż 15 dni są automatycznie usuwane
- **--web.enable-lifecycle**: Umożliwia przeładowanie konfiguracji przez API (`POST /-/reload`)

### Krok 5: Uruchomienie usługi

```bash
sudo systemctl daemon-reload       # Przeładowanie konfiguracji systemd
sudo systemctl enable prometheus   # Autostart przy boocie
sudo systemctl start prometheus    # Natychmiastowe uruchomienie
```

---

## Wyjaśnienie funkcji install_node_exporter()

### Tworzenie użytkownika

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter || true
```

Dedykowany użytkownik dla Node Exporter (zgodnie z zasadą separacji uprawnień).

### Instalacja Node Exporter

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo cp node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
```

Node Exporter to pojedynczy plik binarny bez zewnętrznych zależności.

### Usługa systemd

```ini
[Service]
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=:9100           # Port domyślny dla Node Exporter
```

Node Exporter domyślnie włącza wszystkie kollektory (CPU, RAM, dysk, sieć, itp.).

### Automatyczna konfiguracja Prometheus

```bash
if ! grep -q "node_exporter" /etc/prometheus/prometheus.yml; then
	sudo tee -a /etc/prometheus/prometheus.yml > /dev/null <<EOF

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
EOF
	sudo systemctl restart prometheus
fi
```

**Wyjaśnienie:**
- Sprawdza czy konfiguracja już nie zawiera `node_exporter`
- Dodaje nowy job do `scrape_configs`
- Restartuje Prometheus aby załadować nową konfigurację

**Rezultat:** Prometheus automatycznie zaczyna zbierać metryki systemowe z Node Exporter.

---

## Wyjaśnienie funkcji remove_prometheus()

### Zatrzymanie usług

```bash
sudo systemctl stop prometheus || true
sudo systemctl stop node_exporter || true
sudo systemctl disable prometheus || true
sudo systemctl disable node_exporter || true
```

Zatrzymuje usługi i wyłącza autostart.

### Usunięcie plików

```bash
# Usługi systemd
sudo rm -f /etc/systemd/system/prometheus.service
sudo rm -f /etc/systemd/system/node_exporter.service

# Pliki binarne
sudo rm -f /usr/local/bin/prometheus
sudo rm -f /usr/local/bin/promtool
sudo rm -f /usr/local/bin/node_exporter

# Konfiguracja i dane
sudo rm -rf /etc/prometheus          # UWAGA: Usuwa całą konfigurację!
sudo rm -rf /var/lib/prometheus      # UWAGA: Usuwa wszystkie zebrane metryki!

# Użytkownicy systemowi
sudo userdel prometheus || true
sudo userdel node_exporter || true
```

**WAŻNE**: Operacja nieodwracalna - usuwa wszystkie dane i konfigurację!

---

## Użycie skryptu

### Instalacja Prometheus + Node Exporter

```bash
# Nadanie uprawnień wykonywania
chmod +x install-prometheus.sh

# Uruchomienie instalacji
./install-prometheus.sh --install
```

**Po instalacji dostępne są:**
- Prometheus Web UI: http://localhost:9090
- Node Exporter metrics: http://localhost:9100/metrics

### Weryfikacja instalacji

```bash
# Status usług
sudo systemctl status prometheus
sudo systemctl status node_exporter

# Sprawdzenie wersji
prometheus --version
node_exporter --version

# Test endpointów
curl http://localhost:9090/-/healthy
curl http://localhost:9100/metrics | head -20

# Sprawdzenie targets w Prometheus
curl http://localhost:9090/api/v1/targets | jq
```

### Sprawdzenie w Web UI

Otwórz http://localhost:9090 w przeglądarce:

1. Przejdź do **Status → Targets**
2. Powinieneś zobaczyć dwa aktywne cele:
   - `prometheus` (localhost:9090) - UP
   - `node_exporter` (localhost:9100) - UP

### Przykładowe zapytanie PromQL

W Web UI (http://localhost:9090/graph) wpisz:

```PromQL
# Użycie CPU w procentach
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Zużycie pamięci RAM
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Ilość dostępnej pamięci
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

### Deinstalacja

```bash
./install-prometheus.sh --remove
```

**Uwaga:** Przed deinstalacją warto wykonać backup danych!

### Backup przed usunięciem

```bash
# Backup danych metryk
sudo tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz /var/lib/prometheus/

# Backup konfiguracji
sudo tar -czf prometheus-config-backup-$(date +%Y%m%d).tar.gz /etc/prometheus/
```

---

## Wymagania systemowe

- **System operacyjny**: Ubuntu 20.04+ / Debian 11+
- **Uprawnienia**: root lub sudo
- **Dostęp do internetu**: Wymagany do pobrania pakietów z GitHub
- **Wolne miejsce**: 
  - Minimum 100 MB dla binarek
  - Minimum 1 GB dla danych metryk (zależnie od retencji)
- **Porty**: 9090 (Prometheus), 9100 (Node Exporter)

---

## Domyślna konfiguracja

### Plik prometheus.yml (po instalacji)

```yaml
# Konfiguracja globalna
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Konfiguracja Alertmanagera (wyłączona)
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Pliki z regułami (brak)
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# Konfiguracja zbierania metryk
scrape_configs:
  # Monitorowanie samego Prometheusa
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Node Exporter (dodawany przez skrypt)
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

**Uwaga:** To domyślna konfiguracja gotowa do modyfikacji podczas szkolenia z uczestnikami.

---

## Najlepsze praktyki

### 1. **Weryfikacja po instalacji**

```bash
# Sprawdź czy usługi działają
sudo systemctl is-active prometheus
sudo systemctl is-active node_exporter

# Sprawdź logi w przypadku problemów
sudo journalctl -u prometheus -n 50
sudo journalctl -u node_exporter -n 50

# Sprawdź metryki
curl -s http://localhost:9090/metrics | grep -i "prometheus_build_info"
```

### 2. **Sprawdzenie portów**

```bash
# Sprawdź nasłuchujące porty
sudo ss -tulpn | grep -E "9090|9100"

# Sprawdź połączenia
sudo netstat -tulpn | grep prometheus
```

### 3. **Testowanie konfiguracji**

```bash
# Walidacja pliku konfiguracyjnego przed restartem
promtool check config /etc/prometheus/prometheus.yml

# Sprawdzenie reguł (jeśli są zdefiniowane)
promtool check rules /etc/prometheus/rules/*.yml
```

---

## Troubleshooting

### Problem: Usługa Prometheus nie startuje

**Diagnoza:**
```bash
# Sprawdź logi
sudo journalctl -u prometheus -xe

# Sprawdź status
sudo systemctl status prometheus
```

**Częste przyczyny:**
- Błąd w pliku konfiguracyjnym: `promtool check config /etc/prometheus/prometheus.yml`
- Brak uprawnień do katalogów: `sudo chown -R prometheus:prometheus /var/lib/prometheus`
- Port zajęty: `sudo lsof -i :9090`

### Problem: Node Exporter nie zbiera metryk

**Diagnoza:**
```bash
# Sprawdź czy endpoint odpowiada
curl http://localhost:9100/metrics

# Sprawdź czy Prometheus widzi target
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.job=="node_exporter")'
```

**Rozwiązanie:**
```bash
# Restart usługi
sudo systemctl restart node_exporter

# Sprawdź konfigurację firewall
sudo ufw status
sudo ufw allow 9100/tcp
```

### Problem: Brak danych w Web UI

**Przyczyny:**
- Target jest w stanie DOWN
- Błędna konfiguracja scrape_config
- Problemy sieciowe

**Sprawdzenie:**
```bash
# Zobacz status targets
curl http://localhost:9090/api/v1/targets

# Sprawdź czy są jakieś metryki
curl http://localhost:9090/api/v1/label/__name__/values | jq
```

### Problem: Prometheus zużywa dużo miejsca

```bash
# Sprawdź rozmiar danych
du -sh /var/lib/prometheus

# Zmień retencję (w pliku systemd service)
sudo nano /etc/systemd/system/prometheus.service
# Zmień: --storage.tsdb.retention.time=15d na np. 7d

sudo systemctl daemon-reload
sudo systemctl restart prometheus
```

---

## Co dalej po instalacji?

### Podczas szkolenia można:

1. **Modyfikować konfigurację Prometheus** (`/etc/prometheus/prometheus.yml`)
   - Dodawać nowe targets
   - Konfigurować interwały scrape
   - Dodawać etykiety

2. **Dodać kolejne eksportery**
   - MySQL Exporter
   - Blackbox Exporter
   - Custom exporters

3. **Skonfigurować Alertmanager**
   - Dodać reguły alertów
   - Skonfigurować powiadomienia

4. **Połączyć z Grafaną**
   - Dodać Prometheus jako Data Source
   - Tworzyć dashboardy

5. **Eksperymentować z PromQL**
   - Testować zapytania
   - Tworzyć recording rules

---

## Kompatybilność wersji

Skrypt został przetestowany z:
- **Prometheus**: 2.50.1
- **Node Exporter**: 1.7.0
- **Ubuntu**: 22.04 LTS, 24.04 LTS
- **Debian**: 11, 12

---

## Podsumowanie

Skrypt automatyzuje proces instalacji stack'u monitoringu Prometheus:
- ✅ Instalacja Prometheus z oficjalnych źródeł
- ✅ Instalacja Node Exporter
- ✅ Automatyczna konfiguracja usług systemd
- ✅ Automatyczne połączenie Prometheus z Node Exporter
- ✅ Gotowy do użycia po instalacji
- ✅ Możliwość całkowitej deinstalacji

Idealne rozwiązanie do szybkiego przygotowania środowiska szkoleniowego.
