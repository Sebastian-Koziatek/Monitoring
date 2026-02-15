# Skrypt instalacji InfluxDB 2.x i Telegraf

## Opis

Automatyczny skrypt instalacyjny dla **InfluxDB 2.x** i **Telegraf** na systemie Ubuntu 24.04 LTS.

Skrypt wykonuje:
- Weryfikację systemu i wymagań
- Instalację zależności
- Dodanie repozytoriów InfluxData
- Instalację InfluxDB 2.x
- Instalację Telegraf
- Konfigurację podstawową
- Instrukcje dalszej konfiguracji

## Wymagania

- Ubuntu 24.04 LTS (lub nowszy)
- Uprawnienia root (sudo)
- Minimum 2 GB RAM
- Minimum 5 GB wolnego miejsca na dysku
- Dostęp do internetu

## Instalacja

### Metoda 1: Bezpośrednie uruchomienie

```bash
#!/bin/bash

################################################################################
# Skrypt instalacji InfluxDB 2.x i Telegraf dla Ubuntu 24.04
# Data: $(date +%Y-%m-%d)
################################################################################

set -e  # Zatrzymaj skrypt przy błędzie
set -u  # Zatrzymaj przy użyciu niezdefiniowanych zmiennych

# Kolory dla outputu
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Funkcje pomocnicze
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

check_root() {
    if [ "$EUID" -ne 0 ]; then 
        log_error "Ten skrypt musi być uruchomiony jako root (sudo)"
        exit 1
    fi
}

check_system() {
    log_info "Sprawdzanie systemu..."
    
    # Sprawdź czy to Ubuntu
    if [ ! -f /etc/os-release ]; then
        log_error "Nie można określić systemu operacyjnego"
        exit 1
    fi
    
    . /etc/os-release
    
    if [ "$ID" != "ubuntu" ]; then
        log_error "Ten skrypt jest przeznaczony dla Ubuntu. Wykryto: $ID"
        exit 1
    fi
    
    log_info "System: $PRETTY_NAME"
    log_info "Architektura: $(uname -m)"
    
    # Sprawdź dostępną pamięć
    available_mem=$(free -m | awk '/^Mem:/{print $7}')
    if [ "$available_mem" -lt 500 ]; then
        log_warn "Mało dostępnej pamięci RAM: ${available_mem}MB. Zalecane minimum: 500MB"
    fi
    
    # Sprawdź dostępne miejsce na dysku
    available_space=$(df -BG / | awk 'NR==2 {print $4}' | sed 's/G//')
    if [ "$available_space" -lt 5 ]; then
        log_warn "Mało miejsca na dysku: ${available_space}GB. Zalecane minimum: 5GB"
    fi
}

install_dependencies() {
    log_info "Instalowanie zależności..."
    
    apt-get update -qq
    apt-get install -y \
        wget \
        gnupg2 \
        curl \
        apt-transport-https \
        ca-certificates \
        software-properties-common
    
    log_info "Zależności zainstalowane"
}

install_influxdb() {
    log_info "=== Instalacja InfluxDB 2.x ==="
    
    # Sprawdź czy InfluxDB już istnieje
    if command -v influxd &> /dev/null; then
        log_warn "InfluxDB jest już zainstalowany"
        influxd version
        read -p "Czy chcesz kontynuować instalację? (t/n): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Tt]$ ]]; then
            return 0
        fi
    fi
    
    # Dodaj klucz GPG InfluxData
    log_info "Dodawanie klucza GPG InfluxData (zaktualizowany 2026)..."
    
    # Użyj nowego klucza (zaktualizowanego w styczniu 2026)
    curl -fsSL https://repos.influxdata.com/influxdata-archive.key | \
        gpg --dearmor --yes -o /etc/apt/trusted.gpg.d/influxdata-archive.gpg
    
    # Ustaw odpowiednie uprawnienia
    chmod 644 /etc/apt/trusted.gpg.d/influxdata-archive.gpg
    
    log_info "Klucz GPG został pomyślnie zainstalowany"
    
    # Dodaj repozytorium InfluxData
    log_info "Dodawanie repozytorium InfluxData..."
    echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main" | tee /etc/apt/sources.list.d/influxdata.list
    
    # Zaktualizuj listę pakietów
    apt-get update -qq
    
    # Instaluj InfluxDB
    log_info "Instalowanie InfluxDB..."
    apt-get install -y influxdb2 influxdb2-cli
    
    # Włącz i uruchom InfluxDB
    log_info "Uruchamianie usługi InfluxDB..."
    systemctl enable influxdb
    systemctl start influxdb
    
    # Sprawdź status
    sleep 3
    if systemctl is-active --quiet influxdb; then
        log_info "✓ InfluxDB uruchomiony pomyślnie"
        influxd version
    else
        log_error "Nie udało się uruchomić InfluxDB"
        systemctl status influxdb --no-pager
        exit 1
    fi
    
    log_info "InfluxDB jest dostępny na http://localhost:8086"
}

install_telegraf() {
    log_info "=== Instalacja Telegraf ==="
    
    # Sprawdź czy Telegraf już istnieje
    if command -v telegraf &> /dev/null; then
        log_warn "Telegraf jest już zainstalowany"
        telegraf version
        read -p "Czy chcesz kontynuować instalację? (t/n): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Tt]$ ]]; then
            return 0
        fi
    fi
    
    # Repozytorium InfluxData zostało już dodane przy InfluxDB
    log_info "Instalowanie Telegraf..."
    apt-get install -y telegraf
    
    # Backup oryginalnej konfiguracji
    if [ -f /etc/telegraf/telegraf.conf ]; then
        log_info "Tworzenie kopii zapasowej konfiguracji..."
        cp /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.backup-$(date +%Y%m%d-%H%M%S)
    fi
    
    # Generuj nową podstawową konfigurację
    log_info "Generowanie podstawowej konfiguracji Telegraf..."
    telegraf --sample-config > /etc/telegraf/telegraf.conf.sample
    
    # Tworzenie prostej konfiguracji dla InfluxDB v2
    cat > /etc/telegraf/telegraf.conf << 'EOF'
# Telegraf Configuration

[global_tags]
  # Możesz dodać własne tagi tutaj
  # environment = "production"

[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
  hostname = ""
  omit_hostname = false

###############################################################################
#                            OUTPUT PLUGINS                                   #
###############################################################################

# Konfiguracja output do InfluxDB 2.x
[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  
  ## Token dla autoryzacji (ustaw po konfiguracji InfluxDB)
  token = "$INFLUX_TOKEN"
  
  ## Organization (ustaw po konfiguracji InfluxDB)
  organization = "myorg"
  
  ## Bucket (ustaw po konfiguracji InfluxDB)
  bucket = "telegraf"
  
  ## Timeout dla żądań HTTP
  timeout = "5s"

###############################################################################
#                            INPUT PLUGINS                                    #
###############################################################################

# Zbieranie metryk CPU
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false

# Zbieranie metryk dysku
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]

# Zbieranie metryk I/O dysku
[[inputs.diskio]]

# Zbieranie metryk pamięci
[[inputs.mem]]

# Zbieranie metryk procesów
[[inputs.processes]]

# Zbieranie metryk swap
[[inputs.swap]]

# Zbieranie metryk systemu
[[inputs.system]]

# Zbieranie metryk sieci
[[inputs.net]]
  interfaces = ["eth*", "enp*", "ens*"]

# Zbieranie statystyk netstat
[[inputs.netstat]]

# Zbieranie informacji o kernelu
[[inputs.kernel]]

# Zbieranie obciążenia systemu
[[inputs.system]]
EOF
    
    log_info "Podstawowa konfiguracja Telegraf została utworzona"
    log_warn "UWAGA: Musisz skonfigurować token InfluxDB w /etc/telegraf/telegraf.conf"
    log_warn "Zastąp \$INFLUX_TOKEN prawdziwym tokenem po skonfigurowaniu InfluxDB"
    
    # Nie uruchamiaj jeszcze Telegraf (wymaga konfiguracji tokenu)
    log_info "Włączanie Telegraf do autostartu (nie uruchamianie jeszcze)..."
    systemctl enable telegraf
    
    log_info "✓ Telegraf zainstalowany"
    telegraf version
}

configure_influxdb() {
    log_info "=== Konfiguracja początkowa InfluxDB ==="
    
    log_warn "Konfiguracja InfluxDB wymaga ręcznej inicjalizacji"
    echo ""
    echo "Aby skonfigurować InfluxDB, wykonaj jedną z następujących metod:"
    echo ""
    echo "1. Interfejs webowy:"
    echo "   Otwórz przeglądarkę: http://localhost:8086"
    echo "   Utwórz użytkownika, organizację i bucket"
    echo ""
    echo "2. Wiersz poleceń:"
    echo "   influx setup \\"
    echo "     --username admin \\"
    echo "     --password 'TwojeHaslo123!' \\"
    echo "     --org myorg \\"
    echo "     --bucket telegraf \\"
    echo "     --retention 168h \\"
    echo "     --force"
    echo ""
    echo "Po utworzeniu konfiguracji:"
    echo "3. Wygeneruj token:"
    echo "   influx auth create --org myorg --all-access"
    echo ""
    echo "4. Zaktualizuj konfigurację Telegraf:"
    echo "   sudo nano /etc/telegraf/telegraf.conf"
    echo "   (zastąp \$INFLUX_TOKEN wygenerowanym tokenem)"
    echo ""
    echo "5. Uruchom Telegraf:"
    echo "   sudo systemctl start telegraf"
    echo "   sudo systemctl status telegraf"
    echo ""
}

show_summary() {
    echo ""
    echo "=========================================="
    log_info "PODSUMOWANIE INSTALACJI"
    echo "=========================================="
    echo ""
    
    # Status InfluxDB
    if systemctl is-active --quiet influxdb; then
        log_info "✓ InfluxDB: URUCHOMIONY"
        echo "  URL: http://localhost:8086"
    else
        log_error "✗ InfluxDB: ZATRZYMANY"
    fi
    
    # Status Telegraf
    if systemctl is-active --quiet telegraf; then
        log_info "✓ Telegraf: URUCHOMIONY"
    else
        log_warn "○ Telegraf: ZATRZYMANY (wymaga konfiguracji)"
    fi
    
    echo ""
    echo "NASTĘPNE KROKI:"
    echo "1. Skonfiguruj InfluxDB (zobacz instrukcje powyżej)"
    echo "2. Zaktualizuj token w /etc/telegraf/telegraf.conf"
    echo "3. Uruchom Telegraf: sudo systemctl start telegraf"
    echo ""
    echo "PRZYDATNE KOMENDY:"
    echo "  - Status InfluxDB: sudo systemctl status influxdb"
    echo "  - Status Telegraf: sudo systemctl status telegraf"
    echo "  - Logi InfluxDB: sudo journalctl -u influxdb -f"
    echo "  - Logi Telegraf: sudo journalctl -u telegraf -f"
    echo "  - Test config Telegraf: telegraf --test --config /etc/telegraf/telegraf.conf"
    echo ""
    echo "=========================================="
}

# GŁÓWNY PROGRAM
main() {
    clear
    echo "=========================================="
    echo "  Instalator InfluxDB 2.x + Telegraf"
    echo "  Ubuntu 24.04 LTS"
    echo "=========================================="
    echo ""
    
    check_root
    check_system
    
    echo ""
    read -p "Czy kontynuować instalację? (t/n): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Tt]$ ]]; then
        log_info "Instalacja anulowana"
        exit 0
    fi
    
    install_dependencies
    install_influxdb
    install_telegraf
    configure_influxdb
    show_summary
    
    log_info "Instalacja zakończona!"
}

# Uruchom główny program
main
```

### Metoda 2: Zapisanie do pliku i uruchomienie

1. **Utwórz plik skryptu:**
   ```bash
   nano install_influxdb.sh
   ```

2. **Wklej zawartość skryptu** (powyżej)

3. **Nadaj uprawnienia wykonywania:**
   ```bash
   chmod +x install_influxdb.sh
   ```

4. **Uruchom skrypt:**
   ```bash
   sudo ./install_influxdb.sh
   ```
