# Skrypt instalacji i deinstalacji Auditd z generatorem logów

Automatyczny skrypt do instalacji i usuwania Auditd (Linux Audit Daemon) na systemach Ubuntu/Debian. Skrypt instaluje auditd, konfiguruje podstawowe reguły audytu i **tworzy generator przykładowych logów dla celów szkoleniowych**.

---

## Opis skryptu

Skrypt zawiera trzy główne funkcje:
- **Instalacja Auditd** - automatyczna instalacja demona audytu systemowego
- **Generator logów szkoleniowych** - symulator aktywności systemowej dla celów demonstracyjnych
- **Deinstalacja Auditd** - całkowite usunięcie aplikacji wraz z danymi i konfiguracją

**Co jest instalowane:**
- **auditd** - demon audytu systemowego Linux
- **audispd-plugins** - wtyczki do przekazywania logów
- Skrypt generatora przykładowych zdarzeń
- Podstawowe reguły audytu

**Cel szkoleniowy:**
Czysty serwer generuje bardzo mało zdarzeń audytu. Ten skrypt symuluje normalną aktywność systemową (logowania, zmiany plików, dostępy do wrażliwych danych) aby uczestnicy mieli materiał do analizy.

---

## Pełny skrypt

```bash
#!/bin/bash
set -e

function install_auditd() {
	echo "=== Instalacja Auditd ==="
	
	# Krok 1: Instalacja pakietów
	sudo apt-get update
	sudo apt-get install -y auditd audispd-plugins
	
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
	
	# Krok 5: Instalacja generatora logów szkoleniowych
	install_log_generator
}

function enable_auto_generation() {
	echo ""
	echo "=== Włączanie automatycznego generowania logów ==="
	
	# Tworzymy systemd service
	sudo tee /etc/systemd/system/audit-log-generator.service > /dev/null <<'SERVICE_EOF'
[Unit]
Description=Generator logów audytu dla szkoleń
After=auditd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/audit-log-generator.sh 20
StandardOutput=journal
StandardError=journal
SERVICE_EOF
	
	# Tworzymy systemd timer (uruchamia co 5 minut)
	sudo tee /etc/systemd/system/audit-log-generator.timer > /dev/null <<'TIMER_EOF'
[Unit]
Description=Automatyczne generowanie logów audytu (co 5 min)
After=auditd.service

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
AccuracySec=1s

[Install]
WantedBy=timers.target
TIMER_EOF
	
	# Włączenie timera
	sudo systemctl daemon-reload
	sudo systemctl enable audit-log-generator.timer
	sudo systemctl start audit-log-generator.timer
	
	echo "✓ Automatyczne generowanie włączone (20 zdarzeń co 5 minut)"
	echo ""
	echo "Status timera:"
	sudo systemctl status audit-log-generator.timer --no-pager -l | head -10
	echo ""
	echo "Następne uruchomienie:"
	systemctl list-timers audit-log-generator.timer --no-pager
}

function disable_auto_generation() {
	echo "=== Wyłączanie automatycznego generowania ==="
	sudo systemctl stop audit-log-generator.timer 2>/dev/null || true
	sudo systemctl disable audit-log-generator.timer 2>/dev/null || true
	sudo rm -f /etc/systemd/system/audit-log-generator.service
	sudo rm -f /etc/systemd/system/audit-log-generator.timer
	sudo systemctl daemon-reload
	echo "✓ Automatyczne generowanie wyłączone"
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
	# Symulacja dostępu do wrażliwych plików
	FILES=("/etc/passwd" "/etc/shadow" "/etc/sudoers" "/etc/ssh/sshd_config")
	FILE=${FILES[$RANDOM % ${#FILES[@]}]}
	
	sudo cat "$FILE" > /dev/null 2>&1 || true
	echo "$LOG_PREFIX Wygenerowano dostęp do: $FILE"
}

function generate_process_execution() {
	# Symulacja wykonywania różnych poleceń
	COMMANDS=("whoami" "id" "hostname" "uptime" "df -h" "free -h" "ps aux | head -5")
	CMD=${COMMANDS[$RANDOM % ${#COMMANDS[@]}]}
	
	eval "$CMD" > /dev/null 2>&1
	echo "$LOG_PREFIX Wykonano polecenie: $CMD"
}

function generate_network_activity() {
	# Symulacja aktywności sieciowej
	HOSTS=("google.com" "localhost" "127.0.0.1")
	HOST=${HOSTS[$RANDOM % ${#HOSTS[@]}]}
	
	timeout 1 nc -z "$HOST" 80 2>/dev/null || true
	echo "$LOG_PREFIX Próba połączenia z: $HOST"
}

function generate_sudo_activity() {
	# Symulacja użycia sudo
	sudo whoami > /dev/null 2>&1
	echo "$LOG_PREFIX Użycie sudo"
}

function generate_file_modifications() {
	# Symulacja modyfikacji plików w /tmp
	TEMP_FILE="/tmp/audit_test_$(date +%s).tmp"
	echo "Test audit log - $(date)" > "$TEMP_FILE"
	chmod 644 "$TEMP_FILE"
	rm -f "$TEMP_FILE"
	echo "$LOG_PREFIX Modyfikacja pliku w /tmp"
}

function generate_ssh_simulation() {
	# Symulacja sprawdzania konfiguracji SSH
	sudo cat /etc/ssh/sshd_config > /dev/null 2>&1 || true
	echo "$LOG_PREFIX Dostęp do konfiguracji SSH"
}

# Pętla generująca zdarzenia
echo "========================================"
echo "Generator logów auditd - TRYB SZKOLENIOWY"
echo "========================================"
echo "Generowanie przykładowych zdarzeń..."
echo ""

ITERATIONS=${1:-50}
for i in $(seq 1 $ITERATIONS); do
	# Losowy wybór typu zdarzenia
	EVENT_TYPE=$((RANDOM % 6))
	
	case $EVENT_TYPE in
		0) generate_file_access ;;
		1) generate_process_execution ;;
		2) generate_network_activity ;;
		3) generate_sudo_activity ;;
		4) generate_file_modifications ;;
		5) generate_ssh_simulation ;;
	esac
	
	# Losowe opóźnienie 0.5-2 sekundy
	sleep $(awk -v seed="$RANDOM" 'BEGIN{srand(seed); print 0.5 + rand() * 1.5}')
done

echo ""
echo "✓ Wygenerowano $ITERATIONS przykładowych zdarzeń!"
echo "Sprawdź logi: sudo ausearch -ts today"
GENERATOR_EOF
	
	# Nadanie uprawnień
	sudo chmod +x /usr/local/bin/audit-log-generator.sh
	
	echo "✓ Generator logów zainstalowany: /usr/local/bin/audit-log-generator.sh"
	echo ""
	echo "Użycie:"
	echo "  sudo /usr/local/bin/audit-log-generator.sh        # Generuje 50 zdarzeń (domyślnie)"
	echo "  sudo /usr/local/bin/audit-log-generator.sh 100    # Generuje 100 zdarzeń"
	echo ""
	
	# Generujemy początkowe zdarzenia
	echo "Generowanie początkowych zdarzeń demonstracyjnych..."
	sudo /usr/local/bin/audit-log-generator.sh 30
	
	# Pytamy czy włączyć automatyczne generowanie
	echo ""
	echo "=========================================="
	echo "Czy włączyć automatyczne generowanie logów w tle?"
	echo "(Rekomendowane dla środowiska szkoleniowego)"
	echo "=========================================="
	read -p "Automatyczne generowanie [T/n]? " -n 1 -r
	echo
	if [[ $REPLY =~ ^[TtYy]$ ]] || [[ -z $REPLY ]]; then
		enable_auto_generation
	else
		echo "Pominięto automatyczne generowanie."
		echo "Możesz włączyć później: $0 --enable-auto"
	fi
}

function remove_auditd() {
	echo "=== Deinstalacja Auditd ==="
	
	# Zatrzymanie usługi
	sudo systemctl stop auditd || true
	sudo systemctl disable auditd || true
	
	# Odinstalowanie pakietów
	sudo apt-get purge -y auditd audispd-plugins
	sudo apt-get autoremove -y
	
	# Usunięcie konfiguracji i logów
	sudo rm -rf /etc/audit
	sudo rm -rf /var/log/audit
	sudo rm -f /usr/local/bin/audit-log-generator.sh
	
	echo "✓ Auditd został całkowicie usunięty!"
}

function verify_installation() {
	echo ""
	echo "=== Weryfikacja instalacji ==="
	
	# Status usługi
	echo -n "Auditd: "
	if systemctl is-active --quiet auditd; then
		echo "✓ DZIAŁA"
	else
		echo "✗ NIE DZIAŁA"
	fi
	
	# Liczba reguł
	RULES_COUNT=$(sudo auditctl -l | grep -v "No rules" | wc -l)
	echo "Liczba reguł audytu: $RULES_COUNT"
	
	# Liczba zdarzeń w logu
	if [ -f /var/log/audit/audit.log ]; then
		LOG_SIZE=$(du -h /var/log/audit/audit.log | cut -f1)
		EVENTS_COUNT=$(sudo wc -l < /var/log/audit/audit.log)
		echo "Plik audit.log: $LOG_SIZE ($EVENTS_COUNT linii)"
	fi
	
	echo ""
	echo "=== Przykładowe zdarzenia ==="
	echo "Ostatnie 5 zdarzeń:"
	sudo ausearch -ts today 2>/dev/null | tail -20 || echo "Brak zdarzeń lub ausearch niedostępny"
}

case "$1" in
	--install)
		install_auditd
		verify_installation
		echo ""
		echo "=== Auditd gotowy! ==="
		echo ""
		echo "Przydatne polecenia:"
		echo "  sudo ausearch -ts today              # Wyszukaj zdarzenia z dzisiaj"
		echo "  sudo ausearch -k passwd_changes      # Zdarzenia związane z passwd"
		echo "  sudo aureport --summary              # Raport podsumowujący"
		echo "  sudo tail -f /var/log/audit/audit.log # Podgląd na żywo"
		echo ""
		echo "Generator logów szkoleniowych:"
		echo "  sudo /usr/local/bin/audit-log-generator.sh        # Manualne generowanie"
		echo "  $0 --enable-auto                                  # Włącz auto-generowanie"
		echo "  $0 --disable-auto                                 # Wyłącz auto-generowanie"
		;;
	--generate)
		COUNT=${2:-50}
		if [ ! -f /usr/local/bin/audit-log-generator.sh ]; then
			echo "Błąd: Generator nie jest zainstalowany. Uruchom najpierw: $0 --install"
			exit 1
		fi
		sudo /usr/local/bin/audit-log-generator.sh "$COUNT"
		;;
	--enable-auto)
		if [ ! -f /usr/local/bin/audit-log-generator.sh ]; then
			echo "Błąd: Generator nie jest zainstalowany. Uruchom najpierw: $0 --install"
			exit 1
		fi
		enable_auto_generation
		;;
	--disable-auto)
		disable_auto_generation
		;;
	--remove)
		disable_auto_generation
		remove_auditd
		;;
	*)
		echo "Użycie: $0 --install | --generate [liczba] | --enable-auto | --disable-auto | --remove"
		echo ""
		echo "  --install          Instaluje auditd i generator logów"
		echo "  --generate [N]     Generuje N przykładowych zdarzeń (domyślnie 50)"
		echo "  --enable-auto      Włącza automatyczne generowanie (20 zdarzeń co 5 min)"
		echo "  --disable-auto     Wyłącza automatyczne generowanie"
		echo "  --remove           Usuwa auditd całkowicie"
		exit 1
		;;
esac
```

