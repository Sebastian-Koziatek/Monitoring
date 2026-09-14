# Skrypt instalacji Zabbix 7.0 LTS Server

## Opis

Automatyczny skrypt instalacyjny dla **Zabbix 7.0 LTS Server + Frontend + Agent** na systemie Ubuntu 24.04 LTS z MariaDB i Apache.

Skrypt wykonuje:
- Instalację i konfigurację MariaDB Server
- Zabezpieczenie MariaDB
- Dodanie repozytorium Zabbix 7.0 LTS
- Instalację Zabbix Server, Frontend, Agent
- Instalację polskiej lokalizacji
- Utworzenie i import schematu bazy danych
- Konfigurację PHP i Apache
- Uruchomienie wszystkich usług

## Wymagania

- Ubuntu 24.04 LTS
- Uprawnienia root (sudo)
- Minimum 2 GB RAM (zalecane 4 GB+)
- Minimum 10 GB wolnego miejsca na dysku
- Dostęp do internetu

## Instalacja

### Metoda 1: Bezpośrednie uruchomienie

```bash
#!/bin/bash

################################################################################
# Skrypt instalacji Zabbix 7.0 LTS Server + Frontend + Agent
# Ubuntu 24.04 LTS + MariaDB + Apache
################################################################################

set -e  # Zatrzymaj skrypt przy błędzie
set -u  # Zatrzymaj przy użyciu niezdefiniowanych zmiennych

# Kolory dla outputu
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Domyślne hasła (zmień przed użyciem w produkcji!)
DB_ROOT_PASSWORD="ZabbixRoot123!"
DB_ZABBIX_PASSWORD="ZabbixDB123!"

# Ustawienie trybu nieinteraktywnego dla apt
export DEBIAN_FRONTEND=noninteractive

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

log_step() {
    echo -e "\n${BLUE}==== $1 ====${NC}\n"
}

wait_for_apt_lock() {
    log_info "Sprawdzanie dostępności menedżera pakietów..."
    
    local max_wait=300  # 5 minut timeout
    local waited=0
    
    while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 || \
          fuser /var/lib/dpkg/lock >/dev/null 2>&1 || \
          fuser /var/cache/apt/archives/lock >/dev/null 2>&1; do
        
        if [ $waited -eq 0 ]; then
            log_warn "Menedżer pakietów jest zajęty, czekam na zwolnienie..."
        fi
        
        if [ $waited -ge $max_wait ]; then
            log_error "Timeout: Menedżer pakietów nadal zajęty po ${max_wait}s"
            log_info "Próbuję zatrzymać automatyczne aktualizacje..."
            systemctl stop unattended-upgrades || true
            systemctl stop apt-daily.timer || true
            systemctl stop apt-daily-upgrade.timer || true
            killall -9 apt apt-get dpkg 2>/dev/null || true
            sleep 5
            break
        fi
        
        sleep 2
        waited=$((waited + 2))
    done
    
    if [ $waited -gt 0 ]; then
        log_info "✓ Menedżer pakietów dostępny (czekano ${waited}s)"
    fi
}

stop_auto_updates() {
    log_info "Zatrzymywanie automatycznych aktualizacji..."
    
    # Zatrzymaj usługi automatycznych aktualizacji
    systemctl stop unattended-upgrades 2>/dev/null || true
    systemctl stop apt-daily.timer 2>/dev/null || true
    systemctl stop apt-daily-upgrade.timer 2>/dev/null || true
    
    # Zabij ewentualne wiszące procesy
    pkill -9 -f unattended-upgrade 2>/dev/null || true
    
    log_info "✓ Automatyczne aktualizacje zatrzymane"
}

check_root() {
    if [ "$EUID" -ne 0 ]; then 
        log_error "Ten skrypt musi być uruchomiony jako root (sudo)"
        exit 1
    fi
}

check_system() {
    log_info "Sprawdzanie systemu..."
    
    if [ ! -f /etc/os-release ]; then
        log_error "Nie można określić systemu operacyjnego"
        exit 1
    fi
    
    . /etc/os-release
    
    if [ "$ID" != "ubuntu" ]; then
        log_error "Ten skrypt jest przeznaczony dla Ubuntu. Wykryto: $ID"
        exit 1
    fi
    
    if [ "$VERSION_ID" != "24.04" ]; then
        log_warn "Skrypt jest zoptymalizowany dla Ubuntu 24.04, wykryto: $VERSION_ID"
    fi
    
    log_info "System: $PRETTY_NAME"
}

install_mariadb() {
    log_step "KROK 1: Instalacja MariaDB Server"
    
    if command -v mysql &> /dev/null; then
        log_warn "MySQL/MariaDB jest już zainstalowany"
        mysql --version
    else
        log_info "Instalowanie MariaDB Server..."
        wait_for_apt_lock
        apt-get update -qq
        wait_for_apt_lock
        apt-get install -y mariadb-server mariadb-client
        log_info "✓ MariaDB zainstalowany"
    fi
}

configure_mariadb() {
    log_step "KROK 2: Konfiguracja MariaDB"
    
    log_info "Włączanie i uruchamianie usługi MariaDB..."
    systemctl enable mariadb
    systemctl start mariadb
    
    sleep 2
    
    if systemctl is-active --quiet mariadb; then
        log_info "✓ MariaDB uruchomiony pomyślnie"
    else
        log_error "Nie udało się uruchomić MariaDB"
        systemctl status mariadb --no-pager
        exit 1
    fi
}

secure_mariadb() {
    log_step "KROK 3: Zabezpieczanie MariaDB"
    
    log_info "Automatyczna konfiguracja zabezpieczeń MariaDB..."
    
    # Sprawdź czy można zalogować się bez hasła
    if mysql -u root -e "SELECT 1;" &> /dev/null; then
        log_info "Konfigurowanie zabezpieczeń (pierwsze uruchomienie)..."
        # Ustawienie hasła root i podstawowe zabezpieczenia
        mysql -u root <<-EOF
            -- Ustawienie hasła root
            ALTER USER 'root'@'localhost' IDENTIFIED BY '${DB_ROOT_PASSWORD}';
            
            -- Usunięcie anonimowych użytkowników
            DELETE FROM mysql.user WHERE User='';
            
            -- Wyłączenie zdalnego logowania root
            DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
            
            -- Usunięcie testowej bazy danych
            DROP DATABASE IF EXISTS test;
            DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
            
            -- Przeładowanie uprawnień
            FLUSH PRIVILEGES;
EOF
        log_info "✓ MariaDB zabezpieczony"
        log_warn "Hasło root MariaDB: ${DB_ROOT_PASSWORD}"
    elif mysql -u root -p"${DB_ROOT_PASSWORD}" -e "SELECT 1;" &> /dev/null; then
        log_warn "MariaDB jest już zabezpieczony (hasło już ustawione)"
        log_info "✓ Weryfikacja hasła root - OK"
    else
        log_error "Nie można zalogować się do MariaDB!"
        log_error "Sprawdź hasło root lub zresetuj MariaDB"
        exit 1
    fi
}

install_zabbix_repo() {
    log_step "KROK 4: Dodawanie repozytorium Zabbix"
    
    log_info "Pobieranie pakietu repozytorium Zabbix 7.0 LTS..."
    
    cd /tmp
    wget -q https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu24.04_all.deb
    
    log_info "Instalowanie pakietu repozytorium..."
    wait_for_apt_lock
    dpkg -i zabbix-release_7.0-1+ubuntu24.04_all.deb
    
    log_info "Aktualizacja listy pakietów..."
    wait_for_apt_lock
    apt-get update -qq
    
    log_info "✓ Repozytorium Zabbix dodane"
}

install_zabbix_packages() {
    log_step "KROK 5: Instalacja pakietów Zabbix"
    
    log_info "Instalowanie Zabbix Server, Frontend, Agent..."
    wait_for_apt_lock
    apt-get install -y \
        zabbix-server-mysql \
        zabbix-frontend-php \
        zabbix-apache-conf \
        zabbix-sql-scripts \
        zabbix-agent
    
    log_info "✓ Pakiety Zabbix zainstalowane"
}

install_polish_locale() {
    log_step "KROK 6: Instalacja polskiej lokalizacji"
    
    log_info "Sprawdzanie polskiej lokalizacji..."
    
    if locale -a | grep -q "pl_PL.utf8"; then
        log_info "✓ Polska lokalizacja już zainstalowana"
    else
        log_info "Instalowanie pakietu językowego dla języka polskiego..."
        wait_for_apt_lock
        apt-get install -y language-pack-pl
        
        log_info "✓ Polski pakiet językowy zainstalowany"
        
        # Sprawdzenie po instalacji
        if locale -a | grep -q "pl_PL.utf8"; then
            log_info "✓ Polska lokalizacja (pl_PL.utf8) jest dostępna"
        else
            log_warn "Nie udało się zweryfikować polskiej lokalizacji"
        fi
    fi
}

create_zabbix_database() {
    log_step "KROK 7: Tworzenie bazy danych Zabbix"
    
    log_info "Tworzenie bazy danych i użytkownika..."
    
    mysql -uroot -p"${DB_ROOT_PASSWORD}" <<-EOF
        CREATE DATABASE IF NOT EXISTS zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
        CREATE USER IF NOT EXISTS 'zabbix'@'localhost' IDENTIFIED BY '${DB_ZABBIX_PASSWORD}';
        GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
        SET GLOBAL log_bin_trust_function_creators = 1;
        FLUSH PRIVILEGES;
EOF
    
    log_info "✓ Baza danych i użytkownik Zabbix utworzony"
    log_warn "Hasło użytkownika zabbix: ${DB_ZABBIX_PASSWORD}"
}

import_zabbix_schema() {
    log_step "KROK 8: Import schematu bazy danych Zabbix"
    
    log_info "Importowanie schematu (może to potrwać kilka minut)..."
    
    if [ -f /usr/share/zabbix-sql-scripts/mysql/server.sql.gz ]; then
        zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
            mysql --default-character-set=utf8mb4 -uzabbix -p"${DB_ZABBIX_PASSWORD}" zabbix
        log_info "✓ Schemat zaimportowany"
    else
        log_error "Nie znaleziono pliku schematu: /usr/share/zabbix-sql-scripts/mysql/server.sql.gz"
        exit 1
    fi
    
    # Wyłączenie log_bin_trust_function_creators
    log_info "Wyłączanie log_bin_trust_function_creators..."
    mysql -uroot -p"${DB_ROOT_PASSWORD}" <<-EOF
        SET GLOBAL log_bin_trust_function_creators = 0;
EOF
}

configure_zabbix_server() {
    log_step "KROK 9: Konfiguracja Zabbix Server"
    
    log_info "Konfigurowanie połączenia z bazą danych..."
    
    # Backup oryginalnej konfiguracji
    cp /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf.backup
    
    # Ustawienie hasła bazy danych
    sed -i "s/^# DBPassword=.*/DBPassword=${DB_ZABBIX_PASSWORD}/" /etc/zabbix/zabbix_server.conf
    sed -i "s/^DBPassword=.*/DBPassword=${DB_ZABBIX_PASSWORD}/" /etc/zabbix/zabbix_server.conf
    
    # Jeśli linia nie istnieje, dodaj ją
    if ! grep -q "^DBPassword=" /etc/zabbix/zabbix_server.conf; then
        sed -i "/^DBName=zabbix/a DBPassword=${DB_ZABBIX_PASSWORD}" /etc/zabbix/zabbix_server.conf
    fi
    
    log_info "✓ Konfiguracja Zabbix Server zaktualizowana"
}

configure_php() {
    log_step "KROK 10: Konfiguracja PHP"
    
    log_info "Konfigurowanie strefy czasowej PHP..."
    
    # Znajdź plik konfiguracyjny PHP dla Apache
    PHP_INI=$(find /etc/php -name "php.ini" | grep apache2 | head -1)
    
    if [ -n "$PHP_INI" ]; then
        cp "$PHP_INI" "${PHP_INI}.backup"
        sed -i 's/^;date.timezone =.*/date.timezone = Europe\/Warsaw/' "$PHP_INI"
        log_info "✓ Strefa czasowa PHP ustawiona na Europe/Warsaw"
    else
        log_warn "Nie znaleziono pliku php.ini dla Apache"
    fi
}

start_services() {
    log_step "KROK 11: Uruchamianie usług"
    
    log_info "Restartowanie Apache..."
    systemctl restart apache2
    systemctl enable apache2
    
    log_info "Uruchamianie Zabbix Server..."
    systemctl restart zabbix-server
    systemctl enable zabbix-server
    
    log_info "Uruchamianie Zabbix Agent..."
    systemctl restart zabbix-agent
    systemctl enable zabbix-agent
    
    sleep 3
    
    # Sprawdzenie statusów
    local all_ok=true
    
    if systemctl is-active --quiet apache2; then
        log_info "✓ Apache uruchomiony"
    else
        log_error "✗ Apache nie działa"
        all_ok=false
    fi
    
    if systemctl is-active --quiet zabbix-server; then
        log_info "✓ Zabbix Server uruchomiony"
    else
        log_error "✗ Zabbix Server nie działa"
        systemctl status zabbix-server --no-pager -l | tail -20
        all_ok=false
    fi
    
    if systemctl is-active --quiet zabbix-agent; then
        log_info "✓ Zabbix Agent uruchomiony"
    else
        log_error "✗ Zabbix Agent nie działa"
        all_ok=false
    fi
    
    if [ "$all_ok" = false ]; then
        log_error "Niektóre usługi nie uruchomiły się poprawnie"
        exit 1
    fi
}

show_summary() {
    log_step "PODSUMOWANIE INSTALACJI"
    
    local SERVER_IP=$(hostname -I | awk '{print $1}')
    
    echo "=========================================="
    log_info "✓ Instalacja Zabbix zakończona pomyślnie!"
    echo "=========================================="
    echo ""
    echo "DOSTĘP DO INTERFEJSU:"
    echo "  URL: http://${SERVER_IP}/zabbix"
    echo "  lub:  http://localhost/zabbix"
    echo ""
    echo "DOMYŚLNE DANE LOGOWANIA:"
    echo "  Użytkownik: Admin"
    echo "  Hasło:      zabbix"
    echo ""
    echo "DANE DOSTĘPOWE DO BAZY:"
    echo "  Root MariaDB:"
    echo "    Użytkownik: root"
    echo "    Hasło:      ${DB_ROOT_PASSWORD}"
    echo ""
    echo "  Użytkownik Zabbix DB:"
    echo "    Użytkownik: zabbix"
    echo "    Hasło:      ${DB_ZABBIX_PASSWORD}"
    echo "    Baza:       zabbix"
    echo ""
    echo "NASTĘPNE KROKI:"
    echo "1. Otwórz przeglądarkę: http://${SERVER_IP}/zabbix"
    echo "2. Zaloguj się jako Admin / zabbix"
    echo "3. Zmień domyślne hasło administratora"
    echo "4. Zaktualizuj hasła w tym skrypcie dla produkcji!"
    echo ""
    echo "PRZYDATNE KOMENDY:"
    echo "  - Status Zabbix Server: sudo systemctl status zabbix-server"
    echo "  - Status Zabbix Agent:  sudo systemctl status zabbix-agent"
    echo "  - Logi Zabbix Server:   sudo tail -f /var/log/zabbix/zabbix_server.log"
    echo "  - Logi Zabbix Agent:    sudo tail -f /var/log/zabbix/zabbix_agentd.log"
    echo ""
    echo "PLIKI KONFIGURACYJNE:"
    echo "  - Server: /etc/zabbix/zabbix_server.conf"
    echo "  - Agent:  /etc/zabbix/zabbix_agentd.conf"
    echo "  - Apache: /etc/zabbix/apache.conf"
    echo ""
    echo "=========================================="
}

# GŁÓWNY PROGRAM
main() {
    clear
    echo "=========================================="
    echo "  Instalator Zabbix 7.0 LTS Server"
    echo "  Ubuntu 24.04 LTS"
    echo "  MariaDB + Apache + PHP"
    echo "=========================================="
    echo ""
    
    check_root
    check_system
    
    echo ""
    log_warn "Hasła będą ustawione automatycznie:"
    log_warn "  - Root MariaDB: ${DB_ROOT_PASSWORD}"
    log_warn "  - Zabbix DB:    ${DB_ZABBIX_PASSWORD}"
    echo ""
    read -p "Czy kontynuować instalację? (t/n): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Tt]$ ]]; then
        log_info "Instalacja anulowana"
        exit 0
    fi
    
    stop_auto_updates
    wait_for_apt_lock
    
    install_mariadb
    configure_mariadb
    secure_mariadb
    install_zabbix_repo
    install_zabbix_packages
    install_polish_locale
    create_zabbix_database
    import_zabbix_schema
    configure_zabbix_server
    configure_php
    start_services
    show_summary
    
    log_info "Gotowe!"
}

# Uruchom główny program
main
```

### Metoda 2: Zapisanie do pliku i uruchomienie

1. **Utwórz plik skryptu:**
   ```bash
   nano install_zabbix.sh
   ```

2. **Wklej zawartość skryptu** (powyżej)

3. **Nadaj uprawnienia wykonywania:**
   ```bash
   chmod +x install_zabbix.sh
   ```

4. **Uruchom skrypt:**
   ```bash
   sudo ./install_zabbix.sh
   ```

## Uwagi

- **WAŻNE**: Zmień domyślne hasła przed użyciem w środowisku produkcyjnym
- Skrypt automatycznie obsługuje blokady apt i automatyczne aktualizacje
- Instalacja schematu bazy danych może zająć kilka minut
- Po instalacji zmień hasło administratora Zabbix (Admin/zabbix)
