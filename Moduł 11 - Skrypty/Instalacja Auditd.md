# Skrypt instalacji Auditd z automatycznym generatorem logów

Prosty skrypt instalacyjny Auditd (Linux Audit Daemon) dla systemów Ubuntu/Debian. **Odpalisz raz - działa w tle.**

---

## Opis skryptu

Skrypt automatycznie:
- Instaluje **auditd** i konfiguruje podstawowe reguły
- Tworzy **generator przykładowych logów** dla celów szkoleniowych
- Uruchamia **systemd timer** generujący zdarzenia co 30 sekund w tle

**Cel szkoleniowy:**
Czysty serwer generuje bardzo mało zdarzeń audytu. Ten skrypt symuluje normalną aktywność systemową (logowania, zmiany plików, dostępy do wrażliwych danych) aby uczestnicy mieli materiał do analizy.

**Generator automatyczny:**
- 5 losowych zdarzeń co 30 sekund
- ~150 zdarzeń na godzinę
- Działa ciągle w tle po instalacji

---

## Pełny skrypt

```bash
#!/bin/bash
set -e

echo "======================================"
echo "Instalacja Auditd z generatorem logów"
echo "======================================"
echo ""

function install_auditd() {
	echo "=== Instalacja Auditd ==="
	
	# Krok 1: Instalacja pakietów
	sudo apt-get update
	sudo apt-get install -y auditd audispd-plugins netcat-openbsd
	
	# Krok 2: Konfiguracja podstawowych reguł audytu
	echo "Konfiguracja reguł audytu..."
	sudo tee /etc/audit/rules.d/monitoring.rules > /dev/null <<'EOF'
## System Calls - monitorowanie krytycznych wywołań systemowych
-a always,exit -F arch=b64 -S execve -k exec_commands
-a always,exit -F arch=b64 -S connect -S accept -k network_connections

## File Access - monitorowanie dostępu do plików
-w /etc/passwd -p wa -k passwd_changes
-w /etc/group -p wa -k group_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/ssh/sshd_config -p wa -k sshd_config_changes

## Authentication - zdarzenia logowania
-w /var/log/auth.log -p wa -k auth_log_access
-w /var/log/faillog -p wa -k failed_logins

## Process Execution
-w /usr/bin/sudo -p x -k sudo_execution
-w /usr/bin/ssh -p x -k ssh_execution

## Directory monitoring
-w /tmp -p wa -k tmp_directory
-w /home -p wa -k home_directory
EOF
	
	# Krok 3: Reload reguł
	sudo augenrules --load
	
	# Krok 4: Uruchomienie auditd
	sudo systemctl enable auditd
	sudo systemctl restart auditd
	
	echo "✓ Auditd zainstalowany i uruchomiony!"
}

function install_log_generator() {
	echo ""
	echo "=== Instalacja generatora logów szkoleniowych ==="
	
	# Tworzymy skrypt generujący przykładowe zdarzenia
	sudo tee /usr/local/bin/audit-log-generator.sh > /dev/null <<'GENERATOR_EOF'
#!/bin/bash
# Generator przykładowych zdarzeń audytu dla celów szkoleniowych

LOG_PREFIX="[SZKOLENIE]"

function generate_file_access() {
	FILES=("/etc/passwd" "/etc/shadow" "/etc/sudoers" "/etc/ssh/sshd_config")
	FILE=${FILES[$RANDOM % ${#FILES[@]}]}
	sudo cat "$FILE" > /dev/null 2>&1 || true
	echo "$LOG_PREFIX Wygenerowano dostęp do: $FILE"
}

function generate_process_execution() {
	COMMANDS=("whoami" "id" "hostname" "uptime" "df -h" "free -h" "ps aux | head -5")
	CMD=${COMMANDS[$RANDOM % ${#COMMANDS[@]}]}
	eval "$CMD" > /dev/null 2>&1
	echo "$LOG_PREFIX Wykonano polecenie: $CMD"
}

function generate_network_activity() {
	HOSTS=("google.com" "localhost" "127.0.0.1")
	HOST=${HOSTS[$RANDOM % ${#HOSTS[@]}]}
	timeout 1 nc -z "$HOST" 80 2>/dev/null || true
	echo "$LOG_PREFIX Próba połączenia z: $HOST"
}

function generate_sudo_activity() {
	sudo whoami > /dev/null 2>&1
	echo "$LOG_PREFIX Użycie sudo"
}

function generate_file_modifications() {
	TEMP_FILE="/tmp/audit_test_$(date +%s).tmp"
	echo "Test audit log - $(date)" > "$TEMP_FILE"
	chmod 644 "$TEMP_FILE"
	rm -f "$TEMP_FILE"
	echo "$LOG_PREFIX Modyfikacja pliku w /tmp"
}

function generate_ssh_simulation() {
	sudo cat /etc/ssh/sshd_config > /dev/null 2>&1 || true
	echo "$LOG_PREFIX Dostęp do konfiguracji SSH"
}

# Generowanie zdarzeń
ITERATIONS=${1:-50}
for i in $(seq 1 $ITERATIONS); do
	EVENT_TYPE=$((RANDOM % 6))
	case $EVENT_TYPE in
		0) generate_file_access ;;
		1) generate_process_execution ;;
		2) generate_network_activity ;;
		3) generate_sudo_activity ;;
		4) generate_file_modifications ;;
		5) generate_ssh_simulation ;;
	esac
	sleep $(awk -v seed="$RANDOM" 'BEGIN{srand(seed); print 0.5 + rand() * 1.5}')
done
GENERATOR_EOF
	
	sudo chmod +x /usr/local/bin/audit-log-generator.sh
	echo "✓ Generator logów zainstalowany"
	
	echo ""
	echo "Generowanie początkowych zdarzeń demonstracyjnych..."
	sudo /usr/local/bin/audit-log-generator.sh 30
}

function install_auto_generator() {
	echo ""
	echo "=== Konfiguracja automatycznego generatora ==="
	
	sudo tee /etc/systemd/system/audit-log-generator.service > /dev/null <<'SERVICE_EOF'
[Unit]
Description=Generator logów audytu dla szkoleń
After=auditd.service
Requires=auditd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/audit-log-generator.sh 5
StandardOutput=journal
StandardError=journal
SERVICE_EOF
	
	sudo tee /etc/systemd/system/audit-log-generator.timer > /dev/null <<'TIMER_EOF'
[Unit]
Description=Timer generatora logów audytu (co 30 sekund)
Requires=audit-log-generator.service

[Timer]
OnBootSec=1min
OnUnitActiveSec=30s
AccuracySec=1s

[Install]
WantedBy=timers.target
TIMER_EOF
	
	sudo systemctl daemon-reload
	sudo systemctl enable audit-log-generator.timer
	sudo systemctl start audit-log-generator.timer
	
	echo "✓ Automatyczny generator uruchomiony!"
	echo "  → Generuje 5 losowych zdarzeń co 30 sekund (~150/h)"
	echo "  → Działa w tle automatycznie"
}

# Wykonanie instalacji
install_auditd
install_log_generator
install_auto_generator

echo ""
echo "======================================"
echo "✓ Instalacja zakończona!"
echo "======================================"
echo ""
echo "Auditd działa w tle i automatycznie generuje logi szkoleniowe."
echo ""
echo "Przydatne polecenia:"
echo "  sudo ausearch -ts today              # Wszystkie zdarzenia z dzisiaj"
echo "  sudo ausearch -ts recent | tail -50  # Ostatnie 50 zdarzeń"
echo "  sudo aureport --summary              # Raport podsumowujący"
echo "  sudo tail -f /var/log/audit/audit.log # Podgląd na żywo"
echo ""
echo "Status generatora:"
echo "  systemctl status audit-log-generator.timer   # Status timera"
echo "  journalctl -u audit-log-generator -f         # Logi generatora"
echo ""
```