---

## Wyjaśnienie funkcji install_auditd()

### Krok 1: Instalacja pakietów

```bash
sudo apt-get install -y auditd audispd-plugins
```

- **auditd**: Główny demon audytu systemowego Linux
- **audispd-plugins**: Wtyczki do dispatcher (przesyłanie logów do SIEM, syslog)

### Krok 2: Konfiguracja reguł audytu

```bash
sudo tee /etc/audit/rules.d/monitoring.rules
```

Tworzymy plik z regułami audytu. Każda reguła monitoruje określone zdarzenia:

**Wywołania systemowe:**
```bash
-a always,exit -F arch=b64 -S execve -k exec_commands
```
- `-a always,exit`: Zawsze loguj przy wyjściu z syscall
- `-F arch=b64`: Architektura 64-bit
- `-S execve`: Syscall wykonywania programów
- `-k exec_commands`: Klucz/etykieta do wyszukiwania

**Monitorowanie plików:**
```bash
-w /etc/passwd -p wa -k passwd_changes
```
- `-w`: Watch (obserwuj plik)
- `-p wa`: Permissions - `w`rite, `a`ttribute change
- `-k`: Klucz identyfikujący

**Reguły obejmują:**
1. **Pliki systemowe**: `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`
2. **Pliki konfiguracyjne**: SSH, PAM
3. **Logi autentykacji**: `/var/log/auth.log`
4. **Wykonywanie programów**: `sudo`, `ssh`
5. **Katalogi**: `/tmp`, `/home`

