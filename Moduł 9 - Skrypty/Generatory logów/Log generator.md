```
#!/bin/bash
################################################################################
# Generator logów aplikacyjnych - ERROR, WARNING, CRITICAL, SUCCESS, INFO, DEBUG
# Można wgrać na dowolną maszynę i uruchomić
################################################################################

set -e

LOG_FILE="/var/log/app-simulator.log"
APP_NAME="TestApp"

# Kolory dla konsoli
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
NC='\033[0m'

echo "======================================"
echo "Generator logów aplikacyjnych"
echo "======================================"
echo ""

# Funkcja logowania
function log_message() {
    local level=$1
    local message=$2
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local log_line="[$timestamp] [$APP_NAME] [$level] $message"
    
    # Zapisz do pliku
    echo "$log_line" >> "$LOG_FILE"
    
    # Wyświetl na konsoli z kolorem
    case $level in
        ERROR|CRITICAL) echo -e "${RED}$log_line${NC}" ;;
        WARNING) echo -e "${YELLOW}$log_line${NC}" ;;
        SUCCESS) echo -e "${GREEN}$log_line${NC}" ;;
        INFO) echo -e "${CYAN}$log_line${NC}" ;;
        *) echo -e "${BLUE}$log_line${NC}" ;;
    esac
}

# ERROR - Błędy bazy danych
function generate_database_error() {
    local errors=(
        "Database connection timeout after 30s - host: db.example.com:5432"
        "SQL query failed: deadlock detected on table users"
        "Connection pool exhausted - max connections: 100"
        "Database replication lag: 45 seconds behind master"
        "Failed to acquire lock on table transactions - timeout"
        "Lost connection to MySQL server during query"
        "Duplicate entry violation: unique constraint failed"
    )
    log_message "ERROR" "${errors[$RANDOM % ${#errors[@]}]}"
}

# ERROR - Błędy API
function generate_api_error() {
    local errors=(
        "API request failed: HTTP 500 - Internal Server Error (endpoint: /api/users)"
        "External API timeout: payment-gateway.com after 60s"
        "Rate limit exceeded: 1000 requests/hour (client: 192.168.1.100)"
        "Invalid API response: JSON parse error from weather-api.com"
        "API authentication failed: invalid token for service XYZ"
        "Service unavailable: HTTP 503 from microservice-auth"
        "Gateway timeout: upstream server not responding"
    )
    log_message "ERROR" "${errors[$RANDOM % ${#errors[@]}]}"
}

# ERROR - Problemy sieciowe
function generate_network_problem() {
    local problems=(
        "Network latency high: 450ms to upstream server"
        "Packet loss detected: 15% on interface eth0"
        "DNS resolution failed: unable to resolve api.example.com"
        "Connection refused: redis-server on localhost:6379"
        "SSL certificate verification failed: expired certificate for *.example.com"
        "TCP connection reset by peer: 10.0.1.50:8080"
        "Socket timeout: no response from 192.168.1.200"
    )
    log_message "ERROR" "${problems[$RANDOM % ${#problems[@]}]}"
}

# WARNING - Bezpieczeństwo
function generate_auth_warning() {
    local warnings=(
        "Failed login attempt: user 'admin' from IP 203.0.113.45"
        "Password about to expire: user 'john.doe' - 3 days remaining"
        "Suspicious login pattern detected: user 'bob' - 5 countries in 1 hour"
        "Session timeout: user 'alice' - idle for 30 minutes"
        "Invalid JWT token: expired 2 hours ago"
        "Multiple failed login attempts: IP 45.77.123.45 blocked"
        "Certificate expiring soon: SSL cert valid for 7 more days"
    )
    log_message "WARNING" "${warnings[$RANDOM % ${#warnings[@]}]}"
}

# WARNING - Wydajność
function generate_performance_warning() {
    local warnings=(
        "High memory usage detected: 85% of 16GB (process: java)"
        "CPU usage spike: 95% for 30 seconds (process: node)"
        "Disk space low: /var partition at 90% capacity"
        "Slow query detected: 5.2s execution time (query_id: 12345)"
        "Cache miss rate high: 75% (expected: <20%)"
        "Thread pool saturation: 95 of 100 threads active"
        "Response time degraded: p95 latency at 2.5s"
    )
    log_message "WARNING" "${warnings[$RANDOM % ${#warnings[@]}]}"
}

# CRITICAL - Krytyczne problemy
function generate_critical_issue() {
    local critical=(
        "CRITICAL: Out of memory - killing process (PID: 1234)"
        "CRITICAL: Disk full on /data partition - write operations failing"
        "CRITICAL: Security breach detected - unauthorized access attempt"
        "CRITICAL: Service down - health check failed 5 times consecutively"
        "CRITICAL: Data corruption detected in table: financial_transactions"
        "CRITICAL: Primary database unreachable - failover initiated"
        "CRITICAL: Memory leak detected - heap growing at 100MB/min"
    )
    log_message "CRITICAL" "${critical[$RANDOM % ${#critical[@]}]}"
}

# SUCCESS - Udane operacje
function generate_success_log() {
    local success=(
        "Payment processed successfully: transaction_id=TXN789456 amount=$125.50"
        "Deployment completed: version v2.3.1 deployed to production"
        "Backup completed successfully: 5.2GB backed up to S3"
        "Health check passed: all services operational"
        "Data migration completed: 10,000 records migrated successfully"
        "User registration successful: user_id=9876 email=user@example.com"
        "Cache rebuild completed: 50,000 entries in 12s"
    )
    log_message "SUCCESS" "${success[$RANDOM % ${#success[@]}]}"
}

# INFO - Informacje
function generate_info_log() {
    local info=(
        "User login successful: username 'user123' from 192.168.1.50"
        "New user registered: email 'newuser@example.com'"
        "File uploaded successfully: document.pdf (size: 2.5MB)"
        "Scheduled job completed: backup_database (duration: 45s)"
        "Configuration reloaded from config.yaml"
        "Cache cleared: 1250 entries removed"
        "Email sent successfully: invoice to customer@example.com"
        "API endpoint called: GET /api/v1/users (200 OK)"
    )
    log_message "INFO" "${info[$RANDOM % ${#info[@]}]}"
}

# DEBUG - Szczegóły debugowania
function generate_debug_log() {
    local debug=(
        "Processing request: GET /api/v1/products?page=1&limit=20"
        "Cache hit: key='user:1234:profile' ttl=3600s"
        "Query executed: SELECT * FROM orders WHERE status='pending' (rows: 45)"
        "WebSocket connection established: client_id=abc123"
        "Background worker started: job_id=xyz789 queue=high_priority"
        "Session created: session_id=sess_abcd1234 user_id=5678"
        "Transaction started: tx_id=TX_9988776655 isolation=READ_COMMITTED"
    )
    log_message "DEBUG" "${debug[$RANDOM % ${#debug[@]}]}"
}

# Instalacja jako usługa systemd (opcjonalne)
function install_as_service() {
    echo ""
    echo "Czy chcesz zainstalować jako automatyczną usługę? (generuje logi co 2 min)"
    read -p "Instalować usługę systemd? [T/n]: " install_service
    
    if [[ $install_service =~ ^[Tt]$ ]] || [[ -z $install_service ]]; then
        sudo tee /etc/systemd/system/app-log-generator.service > /dev/null <<'EOF'
[Unit]
Description=Generator logów aplikacyjnych
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/app-log-generator.sh 5
StandardOutput=journal
StandardError=journal
EOF

        sudo tee /etc/systemd/system/app-log-generator.timer > /dev/null <<'EOF'
[Unit]
Description=Timer generatora logów (co 2 minuty)
Requires=app-log-generator.service

[Timer]
OnBootSec=1min
OnUnitActiveSec=2min
AccuracySec=1s

[Install]
WantedBy=timers.target
EOF
        
        # Skopiuj skrypt do lokalizacji systemowej
        sudo cp "$0" /usr/local/bin/app-log-generator.sh
        sudo chmod +x /usr/local/bin/app-log-generator.sh
        
        sudo systemctl daemon-reload
        sudo systemctl enable app-log-generator.timer
        sudo systemctl start app-log-generator.timer
        
        echo -e "${GREEN}✓ Usługa zainstalowana i uruchomiona!${NC}"
        echo "  Status: systemctl status app-log-generator.timer"
        echo "  Logi: journalctl -u app-log-generator -f"
    fi
}

# Główna funkcja generująca logi
function generate_logs() {
    local iterations=$1
    
    echo "Generowanie $iterations zdarzeń..."
    echo ""
    
    for i in $(seq 1 $iterations); do
        EVENT_TYPE=$((RANDOM % 100))
        
        # 20% ERROR (różne typy)
        if [ $EVENT_TYPE -lt 20 ]; then
            SUB_TYPE=$((RANDOM % 3))
            case $SUB_TYPE in
                0) generate_database_error ;;
                1) generate_api_error ;;
                2) generate_network_problem ;;
            esac
        
        # 15% WARNING
        elif [ $EVENT_TYPE -lt 35 ]; then
            SUB_TYPE=$((RANDOM % 2))
            case $SUB_TYPE in
                3) generate_auth_warning ;;
                4) generate_performance_warning ;;
            esac
        
        # 5% CRITICAL
        elif [ $EVENT_TYPE -lt 40 ]; then
            generate_critical_issue
        
        # 20% SUCCESS
        elif [ $EVENT_TYPE -lt 60 ]; then
            generate_success_log
        
        # 20% INFO
        elif [ $EVENT_TYPE -lt 80 ]; then
            generate_info_log
        
        # 20% DEBUG
        else
            generate_debug_log
        fi
        
        # Losowe opóźnienie między zdarzeniami
        sleep $(awk -v seed="$RANDOM" 'BEGIN{srand(seed); print 0.3 + rand() * 1.2}')
    done
}

# ===== MAIN =====

# Sprawdź/utwórz plik logu
if [ ! -f "$LOG_FILE" ]; then
    sudo touch "$LOG_FILE"
    sudo chmod 666 "$LOG_FILE"
fi

# Jeśli uruchamiany bez argumentów - zapytaj użytkownika
if [ $# -eq 0 ]; then
    echo "Wybierz opcję:"
    echo "  1) Generuj logi jednokrotnie"
    echo "  2) Zainstaluj jako usługę systemd (auto-generacja co 2 min)"
    echo ""
    read -p "Wybór [1-2]: " choice
    
    case $choice in
        1)
            read -p "Ile zdarzeń wygenerować? [50]: " events
            events=${events:-50}
            generate_logs $events
            echo ""
            echo -e "${GREEN}Gotowe! Logi zapisane w: $LOG_FILE${NC}"
            echo "Zobacz: tail -f $LOG_FILE"
            ;;
        2)
            install_as_service
            # Wygeneruj początkowe logi
            generate_logs 10
            ;;
        *)
            echo "Nieprawidłowy wybór"
            exit 1
            ;;
    esac
else
    # Uruchomiono z argumentem (liczba zdarzeń)
    generate_logs $1
fi

```