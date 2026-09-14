```bash
#!/bin/bash
################################################################################
# Generator logów z metrykami aplikacyjnymi dla Loki/Grafana
# Generuje logi z wartościami liczbowymi, które można grafować w czasie
################################################################################

set -e

LOG_FILE="/var/log/app-metrics.log"
APP_NAME="MetricsApp"

# Kolory dla konsoli
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
MAGENTA='\033[0;35m'
NC='\033[0m'

echo "=============================================="
echo "Generator logów z metrykami aplikacyjnymi"
echo "=============================================="
echo ""

# Funkcja logowania z metryką
function log_metric() {
    local metric_name=$1
    local metric_value=$2
    local unit=$3
    local level=$4
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local log_line="[$timestamp] [$APP_NAME] [$level] metric=$metric_name value=$metric_value unit=$unit"
    
    # Zapisz do pliku
    echo "$log_line" >> "$LOG_FILE"
    
    # Wyświetl na konsoli z kolorem
    case $level in
        ERROR|CRITICAL) echo -e "${RED}$log_line${NC}" ;;
        WARNING) echo -e "${YELLOW}$log_line${NC}" ;;
        SUCCESS|INFO) echo -e "${GREEN}$log_line${NC}" ;;
        *) echo -e "${CYAN}$log_line${NC}" ;;
    esac
}

# Response Time API - czasy odpowiedzi
function generate_response_time() {
    # Symulacja czasów odpowiedzi z okazjonalnymi spikami
    local spike=$((RANDOM % 100))
    local response_time
    
    if [ $spike -lt 5 ]; then
        # 5% szansa na bardzo wolne zapytanie
        response_time=$((2000 + RANDOM % 3000))
        log_metric "api_response_time" "$response_time" "ms" "WARNING"
    elif [ $spike -lt 15 ]; then
        # 10% szansa na wolne zapytanie
        response_time=$((500 + RANDOM % 1500))
        log_metric "api_response_time" "$response_time" "ms" "INFO"
    else
        # 85% normalne czasy
        response_time=$((50 + RANDOM % 450))
        log_metric "api_response_time" "$response_time" "ms" "INFO"
    fi
}

# Request Rate - liczba requestów na sekundę
function generate_request_rate() {
    # Symulacja zmiennego ruchu (niższy w nocy, wyższy w dzień)
    local hour=$(date +%H)
    local base_rate
    
    # Dzień (8-18h) - więcej ruchu
    if [ $hour -ge 8 ] && [ $hour -le 18 ]; then
        base_rate=$((200 + RANDOM % 300))
    # Wieczór (18-23h) - średni ruch
    elif [ $hour -ge 18 ] && [ $hour -le 23 ]; then
        base_rate=$((100 + RANDOM % 200))
    # Noc (23-8h) - mało ruchu
    else
        base_rate=$((20 + RANDOM % 80))
    fi
    
    log_metric "requests_per_second" "$base_rate" "req/s" "INFO"
}

# Error Rate - procent błędów
function generate_error_rate() {
    # Normalnie niska, czasem spike
    local spike=$((RANDOM % 100))
    local error_rate
    
    if [ $spike -lt 3 ]; then
        # 3% szansa na problemy
        error_rate=$(awk -v seed="$RANDOM" 'BEGIN{srand(seed); printf "%.2f", 5 + rand() * 15}')
        log_metric "error_rate" "$error_rate" "percent" "ERROR"
    else
        # Normalny niski poziom błędów
        error_rate=$(awk -v seed="$RANDOM" 'BEGIN{srand(seed); printf "%.2f", rand() * 3}')
        log_metric "error_rate" "$error_rate" "percent" "INFO"
    fi
}

# Active Sessions - liczba aktywnych sesji użytkowników
function generate_active_sessions() {
    local hour=$(date +%H)
    local sessions
    
    # Symulacja aktywności użytkowników w ciągu dnia
    if [ $hour -ge 9 ] && [ $hour -le 17 ]; then
        # Godziny szczytu
        sessions=$((800 + RANDOM % 400))
    elif [ $hour -ge 18 ] && [ $hour -le 22 ]; then
        # Wieczór
        sessions=$((500 + RANDOM % 300))
    else
        # Noc
        sessions=$((100 + RANDOM % 200))
    fi
    
    log_metric "active_sessions" "$sessions" "count" "INFO"
}

# Database Query Time - czas zapytań do bazy
function generate_db_query_time() {
    local spike=$((RANDOM % 100))
    local query_time
    
    if [ $spike -lt 8 ]; then
        # 8% slow queries
        query_time=$((1000 + RANDOM % 4000))
        log_metric "db_query_time" "$query_time" "ms" "WARNING"
    else
        # Normalne zapytania
        query_time=$((10 + RANDOM % 490))
        log_metric "db_query_time" "$query_time" "ms" "INFO"
    fi
}

# Database Connections - liczba aktywnych połączeń do bazy
function generate_db_connections() {
    # Pool = 100, normalnie wykorzystane 30-70%
    local issue=$((RANDOM % 100))
    local connections
    
    if [ $issue -lt 5 ]; then
        # 5% szansa na wyciek connection pool
        connections=$((85 + RANDOM % 15))
        log_metric "db_active_connections" "$connections" "count" "WARNING"
    else
        connections=$((30 + RANDOM % 40))
        log_metric "db_active_connections" "$connections" "count" "INFO"
    fi
}

# Queue Size - rozmiar kolejki zadań
function generate_queue_size() {
    local spike=$((RANDOM % 100))
    local queue_size
    
    if [ $spike -lt 10 ]; then
        # 10% szansa na zapchana kolejka
        queue_size=$((500 + RANDOM % 1500))
        log_metric "task_queue_size" "$queue_size" "items" "WARNING"
    else
        # Normalna kolejka
        queue_size=$((RANDOM % 200))
        log_metric "task_queue_size" "$queue_size" "items" "INFO"
    fi
}

# Cache Hit Rate - skuteczność cache'a
function generate_cache_hit_rate() {
    local issue=$((RANDOM % 100))
    local hit_rate
    
    if [ $issue -lt 5 ]; then
        # 5% cache problem
        hit_rate=$(awk -v seed="$RANDOM" 'BEGIN{srand(seed); printf "%.2f", 40 + rand() * 30}')
        log_metric "cache_hit_rate" "$hit_rate" "percent" "WARNING"
    else
        # Dobry hit rate (75-98%)
        hit_rate=$(awk -v seed="$RANDOM" 'BEGIN{srand(seed); printf "%.2f", 75 + rand() * 23}')
        log_metric "cache_hit_rate" "$hit_rate" "percent" "INFO"
    fi
}

# Transactions Per Second - liczba transakcji
function generate_transactions() {
    local hour=$(date +%H)
    local tps
    
    if [ $hour -ge 9 ] && [ $hour -le 17 ]; then
        tps=$((150 + RANDOM % 200))
    else
        tps=$((30 + RANDOM % 100))
    fi
    
    log_metric "transactions_per_second" "$tps" "tps" "INFO"
}

# Upload Size - średni rozmiar uploadowanych plików
function generate_upload_size() {
    # Różne rozmiary plików (KB)
    local size=$((100 + RANDOM % 10000))
    log_metric "avg_upload_size" "$size" "KB" "INFO"
}

# API Success Rate - procent udanych zapytań
function generate_success_rate() {
    local issue=$((RANDOM % 100))
    local success_rate
    
    if [ $issue -lt 5 ]; then
        # Problem z serwisem
        success_rate=$(awk -v seed="$RANDOM" 'BEGIN{srand(seed); printf "%.2f", 80 + rand() * 15}')
        log_metric "api_success_rate" "$success_rate" "percent" "ERROR"
    else
        # Normalny wysoki success rate
        success_rate=$(awk -v seed="$RANDOM" 'BEGIN{srand(seed); printf "%.2f", 96 + rand() * 3.9}')
        log_metric "api_success_rate" "$success_rate" "percent" "INFO"
    fi
}

# Concurrent Users - liczba jednoczesnych użytkowników
function generate_concurrent_users() {
    local hour=$(date +%H)
    local users
    
    if [ $hour -ge 10 ] && [ $hour -le 16 ]; then
        users=$((400 + RANDOM % 300))
    elif [ $hour -ge 17 ] && [ $hour -le 22 ]; then
        users=$((250 + RANDOM % 200))
    else
        users=$((50 + RANDOM % 150))
    fi
    
    log_metric "concurrent_users" "$users" "count" "INFO"
}

# Message Queue Lag - opóźnienie w kolejce wiadomości (Kafka/RabbitMQ)
function generate_message_lag() {
    local problem=$((RANDOM % 100))
    local lag
    
    if [ $problem -lt 8 ]; then
        # Problem z konsumerem
        lag=$((5000 + RANDOM % 15000))
        log_metric "message_queue_lag" "$lag" "messages" "WARNING"
    else
        # Normalne opóźnienie
        lag=$((RANDOM % 1000))
        log_metric "message_queue_lag" "$lag" "messages" "INFO"
    fi
}

# Failed Login Attempts - nieudane próby logowania
function generate_failed_logins() {
    local attack=$((RANDOM % 100))
    local attempts
    
    if [ $attack -lt 3 ]; then
        # Możliwy atak brute force
        attempts=$((50 + RANDOM % 150))
        log_metric "failed_login_attempts" "$attempts" "count" "ERROR"
    else
        # Normalny poziom
        attempts=$((RANDOM % 10))
        log_metric "failed_login_attempts" "$attempts" "count" "INFO"
    fi
}

# Webhook Delivery Time - czas dostarczenia webhooka
function generate_webhook_time() {
    local issue=$((RANDOM % 100))
    local delivery_time
    
    if [ $issue -lt 10 ]; then
        # Problemy z zewnętrznym serwisem
        delivery_time=$((3000 + RANDOM % 7000))
        log_metric "webhook_delivery_time" "$delivery_time" "ms" "WARNING"
    else
        delivery_time=$((100 + RANDOM % 900))
        log_metric "webhook_delivery_time" "$delivery_time" "ms" "INFO"
    fi
}

# Funkcja główna - generowanie metryk
function generate_metrics() {
    local iterations=$1
    local interval=${2:-2}  # domyślnie co 2 sekundy
    
    echo "Generowanie metryk: $iterations iteracji, co $interval sekund"
    echo ""
    
    for i in $(seq 1 $iterations); do
        echo -e "${MAGENTA}[Iteracja $i/$iterations]${NC}"
        
        # Generuj różne metryki (nie wszystkie naraz, żeby było bardziej realistyczne)
        generate_response_time
        generate_request_rate
        
        # Co drugą iterację
        if [ $((i % 2)) -eq 0 ]; then
            generate_error_rate
            generate_active_sessions
            generate_db_query_time
        fi
        
        # Co trzecią iterację
        if [ $((i % 3)) -eq 0 ]; then
            generate_db_connections
            generate_queue_size
            generate_cache_hit_rate
        fi
        
        # Co piątą iterację
        if [ $((i % 5)) -eq 0 ]; then
            generate_transactions
            generate_concurrent_users
            generate_success_rate
        fi
        
        # Rzadziej
        if [ $((i % 7)) -eq 0 ]; then
            generate_upload_size
            generate_message_lag
            generate_webhook_time
        fi
        
        # Bardzo rzadko
        if [ $((i % 10)) -eq 0 ]; then
            generate_failed_logins
        fi
        
        # Czekaj określony interwał przed następną iteracją
        if [ $i -lt $iterations ]; then
            sleep $interval
        fi
    done
}

# Tryb ciągły - generuje metryki non-stop
function continuous_mode() {
    local interval=${1:-5}
    
    echo -e "${GREEN}Tryb ciągły - generowanie metryk co $interval sekund${NC}"
    echo "Naciśnij Ctrl+C aby zatrzymać"
    echo ""
    
    while true; do
        generate_metrics 1 $interval
    done
}

# ===== MAIN =====

# Sprawdź/utwórz plik logu
if [ ! -f "$LOG_FILE" ]; then
    sudo touch "$LOG_FILE" 2>/dev/null || touch "$LOG_FILE"
    sudo chmod 666 "$LOG_FILE" 2>/dev/null || chmod 666 "$LOG_FILE"
fi

# Tryby pracy
if [ $# -eq 0 ]; then
    echo "Użycie:"
    echo "  $0 <liczba_iteracji> [interwał_sekund]     # Jednorazowe generowanie"
    echo "  $0 continuous [interwał_sekund]             # Tryb ciągły"
    echo ""
    echo "Przykłady:"
    echo "  $0 50 2           # Wygeneruj 50 pomiarów co 2 sekundy"
    echo "  $0 100 5          # Wygeneruj 100 pomiarów co 5 sekund"
    echo "  $0 continuous 3   # Generuj w kółko co 3 sekundy"
    echo ""
    exit 1
fi

if [ "$1" == "continuous" ]; then
    continuous_mode ${2:-5}
else
    iterations=$1
    interval=${2:-2}
    generate_metrics $iterations $interval
    
    echo ""
    echo -e "${GREEN}✓ Gotowe! Metryki zapisane w: $LOG_FILE${NC}"
    echo ""
    echo "Zobacz logi:"
    echo "  tail -f $LOG_FILE"
    echo ""
    echo "Przykładowe zapytania LogQL w Grafanie:"
    echo '  {job="varlogs"} |= "metric=" | pattern "<_> metric=<metric_name> value=<value>" | line_format "{{.value}}"'
    echo '  {job="varlogs"} |= "api_response_time" | regexp "value=(?P<response_time>[0-9]+)" | unwrap response_time [1m]'
fi
```