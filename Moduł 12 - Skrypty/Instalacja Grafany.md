# Skrypt instalacji i deinstalacji Grafany

Automatyczny skrypt do instalacji i usuwania Grafany na systemach Ubuntu/Debian. Skrypt obsługuje pełny cykl życia aplikacji - od instalacji przez konfigurację, aż po całkowite usunięcie wraz z plikami konfiguracyjnymi.

---

## Opis skryptu

Skrypt zawiera dwie główne funkcje:
- **Instalacja Grafany** - automatyczna instalacja najnowszej wersji stabilnej z oficjalnego repozytorium
- **Deinstalacja Grafany** - całkowite usunięcie aplikacji wraz z danymi i konfiguracją

---

## Pełny skrypt

```bash
#!/bin/bash
set -e

function install_grafana() {
	# Krok 1: Instalacja wymaganych pakietów
	sudo apt-get update
	sudo apt-get install -y apt-transport-https software-properties-common wget gpg

	# Krok 2: Dodanie klucza GPG repozytorium Grafana
	sudo mkdir -p /etc/apt/keyrings/
	wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

	# Krok 3: Dodanie repozytorium Grafana OSS (stable)
	echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

	# Krok 4: Aktualizacja indeksu pakietów i instalacja Grafany
	sudo apt-get update
	sudo apt-get install -y grafana

	# Krok 5: Włączenie i uruchomienie usługi Grafana
	sudo systemctl daemon-reload
	sudo systemctl enable grafana-server
	sudo systemctl start grafana-server

	echo "Grafana została zainstalowana i uruchomiona!"
}

function remove_grafana() {
	echo "Rozpoczynam deinstalację Grafany..."
	# Zatrzymanie usługi Grafana
	sudo systemctl stop grafana-server || true
	sudo systemctl disable grafana-server || true
	sudo systemctl daemon-reload

	# Odinstalowanie pakietu Grafana
	sudo apt-get purge -y grafana
	sudo apt-get autoremove -y

	# Usunięcie repozytorium i klucza
	sudo rm -f /etc/apt/sources.list.d/grafana.list
	sudo rm -f /etc/apt/keyrings/grafana.gpg

	# Usunięcie folderów i plików konfiguracyjnych
	sudo rm -rf /etc/grafana
	sudo rm -rf /var/lib/grafana
	sudo rm -rf /var/log/grafana
	sudo rm -rf /usr/share/grafana

	echo "Grafana została całkowicie usunięta!"
}

case "$1" in
	--install)
		install_grafana
		;;
	--remove)
		remove_grafana
		;;
	*)
		echo "Użycie: $0 --install | --remove"
		exit 1
		;;
esac
```

---

## Wyjaśnienie funkcji install_grafana()

### Krok 1: Instalacja wymaganych pakietów

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common wget gpg
```

Instalowane pakiety:
- **apt-transport-https**: Umożliwia APT pobieranie pakietów przez HTTPS
- **software-properties-common**: Narzędzia do zarządzania repozytoriami
- **wget**: Program do pobierania plików z serwisów webowych
- **gpg**: Narzędzie do weryfikacji podpisów cyfrowych pakietów

### Krok 2: Dodanie klucza GPG

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

**Wyjaśnienie:**
- Tworzy katalog `/etc/apt/keyrings/` dla kluczy GPG (nowoczesna lokalizacja zgodna z Debian/Ubuntu)
- Pobiera klucz publiczny GPG z oficjalnego źródła Grafana
- Konwertuje klucz do formatu binarnego (`gpg --dearmor`)
- Zapisuje klucz w bezpiecznej lokalizacji

**Uwaga:** Klucz GPG służy do weryfikacji autentyczności pakietów z repozytorium Grafana.

### Krok 3: Dodanie repozytorium

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

**Elementy konfiguracji:**
- **deb**: Typ repozytorium (pakiety binarne)
- **signed-by**: Ścieżka do klucza GPG używanego do weryfikacji
- **https://apt.grafana.com**: Adres repozytorium Grafana
- **stable**: Gałąź ze stabilnymi wersjami (rekomendowane dla produkcji)
- **main**: Główny komponent repozytorium

### Krok 4: Instalacja pakietu

```bash
sudo apt-get update
sudo apt-get install -y grafana
```

- Aktualizacja listy pakietów z nowo dodanego repozytorium
- Instalacja najnowszej wersji Grafana OSS
- Flaga `-y` automatycznie potwierdza instalację

### Krok 5: Uruchomienie usługi

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

**Wyjaśnienie komend:**
- **daemon-reload**: Przeładowanie konfiguracji systemd (wykrycie nowej usługi)
- **enable**: Włączenie automatycznego startu Grafany po boocie systemu
- **start**: Natychmiastowe uruchomienie usługi Grafana

Po wykonaniu Grafana jest dostępna pod adresem: `http://localhost:3000`

**Domyślne dane logowania:**
- Login: `admin`
- Hasło: `admin` (przy pierwszym logowaniu system wymusi zmianę hasła)

---

## Wyjaśnienie funkcji remove_grafana()

### Zatrzymanie i wyłączenie usługi

```bash
sudo systemctl stop grafana-server || true
sudo systemctl disable grafana-server || true
sudo systemctl daemon-reload
```