### Krok 3: Załadowanie reguł

```bash
sudo augenrules --load
```

Kompiluje reguły z `/etc/audit/rules.d/` i ładuje do jądra.

### Krok 4: Uruchomienie usługi

```bash
sudo systemctl enable auditd
sudo systemctl restart auditd
```

Włącza autostart i restartuje usługę.

---

## Wyjaśnienie funkcji install_log_generator()

### Generator logów szkoleniowych

Generator symuluje różne typy aktywności systemowej:

#### 1. **Dostęp do wrażliwych plików**
```bash
function generate_file_access() {
	FILES=("/etc/passwd" "/etc/shadow" "/etc/sudoers")
	FILE=${FILES[$RANDOM % ${#FILES[@]}]}
	sudo cat "$FILE" > /dev/null 2>&1
}
```

Losowo odczytuje jeden z wrażliwych plików systemowych.

#### 2. **Wykonywanie poleceń**
```bash
function generate_process_execution() {
	COMMANDS=("whoami" "id" "hostname" "uptime")
	eval "$CMD" > /dev/null 2>&1
}
```

Wykonuje różne polecenia systemowe.

#### 3. **Aktywność sieciowa**
```bash
function generate_network_activity() {
	timeout 1 nc -z "google.com" 80 2>/dev/null
}
```