---

## Użycie skryptu

### Instalacja

```bash
# Nadanie uprawnień
chmod +x install-auditd.sh

# Uruchomienie instalacji
./install-auditd.sh
```

**To wszystko!** Skrypt automatycznie:
1. Zainstaluje auditd i skonfiguruje reguły
2. Stworzy generator logów
3. Uruchomi systemd timer generujący zdarzenia w tle

**Przykładowe wyjście:**
```
======================================
Instalacja Auditd z generatorem logów
======================================

=== Instalacja Auditd ===
✓ Auditd zainstalowany i uruchomiony!

=== Instalacja generatora logów szkoleniowych ===
✓ Generator logów zainstalowany
Generowanie początkowych zdarzeń demonstracyjnych...
[SZKOLENIE] Wygenerowano dostęp do: /etc/passwd
[SZKOLENIE] Wykonano polecenie: whoami
...

=== Konfiguracja automatycznego generatora ===
✓ Automatyczny generator uruchomiony!
  → Generuje 5 losowych zdarzeń co 30 sekund (~150/h)
  → Działa w tle automatycznie

======================================
✓ Instalacja zakończona!
======================================
```

---

## Sprawdzenie czy działa

### Status timera

```bash
systemctl status audit-log-generator.timer
```

Przykładowy wynik:
```
● audit-log-generator.timer - Timer generatora logów audytu (co 30 sekund)
     Loaded: loaded (/etc/systemd/system/audit-log-generator.timer)
     Active: active (waiting)
    Trigger: Wed 2026-02-11 12:05:30 UTC; 15s left
```

### Kiedy następne uruchomienie

```bash
systemctl list-timers audit-log-generator.timer
```

### Historia generowanych zdarzeń

```bash
journalctl -u audit-log-generator -f
```

---

## Przeglądanie logów audytu

### Wszystkie zdarzenia z dzisiaj

```bash
sudo ausearch -ts today
```

### Ostatnie zdarzenia

```bash
sudo ausearch -ts recent | tail -50
```

### Zdarzenia szkoleniowe