**Wyjaśnienie:**
- **stop**: Zatrzymanie działającej usługi Grafana
- **disable**: Wyłączenie autostartu przy boocie systemu
- **|| true**: Operator zapewniający kontynuację skryptu nawet jeśli komenda się nie powiedzie (np. usługa już zatrzymana)
- **daemon-reload**: Aktualizacja konfiguracji systemd

### Usunięcie pakietu

```bash
sudo apt-get purge -y grafana
sudo apt-get autoremove -y
```

**Różnica między remove a purge:**
- **remove**: Usuwa pakiet, ale zostawia pliki konfiguracyjne
- **purge**: Usuwa pakiet wraz z plikami konfiguracyjnymi
- **autoremove**: Usuwa niepotrzebne już zależności

### Czyszczenie repozytorium

```bash
sudo rm -f /etc/apt/sources.list.d/grafana.list
sudo rm -f /etc/apt/keyrings/grafana.gpg
```

Usuwa wpis repozytorium oraz klucz GPG, aby system nie próbował aktualizować Grafany w przyszłości.

### Usunięcie katalogów danych

```bash
sudo rm -rf /etc/grafana          # Pliki konfiguracyjne
sudo rm -rf /var/lib/grafana      # Baza danych, dashboardy, pluginy
sudo rm -rf /var/log/grafana      # Logi aplikacji
sudo rm -rf /usr/share/grafana    # Pliki aplikacji
```

**WAŻNE**: Ta operacja jest nieodwracalna i usuwa wszystkie dane, dashboardy i ustawienia Grafany!

---

## Użycie skryptu

### Instalacja Grafany

```bash
# Nadanie uprawnień wykonywania
chmod +x install-grafana.sh

# Uruchomienie instalacji
./install-grafana.sh --install
```

### Weryfikacja instalacji

```bash
# Sprawdzenie statusu usługi
sudo systemctl status grafana-server

# Sprawdzenie wersji
grafana-server -v

# Test dostępności (powinno zwrócić kod HTML)
curl http://localhost:3000
```

### Deinstalacja Grafany

```bash
# Całkowite usunięcie Grafany
./install-grafana.sh --remove
```

**Uwaga:** Przed deinstalacją warto wykonać backup dashboardów i źródeł danych!

### Backup przed usunięciem

```bash
# Backup całego katalogu /var/lib/grafana
sudo tar -czf grafana-backup-$(date +%Y%m%d).tar.gz /var/lib/grafana/

# Backup tylko bazy danych
sudo cp /var/lib/grafana/grafana.db ~/grafana-backup.db
```

---

## Wymagania systemowe

- **System operacyjny**: Ubuntu 20.04+ / Debian 11+ / RHEL 8+
- **Uprawnienia**: root lub sudo
- **Dostęp do internetu**: Wymagany do pobrania pakietów
- **Wolne miejsce na dysku**: Minimum 500 MB

---

## Najlepsze praktyki

### 1. **Backup przed operacjami**

Zawsze wykonuj backup przed modyfikacją lub usunięciem:

```bash
# Backup konfiguracji
sudo cp -r /etc/grafana /etc/grafana.backup

# Backup danych
sudo cp -r /var/lib/grafana /var/lib/grafana.backup
```

### 2. **Weryfikacja instalacji**

Po instalacji sprawdź:

```bash
# Status usługi
sudo systemctl is-active grafana-server

# Porty nasłuchujące
sudo ss -tulpn | grep 3000

# Logi systemowe
sudo journalctl -u grafana-server -n 50
```

### 3. **Zabezpieczenie instalacji**

Po pierwszej instalacji:
- Zmień domyślne hasło administratora
- Skonfiguruj HTTPS
- Ogranicz dostęp przez firewall
- Rozważ użycie reverse proxy (nginx/Apache)

---

## Troubleshooting

### Problem: Usługa nie startuje

```bash
# Sprawdź logi
sudo journalctl -u grafana-server -xe

# Sprawdź konfigurację
sudo grafana-server -config /etc/grafana/grafana.ini -homepath /usr/share/grafana
```

### Problem: Port 3000 zajęty

```bash
# Sprawdź co używa portu
sudo lsof -i :3000

# Zmień port w konfiguracji
sudo nano /etc/grafana/grafana.ini
# Znajdź: http_port = 3000
# Zmień na: http_port = 3001
```

### Problem: Błąd klucza GPG

```bash
# Usuń stary klucz i dodaj ponownie
sudo rm -f /etc/apt/keyrings/grafana.gpg
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
sudo apt-get update
```

---

## Dodatkowe opcje instalacji

### Instalacja konkretnej wersji

```bash
# Sprawdź dostępne wersje
apt-cache madison grafana

# Zainstaluj konkretną wersję
sudo apt-get install grafana=10.2.3
```

### Instalacja Grafana Enterprise

Zmień w skrypcie linię repozytorium na:

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

I zainstaluj pakiet `grafana-enterprise`:

```bash
sudo apt-get install -y grafana-enterprise
```

---

## Podsumowanie

Skrypt automatyzuje proces instalacji i deinstalacji Grafany, zapewniając:
- ✅ Bezpieczną instalację z oficjalnego repozytorium
- ✅ Weryfikację pakietów kluczem GPG
- ✅ Automatyczną konfigurację usługi systemd
- ✅ Całkowite usunięcie bez pozostawiania śmieci w systemie

Skrypt jest odpowiedni do użycia w środowiskach testowych, development i produkcyjnych.