Symuluje połączenia sieciowe (port scanning).

#### 4. **Użycie sudo**
```bash
function generate_sudo_activity() {
	sudo whoami > /dev/null
}
```

Generuje zdarzenia związane z podwyższeniem uprawnień.

#### 5. **Modyfikacje plików**
```bash
function generate_file_modifications() {
	TEMP_FILE="/tmp/audit_test_$(date +%s).tmp"
	echo "Test" > "$TEMP_FILE"
	rm -f "$TEMP_FILE"
}
```

Tworzy i usuwa pliki w `/tmp` (często używane przez atakujących).

#### 6. **Konfiguracja SSH**
```bash
function generate_ssh_simulation() {
	sudo cat /etc/ssh/sshd_config > /dev/null
}
```

Odczytuje konfigurację SSH.

### Pętla generująca

```bash
for i in $(seq 1 $ITERATIONS); do
	EVENT_TYPE=$((RANDOM % 6))
	case $EVENT_TYPE in
		0) generate_file_access ;;
		1) generate_process_execution ;;
		# itd...
	esac
	sleep $(awk 'BEGIN{print 0.5 + rand() * 1.5}')
done
```

- Losowo wybiera typ zdarzenia (0-5)
- Wykonuje odpowiednią funkcję
- Czeka losowy czas (0.5-2s) dla realizmu

**Prefix `[SZKOLENIE]`:** Wszystkie komunikaty generatora mają prefix aby odróżnić je od rzeczywistych działań.

---

## Wyjaśnienie funkcji remove_auditd()

### Zatrzymanie usługi

```bash
sudo systemctl stop auditd
sudo systemctl disable auditd
```

Wyłącza auditd.

### Usunięcie pakietów

```bash
sudo apt-get purge -y auditd audispd-plugins
```

`purge` usuwa pakiety wraz z plikami konfiguracyjnymi.

### Czyszczenie danych

```bash
sudo rm -rf /etc/audit          # Konfiguracja
sudo rm -rf /var/log/audit      # Logi
sudo rm -f /usr/local/bin/audit-log-generator.sh  # Generator
```

Usuwa wszystkie ślady instalacji.

---

## Wyjaśnienie funkcji enable_auto_generation()

### Tworzenie systemd service

```bash
sudo tee /etc/systemd/system/audit-log-generator.service
```

**Zawartość service:**
```ini
[Unit]
Description=Generator logów audytu (szkoleniowy)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/audit-log-generator.sh 20
```

- `Type=oneshot`: Jednorazowe uruchomienie (zakończenie po działaniu)
- `ExecStart`: Generuje **20 zdarzeń** przy każdym wywołaniu

### Tworzenie systemd timer

```bash
sudo tee /etc/systemd/system/audit-log-generator.timer
```

**Zawartość timer:**
```ini
[Unit]
Description=Automatyczne generowanie logów audytu (co 5 min)

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

- `OnBootSec=2min`: Pierwsze uruchomienie 2 minuty po starcie systemu
- `OnUnitActiveSec=5min`: Kolejne uruchomienia **co 5 minut**
- `timers.target`: Automatyczne uruchamianie timera przy boocie

### Aktywacja

```bash
sudo systemctl daemon-reload
sudo systemctl enable audit-log-generator.timer
sudo systemctl start audit-log-generator.timer
```

- `daemon-reload`: Przeładowanie konfiguracji systemd (nowe pliki)
- `enable`: Timer uruchamia się przy każdym boot
- `start`: Natychmiastowe uruchomienie timera

### Wynik

```
✓ Automatyczne generowanie włączone (20 zdarzeń co 5 minut)
```

Generator działa w tle: **20 zdarzeń co 5 minut = 288 zdarzeń dziennie**.

---

## Wyjaśnienie funkcji disable_auto_generation()

### Zatrzymanie timera

```bash
sudo systemctl stop audit-log-generator.timer
sudo systemctl disable audit-log-generator.timer
```

- Zatrzymuje timer (nie będzie już wywoływał generatora)
- Wyłącza autostart przy boot

### Czyszczenie plików

```bash
sudo rm -f /etc/systemd/system/audit-log-generator.service
sudo rm -f /etc/systemd/system/audit-log-generator.timer
```

Usuwa pliki service i timer z systemd.

### Przeładowanie systemd

```bash
sudo systemctl daemon-reload
```

Systemd ponownie skanuje katalogi (przestanie widzieć usunięte pliki).

### Wynik

```
✓ Automatyczne generowanie wyłączone i usunięte
```

Generator nie będzie już działał w tle. Nadal można używać ręcznie: `sudo /usr/local/bin/audit-log-generator.sh N`

---

## Użycie skryptu

### Instalacja Auditd z generatorem

```bash
# Nadanie uprawnień
chmod +x install-auditd.sh