```bash
sudo ausearch -ts today | grep SZKOLENIE
```

### Zdarzenia według klucza

```bash
sudo ausearch -k passwd_changes    # Zmiany w /etc/passwd
sudo ausearch -k exec_commands      # Wykonane polecenia
sudo ausearch -k sudo_execution     # Użycie sudo
sudo ausearch -k network_connections # Połączenia sieciowe
```

### Raporty podsumowujące

```bash
sudo aureport --summary             # Podsumowanie wszystkich zdarzeń
sudo aureport --auth                # Zdarzenia autentykacji
sudo aureport --file                # Dostępy do plików
sudo aureport --executable          # Wykonane programy
```

### Podgląd na żywo

```bash
sudo tail -f /var/log/audit/audit.log
```

---

## Zarządzanie generatorem

### Zatrzymanie generatora

```bash
sudo systemctl stop audit-log-generator.timer
```

### Uruchomienie generatora

```bash
sudo systemctl start audit-log-generator.timer
```

### Wyłączenie automatycznego startu

```bash
sudo systemctl disable audit-log-generator.timer
```

### Włączenie automatycznego startu

```bash
sudo systemctl enable audit-log-generator.timer
```

### Status

```bash
systemctl is-active audit-log-generator.timer
systemctl is-enabled audit-log-generator.timer
```

---

## Wyjaśnienie działania

### Podstawowe komponenty

**1. Auditd**
- Demon audytu systemowego Linux
- Monitoruje wywołania systemowe (execve, connect, accept)
- Śledzi dostęp do plików (/etc/passwd, /etc/shadow, /etc/sudoers)
- Zapisuje zdarzenia do `/var/log/audit/audit.log`

**2. Generator logów**
- Skrypt `/usr/local/bin/audit-log-generator.sh`
- Symuluje 6 typów aktywności systemowej
- Każde zdarzenie ma prefix `[SZKOLENIE]`

**3. Systemd timer**
- Uruchamia generator co 30 sekund
- 5 zdarzeń na wywołanie = ~150 zdarzeń/h
- Automatyczny start przy boocie systemu

### Typy generowanych zdarzeń

1. **Dostęp do wrażliwych plików** - odczyt /etc/passwd, /etc/shadow, /etc/sudoers
2. **Wykonywanie poleceń** - whoami, id, hostname, df, free, ps
3. **Aktywność sieciowa** - próby połączeń (nc -z)
4. **Użycie sudo** - zdarzenia podwyższenia uprawnień
5. **Modyfikacje plików** - tworzenie/usuwanie plików w /tmp
6. **Konfiguracja SSH** - odczyt /etc/ssh/sshd_config

### Reguły audytu

Skrypt konfiguruje następujące reguły w `/etc/audit/rules.d/monitoring.rules`:

**Wywołania systemowe:**
```
-a always,exit -F arch=b64 -S execve -k exec_commands
-a always,exit -F arch=b64 -S connect -S accept -k network_connections
```

**Monitorowanie plików:**
```
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/ssh/sshd_config -p wa -k sshd_config_changes
```

**Katalogi:**
```
-w /tmp -p wa -k tmp_directory
-w /home -p wa -k home_directory
```

Gdzie:
- `-w` - watch (obserwuj plik)
- `-p wa` - permissions: write, attribute change
- `-k` - klucz do wyszukiwania
- `-a` - append rule
- `-S` - syscall name

---

## Deinstalacja

```bash
# Zatrzymanie timera
sudo systemctl stop audit-log-generator.timer
sudo systemctl disable audit-log-generator.timer

# Usunięcie plików systemd
sudo rm -f /etc/systemd/system/audit-log-generator.service
sudo rm -f /etc/systemd/system/audit-log-generator.timer
sudo systemctl daemon-reload

# Usunięcie generatora
sudo rm -f /usr/local/bin/audit-log-generator.sh

# Usunięcie auditd
sudo systemctl stop auditd
sudo systemctl disable auditd
sudo apt-get purge -y auditd audispd-plugins
sudo apt-get autoremove -y

# Usunięcie konfiguracji i logów
sudo rm -rf /etc/audit
sudo rm -rf /var/log/audit
```

---

## Podsumowanie

Prosty skrypt instalacyjny Auditd dla środowisk szkoleniowych:

✅ **Jedna komenda** - odpalisz i działa  
✅ **Automatyczne generowanie** - 5 zdarzeń co 30 sekund (~150/h)  
✅ **6 typów zdarzeń** - pliki, procesy, sieć, sudo, SSH, modyfikacje  
✅ **Działa w tle** - systemd timer, auto-start po restarcie  
✅ **Gotowe logi** - natychmiast po instalacji materiał do analizy  

**Idealne dla szkoleń:**
- Security Operations Center (SOC)
- Incident Response
- Digital Forensics
- Compliance Auditing
- SIEM (Elasticsearch/Splunk/Grafana Loki)

Generator tworzy realistyczne zdarzenia bez potrzeby rzeczywistej aktywności na serwerze!