# Instalacja
./install-auditd.sh --install
```

**Wynik:**
```
=== Instalacja Auditd ===
✓ Auditd zainstalowany i uruchomiony!

=== Instalacja generatora logów szkoleniowych ===
✓ Generator logów zainstalowany
Generowanie początkowych zdarzeń demonstracyjnych...
[SZKOLENIE] Wygenerowano dostęp do: /etc/passwd
[SZKOLENIE] Wykonano polecenie: whoami
...
✓ Wygenerowano 30 przykładowych zdarzeń!
```

### Generowanie dodatkowych logów

```bash
# Generuj 50 zdarzeń (domyślnie)
./install-auditd.sh --generate

# Generuj 200 zdarzeń
./install-auditd.sh --generate 200
```

**Użycie bezpośrednio:**
```bash
sudo /usr/local/bin/audit-log-generator.sh 100
```

### Automatyczne generowanie w tle

```bash
# Włączenie automatycznego generowania
./install-auditd.sh --enable-auto
```

**Co się dzieje:**
- Tworzy systemd service: `audit-log-generator.service`
- Tworzy systemd timer: `audit-log-generator.timer`
- Timer uruchamia generator **co 5 minut**
- Każde uruchomienie generuje **20 zdarzeń**
- Pierwsze uruchomienie 2 minuty po starcie systemu

**Wynik:**
```
✓ Automatyczne generowanie włączone (20 zdarzeń co 5 minut)

Status timera:
● audit-log-generator.timer - Automatyczne generowanie logów audytu (co 5 min)
     Loaded: loaded
     Active: active (waiting)

Następne uruchomienie:
NEXT                         LEFT          LAST PASSED  UNIT
Wed 2026-02-11 12:20:00 UTC  4min 23s left -    -       audit-log-generator.timer
```

**Sprawdzenie statusu:**
```bash
# Status timera
systemctl status audit-log-generator.timer

# Lista timerów
systemctl list-timers audit-log-generator.timer

# Historia uruchomień
journalctl -u audit-log-generator.service -n 50
```

**Wyłączenie:**
```bash
./install-auditd.sh --disable-auto
```

### Przeglądanie logów audytu

#### Wszystkie zdarzenia z dzisiaj
```bash
sudo ausearch -ts today
```

#### Zdarzenia z ostatniej godziny
```bash
sudo ausearch -ts recent
```

#### Zdarzenia związane z konkretnym kluczem
```bash
sudo ausearch -k passwd_changes
sudo ausearch -k exec_commands
sudo ausearch -k sudo_execution
```

#### Zdarzenia konkretnego użytkownika
```bash
sudo ausearch -ua szkolenie
```

#### Raport podsumowujący
```bash
sudo aureport --summary
sudo aureport --auth        # Zdarzenia autentykacji
sudo aureport --file        # Dostępy do plików
sudo aureport --executable  # Wykonane programy
```

#### Podgląd na żywo
```bash
sudo tail -f /var/log/audit/audit.log
```

---

### Diagnostyka automatycznego generowania

#### Sprawdzenie czy timer jest aktywny

```bash
systemctl is-enabled audit-log-generator.timer
systemctl is-active audit-log-generator.timer
```

**Wynik (jeśli włączony):**
```
enabled
active
```

#### Status szczegółowy timera

```bash
systemctl status audit-log-generator.timer
```

**Przykładowy wynik:**
```
● audit-log-generator.timer - Automatyczne generowanie logów audytu (co 5 min)
     Loaded: loaded (/etc/systemd/system/audit-log-generator.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since Wed 2026-02-11 12:00:00 UTC; 15min ago
    Trigger: Wed 2026-02-11 12:20:00 UTC; 4min 23s left
   Triggers: ● audit-log-generator.service

Feb 11 12:00:00 ubuntu systemd[1]: Started Automatyczne generowanie logów audytu (co 5 min).
```

**Interpretacja:**
- `Active: active (waiting)`: Timer działa, czeka na następne uruchomienie
- `Trigger: ... 4min 23s left`: Kolejne uruchomienie za 4min 23s

#### Lista wszystkich timerów

```bash
systemctl list-timers
```

Znajdź wpis `audit-log-generator.timer` z kolumnami:
- **NEXT**: Kiedy następne uruchomienie
- **LEFT**: Za ile czasu
- **LAST**: Kiedy ostatnie uruchomienie
- **PASSED**: Ile czasu temu

#### Historia uruchomień generatora

```bash
# Ostatnie 20 uruchomień
journalctl -u audit-log-generator.service -n 20

# Z datami ostatnie 2 godziny
journalctl -u audit-log-generator.service --since "2 hours ago"

# Z datami od konkretnego momentu
journalctl -u audit-log-generator.service --since "2026-02-11 10:00:00"
```

**Przykładowy log:**
```
Feb 11 12:00:03 ubuntu audit-log-generator.sh[1234]: [SZKOLENIE] Wygenerowano dostęp do: /etc/shadow
Feb 11 12:00:04 ubuntu audit-log-generator.sh[1234]: [SZKOLENIE] Wykonano polecenie: hostname
Feb 11 12:00:06 ubuntu systemd[1]: audit-log-generator.service: Deactivated successfully.
Feb 11 12:00:06 ubuntu systemd[1]: Finished Generator logów audytu (szkoleniowy).
```

#### Ręczne uruchomienie service (testowanie)

```bash
sudo systemctl start audit-log-generator.service
```

Natychmiast generuje 20 zdarzeń bez czekania na timer.

#### Zmiana częstotliwości generowania

```bash
# Edycja timera (zmiana z 5min na inną wartość)
sudo systemctl edit --full audit-log-generator.timer
```

Zmień linię `OnUnitActiveSec=5min` na:
- `OnUnitActiveSec=1min` - co minutę (intensywne)
- `OnUnitActiveSec=10min` - co 10 minut (rzadkie)
- `OnUnitActiveSec=1h` - co godzinę (bardzo rzadkie)

Po zapisaniu:
```bash
sudo systemctl daemon-reload
sudo systemctl restart audit-log-generator.timer
```

#### Rozwiązywanie problemów

**Problem: Timer nie generuje zdarzeń**

Sprawdź:
```bash
# 1. Czy timer jest aktywny
systemctl status audit-log-generator.timer

# 2. Czy generator istnieje
ls -la /usr/local/bin/audit-log-generator.sh

# 3. Czy ma uprawnienia wykonywalne
stat /usr/local/bin/audit-log-generator.sh | grep Access

# 4. Ostatni błąd service
journalctl -u audit-log-generator.service -n 1 -p err
```

**Problem: Service kończy się błędem**

```bash
# Pełne logi ostatniego uruchomienia
journalctl -u audit-log-generator.service -n 50 --no-pager

# Ręczne uruchomienie generatora (zobacz błędy)
sudo /usr/local/bin/audit-log-generator.sh 5
```

---

### Deinstalacja

```bash
./install-auditd.sh --remove
```

---

## Struktura logów Auditd

### Format wpisu w audit.log

```
type=SYSCALL msg=audit(1707654321.123:456): arch=c000003e syscall=59 success=yes exit=0 a0=... pid=1234 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=3 comm="sudo" exe="/usr/bin/sudo" key="sudo_execution"
```

**Komponenty:**
- **type**: Typ zdarzenia (SYSCALL, EXECVE, PATH, CWD)
- **msg=audit(timestamp.ms:event_id)**: Identyfikator zdarzenia
- **arch**: Architektura procesora
- **syscall**: Numer wywołania systemowego (59 = execve)
- **pid**: Process ID
- **auid**: Audit UID (rzeczywisty użytkownik, nie zmienia się przez sudo)
- **uid/gid**: User/Group ID
- **comm**: Nazwa procesu
- **exe**: Ścieżka do wykonywalnego pliku
- **key**: Klucz z reguły audytu

### Przykładowe zdarzenia generowane przez skrypt

#### Dostęp do /etc/passwd
```
type=SYSCALL msg=audit(1707654321.123:789): syscall=2 success=yes comm="cat" exe="/usr/bin/cat" key="passwd_changes"
type=PATH msg=audit(1707654321.123:789): item=0 name="/etc/passwd" inode=12345 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00
```

#### Wykonanie sudo
```
type=EXECVE msg=audit(1707654321.234:790): argc=2 a0="sudo" a1="whoami"
type=SYSCALL msg=audit(1707654321.234:790): syscall=59 success=yes exe="/usr/bin/sudo" key="sudo_execution"
```

#### Modyfikacja pliku w /tmp
```
type=SYSCALL msg=audit(1707654321.345:791): syscall=257 success=yes comm="bash" exe="/usr/bin/bash" key="tmp_directory"
type=PATH msg=audit(1707654321.345:791): item=1 name="/tmp/audit_test_1707654321.tmp" inode=67890
```

---

## Scenariusze szkoleniowe

### 1. **Detekcja podejrzanej aktywności**

Generator może symulować scenariusze ataku:

```bash
# Symuluj rekonesans systemu
sudo /usr/local/bin/audit-log-generator.sh 50
```

**Zadanie dla uczestników:**
- Znajdź wszystkie dostępy do `/etc/shadow` w ostatniej godzinie
- Zidentyfikuj użytkownika wykonującego podejrzane polecenia
- Sprawdź czy ktoś modyfikował `/etc/sudoers`

### 2. **Analiza timeline**

```bash
# Generuj zdarzenia przez dłuższy czas
for i in {1..5}; do 
	sudo /usr/local/bin/audit-log-generator.sh 20
	sleep 60
done
```

**Zadanie:**
- Stwórz timeline aktywności
- Zidentyfikuj wzorce zachowań
- Znajdź anomalie

### 3. **Korelacja z innymi logami**

Generator tworzy zdarzenia które można skorelować z:
- `/var/log/auth.log` - logowania
- `/var/log/syslog` - zdarzenia systemowe
- ELK Stack - agregacja i wizualizacja

### 4. **Odpowiedź na incydent**

Symuluj scenariusz:
1. Generator tworzy "podejrzaną" aktywność
2. Uczestnicy identyfikują anomalie
3. Tworzą raport incydentu
4. Proponują reguły mitygacji

---

## Integracja z ELK Stack

### Przesyłanie logów do Elasticsearch

Po zainstalowaniu ELK Stack można skonfigurować Filebeat do odczytu audit.log:

```bash
# Instalacja Filebeat
sudo apt-get install filebeat

# Konfiguracja
sudo cat > /etc/filebeat/filebeat.yml <<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/audit/audit.log
  fields:
    log_type: audit

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "szkolenie"
  password: "szkolenie"

setup.kibana:
  host: "localhost:5601"
EOF

# Uruchomienie
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

### Wizualizacja w Kibana

Po skonfigurowaniu Filebeat:
1. Otwórz Kibana: http://10.123.1.81:5601
2. Zaloguj się: `szkolenie` / `szkolenie`
3. Przejdź do **Management → Stack Management → Index Management**
4. Stwórz **Data View** dla `filebeat-*`
5. Przejdź do **Discover** i analizuj logi audytu

**Dashboard przykładowy:**
- Top 10 wykonywanych poleceń
- Dostępy do wrażliwych plików w czasie
- Aktywność użytkowników
- Zdarzenia sudo

---

## Troubleshooting

### Problem: Auditd nie startuje

```bash
# Sprawdź status
sudo systemctl status auditd

# Sprawdź logi
sudo journalctl -u auditd -n 50

# Sprawdź składnię reguł
sudo auditctl -l
```

### Problem: Za dużo logów

Auditd może generować **bardzo dużo** danych. Dla środowiska szkoleniowego to OK, ale w produkcji:

```bash
# Ogranicz rotację logów
sudo nano /etc/audit/auditd.conf

# Zmień:
max_log_file = 8          # MB (domyślnie 8)
num_logs = 5              # Liczba rotowanych plików
max_log_file_action = ROTATE
```

### Problem: Pełny dysk

```bash
# Sprawdź rozmiar logów
du -h /var/log/audit/

# Wyczyść stare logi (UWAGA: traci dane!)
sudo rm /var/log/audit/audit.log.*
```

### Problem: Generator nie działa

```bash
# Sprawdź czy nc (netcat) jest zainstalowany
sudo apt-get install netcat-openbsd

# Sprawdź uprawnienia
ls -la /usr/local/bin/audit-log-generator.sh
sudo chmod +x /usr/local/bin/audit-log-generator.sh

# Uruchom z debugowaniem
bash -x /usr/local/bin/audit-log-generator.sh 10
```

---

## Najlepsze praktyki

### 1. **Dla szkoleń**

✅ **Dobrze:**
- Używaj generatora do tworzenia przykładowych scenariuszy
- Oznaczaj wygenerowane zdarzenia (prefix `[SZKOLENIE]`)
- Wyczyść logi przed każdą sesją szkoleniową
- Generuj kilkadziesiąt zdarzeń naraz (50-200)

❌ **Nie:**
- Nie uruchamiaj generatora w pętli nieskończonej
- Nie zapełniaj dysku logami
- Nie mieszaj rzeczywistych zdarzeń produkcyjnych z szkoleniowymi

### 2. **Monitorowanie rozmiaru logów**

```bash
# Sprawdź rozmiar
watch -n 5 'du -h /var/log/audit/audit.log'

# Alert przy przekroczeniu 100MB
if [ $(du -m /var/log/audit/audit.log | cut -f1) -gt 100 ]; then
	echo "UWAGA: audit.log przekroczył 100MB!"
fi
```

### 3. **Analiza wydajności**

Auditd ma minimalny wpływ na wydajność, ale:

```bash
# Sprawdź czas odpowiedzi
time sudo ausearch -ts today > /dev/null

# Monitoruj obciążenie
top -p $(pgrep auditd)
```

---

## Wymagania systemowe

- **System operacyjny**: Ubuntu 20.04+ / Debian 11+
- **RAM**: +100 MB na auditd
- **Dysk**: 
  - 10 MB na instalację
  - 1-10 GB na logi (zależy od aktywności)
- **Procesor**: Minimalne obciążenie (<1%)
- **Uprawnienia**: root (sudo)

---

## Bezpieczeństwo

### Produkcja vs Szkolenie

**Środowisko szkoleniowe (ten skrypt):**
- ✅ Generator logów - OK dla demonstracji
- ✅ Proste reguły - łatwe do zrozumienia
- ⚠️  Brak zabezpieczeń integralności logów

**Środowisko produkcyjne (wymaga dodatkowej konfiguracji):**
- 🔒 **Integralność logów**: Podpisywanie/szyfrowanie audit.log
- 🔒 **Remote logging**: Wysyłanie do centralnego SIEM
- 🔒 **Immutable logs**: Ochrona przed modyfikacją (`chattr +i`)
- 🔒 **Rozbudowane reguły**: Monitoring zgodności (PCI-DSS, HIPAA)

### Ochrona logów audytu

```bash
# Ustaw immutable flag (tylko do odczytu, nawet dla roota)
sudo chattr +i /var/log/audit/audit.log

# Zdejmij immutable (np. przed rotacją)
sudo chattr -i /var/log/audit/audit.log
```

---

## Kompatybilność

Skrypt przetestowany z:
- **Ubuntu**: 20.04 LTS, 22.04 LTS, 24.04 LTS
- **Debian**: 11 (Bullseye), 12 (Bookworm)
- **Auditd**: 3.x+

---

## Co dalej podczas szkolenia?

Po zainstalowaniu i wygenerowaniu logów uczestnicy mogą:

1. **Analiza podstawowa**
   - Przeglądanie surowych logów
   - Wyszukiwanie po kluczach
   - Filtrowanie po czasie/użytkowniku

2. **Raporty i statystyki**
   - Generowanie raportów aureport
   - Identyfikacja top poleceń
   - Analiza nieudanych prób dostępu

3. **Detekcja anomalii**
   - Identyfikacja nietypowych wzorców
   - Korelacja zdarzeń
   - Tworzenie alertów

4. **Integracja z SIEM**
   - Wysyłanie do Elasticsearch
   - Wizualizacja w Kibana
   - Dashboardy i alerty

5. **Zgodność i audyt**
   - Raportowanie zgodności (PCI-DSS, SOX)
   - Ślady audytowe
   - Forensics i incident response

---

## Przydatne polecenia

### Generowanie logów
```bash
# Wygeneruj 50 zdarzeń
sudo /usr/local/bin/audit-log-generator.sh

# Wygeneruj 200 zdarzeń
sudo /usr/local/bin/audit-log-generator.sh 200

# Ciągłe generowanie (uwaga na rozmiar!)
while true; do
	sudo /usr/local/bin/audit-log-generator.sh 20
	sleep 300  # co 5 minut
done
```

### Wyszukiwanie
```bash
# Wszystkie zdarzenia z ostatnich 10 minut
sudo ausearch -ts recent

# Zdarzenia konkretnego użytkownika
sudo ausearch -ua 1000

# Zdarzenia z kluczem
sudo ausearch -k passwd_changes
sudo ausearch -k exec_commands

# Zdarzenia sukcesu/porażki
sudo ausearch --success yes
sudo ausearch --success no
```

### Raportowanie
```bash
# Raport podsumowujący
sudo aureport --summary

# Top 10 wykonanych programów
sudo aureport --executable | head -20

# Zdarzenia autentykacji
sudo aureport --auth

# Zdarzenia nieudane
sudo aureport --failed

# Raport w określonym czasie
sudo aureport --start today --end now
```

### Zarządzanie regułami
```bash
# Lista aktywnych reguł
sudo auditctl -l

# Dodaj regułę dynamicznie (do restartu)
sudo auditctl -w /etc/hosts -p wa -k hosts_changes

# Usuń konkretną regułę
sudo auditctl -W /etc/hosts -p wa -k hosts_changes

# Wyczyść wszystkie reguły (do restartu)
sudo auditctl -D
```

### Monitoring
```bash
# Status auditd
sudo auditctl -s

# Statystyki
sudo auditctl --backlog_wait_time

# Podgląd na żywo
sudo tail -f /var/log/audit/audit.log
```

---

## Podsumowanie

Skrypt automatyzuje instalację i konfigurację Auditd z dodatkowym generatorem logów:
- ✅ Instalacja auditd z podstawowymi regułami
- ✅ Generator przykładowych zdarzeń dla środowiska szkoleniowego
- ✅ 6 typów symulowanych aktywności (pliki, procesy, sieć, sudo, itp.)
- ✅ Elastyczna konfiguracja liczby generowanych zdarzeń
- ✅ Weryfikacja instalacji i podstawowe przykłady użycia
- ✅ Możliwość całkowitej deinstalacji

**Idealne dla szkoleń z:**
- Security Operations Center (SOC)
- Incident Response
- Digital Forensics
- Compliance Auditing
- SIEM (Elasticsearch/Splunk)

Generator tworzy realistyczne scenariusze bez potrzeby rzeczywistej aktywności na serwerze!
