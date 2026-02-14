# Skrypt instalacji i deinstalacji ELK Stack

Automatyczny skrypt do instalacji i usuwania ELK Stack (Elasticsearch + Kibana) na systemach Ubuntu/Debian. Skrypt instaluje najnowsze stabilne wersje z oficjalnych repozytoriów Elastic, **włącza security i generuje hasła**, konfiguruje usługi systemd i uruchamia komponenty gotowe do pracy z pełną funkcjonalnością Fleet/Integrations.

---

## Opis skryptu

Skrypt zawiera dwie główne funkcje:
- **Instalacja ELK Stack** - automatyczna instalacja Elasticsearch i Kibana z włączonym security
- **Deinstalacja ELK Stack** - całkowite usunięcie aplikacji wraz z danymi i konfiguracją

**Co jest instalowane:**
- **Elasticsearch** - silnik wyszukiwania i analityki danych (port 9200) **z security**
- **Kibana** - interfejs webowy do wizualizacji i zarządzania (port 5601) **z encryption keys**

**Kluczowe cechy:**
- ✅ X-Pack Security włączony (autentykacja, autoryzacja)
- ✅ Automatyczne generowanie haseł dla użytkowników
- ✅ Encryption keys dla Fleet/Integrations
- ✅ SSL wyłączony dla uproszczenia (środowisko szkoleniowe)
- ✅ Fleet/Integrations w pełni funkcjonalne

---

## Pełny skrypt

```bash
#!/bin/bash
set -e

ELASTIC_VERSION="8.x"

function install_elasticsearch() {
	echo "=== Instalacja Elasticsearch ==="
	
	# Krok 1: Instalacja wymaganych pakietów
	sudo apt-get update
	sudo apt-get install -y apt-transport-https wget gpg
	
	# Krok 2: Dodanie klucza GPG Elastic
	wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
	
	# Krok 3: Dodanie repozytorium Elastic
	echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/${ELASTIC_VERSION}/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-${ELASTIC_VERSION}.list
	
	# Krok 4: Instalacja Elasticsearch
	sudo apt-get update
	sudo apt-get install -y elasticsearch
	
	# Krok 5: Konfiguracja z włączonym security
	sudo tee /etc/elasticsearch/elasticsearch.yml > /dev/null <<EOF
# Konfiguracja dla środowiska szkoleniowego z włączonym security
cluster.name: elasticsearch
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node

# Security - włączone dla Fleet/Integrations
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# Wyłączamy SSL dla uproszczenia (środowisko szkoleniowe)
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
EOF
	
	# Krok 6: Uruchomienie Elasticsearch
	sudo systemctl daemon-reload
	sudo systemctl enable elasticsearch
	sudo systemctl start elasticsearch
	
	echo "✓ Elasticsearch zainstalowany!"
	echo "  Czekam 30 sekund na uruchomienie..."
	sleep 30
	
	# Krok 7: Generowanie haseł dla użytkowników
	echo "=== Konfiguracja użytkowników Elasticsearch ==="
	
	# Generujemy hasło dla elastic (admin backup)
	sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b > /tmp/elastic-pass.txt 2>&1
	ELASTIC_PASSWORD=$(grep "New value:" /tmp/elastic-pass.txt | awk '{print $3}')
	
	# Generujemy hasło dla kibana_system (połączenie Kibana->ES)
	sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b > /tmp/kibana-pass.txt 2>&1
	KIBANA_PASSWORD=$(grep "New value:" /tmp/kibana-pass.txt | awk '{print $3}')
	
	# Czekamy aż Elasticsearch będzie gotowy do przyjęcia API calls
	echo "Czekam na gotowość Elasticsearch API..."
	sleep 10
	
	# Tworzymy użytkownika "szkolenie" z hasłem "szkolenie" (dla uczestników)
	echo "Tworzenie użytkownika szkoleniowego..."
	curl -X POST "http://localhost:9200/_security/user/szkolenie" \
	  -u "elastic:$ELASTIC_PASSWORD" \
	  -H "Content-Type: application/json" \
	  -d '{
	    "password": "szkolenie",
	    "roles": ["superuser"],
	    "full_name": "Użytkownik Szkoleniowy"
	  }' 2>/dev/null
	
	echo ""
	echo "✓ Użytkownicy skonfigurowani!"
	echo ""
	echo "=========================================="
	echo "DANE LOGOWANIA DLA UCZESTNIKÓW:"
	echo "=========================================="
	echo "Login:    szkolenie"
	echo "Hasło:    szkolenie"
	echo "=========================================="
	echo ""
	echo "Hasło administratora elastic: $ELASTIC_PASSWORD"
	echo "(zapisz w bezpiecznym miejscu)"
	echo ""
	
	# Zapisanie hasła elastic do pliku tymczasowego
	echo "$ELASTIC_PASSWORD" | sudo tee /tmp/elastic-admin-pass.txt > /dev/null
	echo "$KIBANA_PASSWORD" | sudo tee /tmp/kibana-system-pass.txt > /dev/null
}

function install_kibana() {
	echo "=== Instalacja Kibana ==="
	
	# Krok 1: Instalacja Kibana (repozytorium już dodane przez Elasticsearch)
	sudo apt-get install -y kibana
	
	# Krok 2: Pobranie hasła Kibana z pliku tymczasowego
	KIBANA_PASSWORD=$(sudo cat /tmp/kibana-system-pass.txt)
	
	# Krok 3: Generowanie encryption key dla Kibana (wymagane dla Fleet)
	ENCRYPTION_KEY=$(openssl rand -hex 32)
	
	# Krok 4: Konfiguracja z security
	sudo tee /etc/kibana/kibana.yml > /dev/null <<EOF
# Konfiguracja dla środowiska szkoleniowego z security
server.port: 5601
server.host: "0.0.0.0"
server.publicBaseUrl: "http://\$(hostname -I | awk '{print \$1}'):5601"

# Połączenie z Elasticsearch (z autentykacją)
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "$KIBANA_PASSWORD"

# Encryption keys dla Fleet i Saved Objects (wymagane!)
xpack.encryptedSavedObjects.encryptionKey: "$ENCRYPTION_KEY"
xpack.security.encryptionKey: "$ENCRYPTION_KEY"
xpack.reporting.encryptionKey: "$ENCRYPTION_KEY"
EOF
	
	# Krok 5: Uruchomienie Kibana
	sudo systemctl daemon-reload
	sudo systemctl enable kibana
	sudo systemctl start kibana
	
	echo "✓ Kibana zainstalowana!"
	echo "  Czekam 30 sekund na uruchomienie..."
	sleep 30
}

function remove_elk() {
	echo "=== Deinstalacja ELK Stack ==="
	
	# Zatrzymanie usług
	sudo systemctl stop kibana || true
	sudo systemctl stop elasticsearch || true
	sudo systemctl disable kibana || true
	sudo systemctl disable elasticsearch || true
	
	# Odinstalowanie pakietów
	sudo apt-get purge -y elasticsearch kibana
	sudo apt-get autoremove -y
	
	# Usunięcie repozytoriów i kluczy
	sudo rm -f /etc/apt/sources.list.d/elastic-*.list
	sudo rm -f /usr/share/keyrings/elasticsearch-keyring.gpg
	
	# Usunięcie danych i konfiguracji
	sudo rm -rf /etc/elasticsearch
	sudo rm -rf /var/lib/elasticsearch
	sudo rm -rf /var/log/elasticsearch
	sudo rm -rf /etc/kibana
	sudo rm -rf /var/lib/kibana
	sudo rm -rf /var/log/kibana
	
	echo "✓ ELK Stack został całkowicie usunięty!"
}

function verify_installation() {
	echo ""
	echo "=== Weryfikacja instalacji ==="
	
	# Sprawdzenie Elasticsearch z użytkownikiem szkoleniowym
	echo -n "Elasticsearch: "
	if curl -s -u szkolenie:szkolenie http://localhost:9200 > /dev/null 2>&1; then
		echo "✓ DZIAŁA"
		curl -s -u szkolenie:szkolenie http://localhost:9200 | grep "cluster_name"
	else
		echo "✗ NIE ODPOWIADA"
	fi
	
	# Sprawdzenie Kibana
	echo -n "Kibana: "
	if curl -s http://localhost:5601/api/status > /dev/null 2>&1; then
		echo "✓ DZIAŁA"
	else
		echo "✗ NIE ODPOWIADA (może potrzebować więcej czasu)"
	fi
	
	echo ""
	echo "=== Status usług ==="
	sudo systemctl status elasticsearch --no-pager -l | head -5
	sudo systemctl status kibana --no-pager -l | head -5
}

case "$1" in
	--install)
		install_elasticsearch
		install_kibana
		verify_installation
		
		SERVER_IP=$(hostname -I | awk '{print $1}')
		
		echo ""
		echo "==================================================="
		echo "           ELK Stack gotowy do użycia!            "
		echo "==================================================="
		echo ""
		echo "Elasticsearch: http://$SERVER_IP:9200"
		echo "Kibana:        http://$SERVER_IP:5601"
		echo ""
		echo "==================================================="
		echo "        DANE LOGOWANIA DLA UCZESTNIKÓW:           "
		echo "==================================================="
		echo "    Login:    szkolenie"
		echo "    Hasło:    szkolenie"
		echo "==================================================="
		echo ""
		echo "UWAGA: Kibana może potrzebować 1-2 minuty na pełne uruchomienie"
		echo ""
		;;
	--remove)
		remove_elk
		;;
	*)
		echo "Użycie: $0 --install | --remove"
		exit 1
		;;
esac
```

---

## Wyjaśnienie funkcji install_elasticsearch()

### Krok 1: Instalacja wymaganych pakietów

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https wget gpg
```

Analogicznie jak przy instalacji Prometheus - potrzebne są narzędzia do obsługi HTTPS i weryfikacji GPG.

### Krok 2: Dodanie klucza GPG

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

**Wyjaśnienie:**
- Pobiera oficjalny klucz publiczny GPG od Elastic
- Konwertuje do formatu binarnego (`--dearmor`)
- Zapisuje w bezpiecznej lokalizacji `/usr/share/keyrings/`

### Krok 3: Dodanie repozytorium

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/${ELASTIC_VERSION}/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-${ELASTIC_VERSION}.list
```

**Parametry:**
- **deb**: Typ repozytorium (pakiety binarne)
- **signed-by**: Wskazuje klucz GPG do weryfikacji pakietów
- **${ELASTIC_VERSION}**: Zmienna określająca wersję (8.x)
- **stable**: Stabilna gałąź pakietów

### Krok 4: Instalacja Elasticsearch

```bash
sudo apt-get update
sudo apt-get install -y elasticsearch
```

Instaluje najnowszą wersję Elasticsearch 8.x z repozytorium.

### Krok 5: Konfiguracja z włączonym security

```bash
sudo tee /etc/elasticsearch/elasticsearch.yml > /dev/null <<EOF
cluster.name: elasticsearch
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node

# Security - włączone dla Fleet/Integrations
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# Wyłączamy SSL dla uproszczenia (środowisko szkoleniowe)
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
EOF
```

**Kluczowe zmiany:**
- **xpack.security.enabled: true**: Włączamy security - **wymagane** przez Fleet/Integrations w Kibanie
- **xpack.security.enrollment.enabled: true**: Włączamy możliwość rejestracji
- **xpack.security.http.ssl.enabled: false**: Wyłączamy SSL dla HTTP (uproszczenie dla szkoleń)
- **xpack.security.transport.ssl.enabled: false**: Wyłączamy SSL dla komunikacji inter-node

**UWAGA:** W środowisku produkcyjnym SSL powinien być włączony! To ustawienie tylko dla szkoleń.

### Krok 6: Uruchomienie Elasticsearch

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
sleep 30
```

Elasticsearch potrzebuje ~30 sekund na pełne uruchomienie i zainicjowanie security.

### Krok 7: Generowanie haseł i tworzenie użytkownika szkoleniowego

```bash
# Generowanie haseł dla użytkowników systemowych
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b > /tmp/elastic-pass.txt 2>&1
ELASTIC_PASSWORD=$(grep "New value:" /tmp/elastic-pass.txt | awk '{print $3}')

sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b > /tmp/kibana-pass.txt 2>&1
KIBANA_PASSWORD=$(grep "New value:" /tmp/kibana-pass.txt | awk '{print $3}')

# Tworzenie użytkownika szkoleniowego przez API
curl -X POST "http://localhost:9200/_security/user/szkolenie" \
  -u "elastic:$ELASTIC_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "szkolenie",
    "roles": ["superuser"],
    "full_name": "Użytkownik Szkoleniowy"
  }'
```

**Wyjaśnienie:**

**Trzej użytkownicy w systemie:**

1. **elastic** - administrator z pełnymi uprawnieniami
   - Hasło: automatycznie wygenerowane (losowe, bezpieczne)
   - Przeznaczenie: backup admin, awaryjny dostęp
   - Używany tylko przez trenera

2. **kibana_system** - użytkownik techniczny
   - Hasło: automatycznie wygenerowane
   - Przeznaczenie: wewnętrzne połączenie Kibana → Elasticsearch
   - NIE służy do logowania użytkowników

3. **szkolenie** - użytkownik dla uczestników szkolenia
   - Login: `szkolenie`
   - Hasło: `szkolenie`
   - Przeznaczenie: proste logowanie dla uczestników
   - Rola: `superuser` (pełne uprawnienia)

**Dlaczego proste hasło dla środowiska szkoleniowego?**
- Łatwe do zapamiętania
- Nie trzeba dyktować skomplikowanych haseł
- Uczestnicy mogą od razu zacząć pracę
- **UWAGA:** W produkcji użyj silnych haseł!

**API do tworzenia użytkownika:**
- Endpoint: `/_security/user/szkolenie`
- Metoda: POST
- Autentykacja: używamy wygenerowanego hasła użytkownika `elastic`
- Parametry:
  - `password`: proste hasło "szkolenie"
  - `roles`: ["superuser"] - pełne uprawnienia
  - `full_name`: opis użytkownika

### Zapisanie haseł

```bash
echo "$ELASTIC_PASSWORD" | sudo tee /tmp/elastic-admin-pass.txt > /dev/null
echo "$KIBANA_PASSWORD" | sudo tee /tmp/kibana-system-pass.txt > /dev/null
```

Zapisujemy hasła do plików tymczasowych w `/tmp/`:
- **elastic-admin-pass.txt**: Hasło administratora (dla awaryjnego dostępu)
- **kibana-system-pass.txt**: Hasło dla użytkownika `kibana_system` (do połączenia Kibana→ES)

**Dlaczego `/tmp/` a nie `/etc/kibana/`?**
- W tym momencie pakiet Kibana NIE JEST JESZCZE ZAINSTALOWANY
- Katalog `/etc/kibana/` nie istnieje (powstanie dopiero po `apt-get install kibana`)
- Gdybyśmy teraz próbowali zapisać do `/etc/kibana/`, dostalibyśmy błąd:
  ```
  tee: /etc/kibana/kibana-password.txt: No such file or directory
  ```
- Hasło zostanie skopiowane do właściwej lokalizacji w funkcji `install_kibana()` po instalacji pakietu

---

## Wyjaśnienie funkcji install_kibana()

### Instalacja Kibana

```bash
sudo apt-get install -y kibana
```

Używa tego samego repozytorium co Elasticsearch (już dodanego wcześniej).

### Pobranie hasła Kibana

```bash
KIBANA_PASSWORD=$(sudo cat /tmp/kibana-system-pass.txt)
```

Odczytujemy hasło użytkownika `kibana_system` z pliku tymczasowego utworzonego wcześniej przez funkcję `install_elasticsearch()`.

**UWAGA:** Nie możemy odczytać tego z `/etc/kibana/` w funkcji `install_elasticsearch()`, bo ten katalog nie istnieje przed instalacją pakietu Kibana. Dlatego używamy `/tmp/`.

### Generowanie encryption key

```bash
ENCRYPTION_KEY=$(openssl rand -hex 32)
```

Generujemy losowy 64-znakowy klucz szyfrowania (32 bajty hex). Jest **wymagany** przez:
- **Fleet** - zarządzanie agentami Elastic
- **Encrypted Saved Objects** - szyfrowanie wrażliwych danych w Kibanie
- **Reporting** - generowanie raportów

**Bez tego klucza Fleet/Integrations nie będą działać!**

### Konfiguracja Kibana z security

```bash
sudo tee /etc/kibana/kibana.yml > /dev/null <<EOF
server.port: 5601
server.host: "0.0.0.0"
server.publicBaseUrl: "http://$(hostname -I | awk '{print $1}'):5601"

elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "$KIBANA_PASSWORD"

xpack.encryptedSavedObjects.encryptionKey: "$ENCRYPTION_KEY"
xpack.security.encryptionKey: "$ENCRYPTION_KEY"
xpack.reporting.encryptionKey: "$ENCRYPTION_KEY"
EOF
```

**Parametry konfiguracji:**
- **server.port: 5601**: Port interfejsu webowego Kibany
- **server.host: "0.0.0.0"**: Kibana nasłuchuje na wszystkich interfejsach
- **server.publicBaseUrl**: Publiczny adres URL Kibany (automatycznie wykrywa IP serwera)
- **elasticsearch.hosts**: Adres Elasticsearch (HTTP bez SSL dla uproszczenia)
- **elasticsearch.username: "kibana_system"**: Użytkownik do wewnętrznej komunikacji
- **elasticsearch.password**: Hasło wygenerowane w kroku instalacji Elasticsearch
- **xpack.encryptedSavedObjects.encryptionKey**: Klucz dla szyfrowanych obiektów (Fleet!)
- **xpack.security.encryptionKey**: Klucz security
- **xpack.reporting.encryptionKey**: Klucz dla raportów

**UWAGA:** Wszystkie trzy klucze encryption mogą być takie same w środowisku testowym. W produkcji powinny być różne.

### Uruchomienie Kibana

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana
sleep 30
```

Kibana również potrzebuje czasu na uruchomienie - buduje cache, inicjalizuje połączenie z Elasticsearch.

---

## Wyjaśnienie funkcji remove_elk()

### Zatrzymanie usług

```bash
sudo systemctl stop kibana || true
sudo systemctl stop elasticsearch || true
sudo systemctl disable kibana || true
sudo systemctl disable elasticsearch || true
```

Kolejność jest ważna:
1. Najpierw Kibana (klient)
2. Potem Elasticsearch (serwer)

### Usunięcie pakietów

```bash
sudo apt-get purge -y elasticsearch kibana
sudo apt-get autoremove -y
```

`purge` usuwa pakiety wraz z plikami konfiguracyjnymi.

### Czyszczenie repozytoriów

```bash
sudo rm -f /etc/apt/sources.list.d/elastic-*.list
sudo rm -f /usr/share/keyrings/elasticsearch-keyring.gpg
```

Usuwa wpisy repozytorium i klucze GPG.

### Usunięcie danych

```bash
sudo rm -rf /etc/elasticsearch      # Konfiguracja Elasticsearch
sudo rm -rf /var/lib/elasticsearch  # Dane (indeksy, dokumenty)
sudo rm -rf /var/log/elasticsearch  # Logi
sudo rm -rf /etc/kibana            # Konfiguracja Kibana
sudo rm -rf /var/lib/kibana        # Dane Kibana (saved objects)
sudo rm -rf /var/log/kibana        # Logi Kibana
```

**UWAGA:** To nieodwracalnie usuwa wszystkie dane! Przed deinstalacją warto wykonać backup.

---

## Wyjaśnienie funkcji verify_installation()

### Test Elasticsearch

```bash
curl -s http://localhost:9200
```

Sprawdza czy Elasticsearch odpowiada na API. Powinien zwrócić JSON z informacjami o klastrze:

```json
{
  "name" : "hostname",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xxx",
  "version" : {
    "number" : "8.x.x"
  },
  "tagline" : "You Know, for Search"
}
```

### Test Kibana

```bash
curl -s http://localhost:5601/api/status
```

Sprawdza status API Kibana. Zwraca JSON z informacją o stanie aplikacji.

---

## Użycie skryptu

### Instalacja ELK Stack

```bash
# Nadanie uprawnień wykonywania
chmod +x install-elk.sh

# Uruchomienie instalacji
./install-elk.sh --install
```

**Czas instalacji:** ~5-10 minut (zależnie od prędkości pobierania i mocy serwera)

**WAŻNE:** Skrypt wyświetli dane logowania na końcu instalacji:

```
=================================================
        DANE LOGOWANIA DLA UCZESTNIKÓW:           
=================================================
    Login:    szkolenie
    Hasło:    szkolenie
=================================================
```

**Proste dane dla środowiska szkoleniowego!** Uczestnicy mogą od razu zacząć pracę.

### Weryfikacja instalacji

#### Test Elasticsearch (przez API)

```bash
# Użyj prostych danych logowania
curl -u szkolenie:szkolenie http://localhost:9200
```

Przykładowa odpowiedź:
```json
{
  "name" : "monitoring",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "8.12.0"
  },
  "tagline" : "You Know, for Search"
}
```

#### Test Kibana (przeglądarka)

1. Otwórz: `http://adres_serwera:5601`
2. Zaloguj się:
   - **Username:** `szkolenie`
   - **Password:** `szkolenie`
3. Po zalogowaniu zobaczysz dashboard Kibany

**Pierwsze kroki w Kibana:**

1. **Fleet/Integrations działa!** - Teraz możesz przejść do: Management → Fleet → Integrations
2. **Dev Tools:** Management → Dev Tools - testuj zapytania do Elasticsearch
3. **Discover:** Przeglądaj dane i logi
4. **Dashboard:** Twórz wizualizacje

**Przykładowe zapytania w Dev Tools (w Kibanie):**

```json
GET /

GET _cluster/health

GET _cat/indices?v

PUT /test-index
{
  "settings": {
    "number_of_shards": 1
  }
}
```

**UWAGA:** W Dev Tools NIE musisz podawać autentykacji - jesteś już zalogowany przez interfejs webowy Kibany.

### Deinstalacja ELK Stack

```bash
./install-elk.sh --remove
```

---

## Wymagania systemowe

- **System operacyjny**: Ubuntu 20.04+ / Debian 11+
- **RAM**: Minimum 4 GB (zalecane 8 GB)
  - Elasticsearch: ~2 GB RAM
  - Kibana: ~1 GB RAM
- **Procesor**: Minimum 2 rdzenie
- **Dysk**: Minimum 10 GB wolnego miejsca
- **Dostęp do internetu**: Wymagany do pobrania pakietów
- **Porty**: 9200 (Elasticsearch), 5601 (Kibana)

**UWAGA:** Elasticsearch wymaga więcej zasobów niż Prometheus/Grafana!

---

## Najlepsze praktyki

### 1. **Weryfikacja po instalacji**

```bash
# Status usług
sudo systemctl status elasticsearch
sudo systemctl status kibana

# Logi w przypadku problemów
sudo journalctl -u elasticsearch -n 50
sudo journalctl -u kibana -n 50

# Test API (używając hasła elastic)
curl -u elastic:HASŁO http://localhost:9200/_cluster/health?pretty
```

### 2. **Sprawdzenie użycia zasobów**

```bash
# Sprawdź zużycie pamięci przez Elasticsearch
ps aux | grep elasticsearch

# Sprawdź użycie pamięci
free -h

# Sprawdź miejsce na dysku
df -h /var/lib/elasticsearch
```

### 3. **Monitoring zdrowia klastra**

```bash
# Zdrowie klastra
curl -u szkolenie:szkolenie http://localhost:9200/_cluster/health?pretty

# Lista węzłów
curl -u szkolenie:szkolenie http://localhost:9200/_cat/nodes?v

# Lista indeksów
curl -u szkolenie:szkolenie http://localhost:9200/_cat/indices?v
```

---

## Troubleshooting

### Problem: Elasticsearch nie startuje

**Diagnoza:**
```bash
sudo journalctl -u elasticsearch -n 100
```

**Częste przyczyny:**

#### 1. Za mało pamięci RAM

```
ERROR: [1] bootstrap checks failed
[1]: max virtual memory areas vm.max_map_count [65530] is too low
```

**Rozwiązanie:**
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo systemctl restart elasticsearch
```

#### 2. Port zajęty

```bash
# Sprawdź co używa portu 9200
sudo lsof -i :9200

# Zmień port w konfiguracji
sudo nano /etc/elasticsearch/elasticsearch.yml
# http.port: 9201
```

#### 3. Błędy uprawnień

```bash
# Napraw uprawnienia
sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch
sudo systemctl restart elasticsearch
```

### Problem: Kibana nie łączy się z Elasticsearch

**Objaw:** Kibana pokazuje "Kibana server is not ready yet"

**Diagnoza:**
```bash
sudo journalctl -u kibana -n 100
```

**Sprawdź:**

#### 1. Czy Elasticsearch działa?

```bash
# Użyj danych szkoleniowych
curl -u szkolenie:szkolenie http://localhost:9200
```

#### 2. Czy hasło w Kibanie jest poprawne?

```bash
# Sprawdź hasło zapisane dla Kibany
sudo cat /etc/kibana/kibana-password.txt

# Sprawdź konfigurację Kibany
sudo grep "elasticsearch.password" /etc/kibana/kibana.yml
```

**Rozwiązanie - Reset hasła:**
```bash
# Wygeneruj nowe hasło dla kibana_system
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b

# Zapisz nowe hasło do konfiguracji Kibany
sudo nano /etc/kibana/kibana.yml
# Zaktualizuj linię: elasticsearch.password: "NOWE_HASŁO"

# Restart Kibany
sudo systemctl restart kibana
```

### Problem: Nie mogę się zalogować do Kibany

**Objaw:** "Incorrect username or password"

**Rozwiązanie 1 - Sprawdź dane:**
```bash
# Upewnij się że używasz poprawnych danych
# Login: szkolenie
# Hasło: szkolenie
```

**Rozwiązanie 2 - Reset hasła użytkownika szkolenie:**
```bash
# Pobierz hasło elastic (admin)
ELASTIC_PASS=$(sudo cat /tmp/elastic-admin-pass.txt)

# Zresetuj hasło użytkownika szkolenie
curl -X PUT "http://localhost:9200/_security/user/szkolenie/_password" \
  -u "elastic:$ELASTIC_PASS" \
  -H "Content-Type: application/json" \
  -d '{"password": "szkolenie"}'
```

**Rozwiązanie 3 - Użyj administratora (elastic):**
```bash
# Jeśli użytkownik szkolenie nie działa, użyj elastic
# Hasło znajduje się w pliku
sudo cat /tmp/elastic-admin-pass.txt
```

### Problem: Fleet/Integrations nie ładuje listy

**Objaw:** Spinner ładowania bez końca, komunikat "Kibana security must be enabled"

**Przyczyna:** Brak encryption keys w konfiguracji Kibany

**Rozwiązanie:**
```bash
# Wygeneruj encryption key
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Dodaj do konfiguracji Kibany
sudo tee -a /etc/kibana/kibana.yml > /dev/null <<EOF

# Encryption keys dla Fleet
xpack.encryptedSavedObjects.encryptionKey: "$ENCRYPTION_KEY"
xpack.security.encryptionKey: "$ENCRYPTION_KEY"
xpack.reporting.encryptionKey: "$ENCRYPTION_KEY"
EOF

# Restart Kibany
sudo systemctl restart kibana
```

### Problem: Zapomniałem hasła

**Dla użytkownika szkolenie:**
Hasło jest proste i stałe: `szkolenie`

**Dla użytkownika elastic (administrator):**
```bash
# Hasło zapisane w pliku (jeśli instalacja była niedawno)
sudo cat /tmp/elastic-admin-pass.txt

# Jeśli plik nie istnieje - wygeneruj nowe hasło
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b
```

### Problem: Chcę zmienić hasło użytkownika "szkolenie"

```bash
# Pobierz hasło elastic
ELASTIC_PASS=$(sudo cat /tmp/elastic-admin-pass.txt)

# Zmień hasło użytkownika szkolenie (np. na "nowe_haslo")
curl -X PUT "http://localhost:9200/_security/user/szkolenie/_password" \
  -u "elastic:$ELASTIC_PASS" \
  -H "Content-Type: application/json" \
  -d '{"password": "nowe_haslo"}'
```

### Bezpieczeństwo haseł

**Środowisko szkoleniowe:**
- Login: `szkolenie` / Hasło: `szkolenie` - proste i łatwe dla uczestników
- To jest OK dla środowisk zamkniętych, nieprodukcyjnych

**UWAGA dla produkcji:**
- NIGDY nie używaj prostych haseł typu "szkolenie"
- Używaj silnych, losowych haseł (jak generowane automatycznie)
- Włącz SSL/TLS dla szyfrowanej komunikacji
- Ogranicz dostęp sieciowy (firewall, VPN)
- Używaj mechanizmów secrets management (Vault, AWS Secrets Manager)

**Usuwanie plików z hasłami:**
```bash
# Po zapisaniu haseł w bezpiecznym miejscu usuń pliki tymczasowe
sudo rm -f /tmp/elastic-pass.txt
sudo rm -f /tmp/kibana-pass.txt
sudo rm -f /tmp/elastic-admin-pass.txt
```

### Problem: Kibana długo się uruchamia

Kibana może potrzebować 2-3 minuty na pełne uruchomienie. To normalne przy pierwszym starcie.

```bash
# Obserwuj logi
sudo journalctl -u kibana -f

# Poczekaj na komunikat:
# "http server running at http://0.0.0.0:5601"
```

### Problem: Elasticsearch zużywa za dużo pamięci

**Domyślnie Elasticsearch używa 50% dostępnej pamięci RAM.**

**Ograniczenie pamięci:**
```bash
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```

Dodaj:
```
-Xms1g
-Xmx1g
```

To ustawia heap na 1GB. Dostosuj do swoich potrzeb (nie więcej niż 50% dostępnej RAM).

```bash
sudo systemctl restart elasticsearch
```

---

## Struktura katalogów

### Elasticsearch

```
/etc/elasticsearch/          # Konfiguracja
/var/lib/elasticsearch/      # Dane (indeksy)
/var/log/elasticsearch/      # Logi
/usr/share/elasticsearch/    # Pliki aplikacji
```

### Kibana

```
/etc/kibana/                 # Konfiguracja
/var/lib/kibana/             # Dane (saved objects, visualizations)
/var/log/kibana/             # Logi
/usr/share/kibana/           # Pliki aplikacji
```

---

## Backup i restore

### Backup konfiguracji

```bash
# Backup konfiguracji
sudo tar -czf elk-config-backup-$(date +%Y%m%d).tar.gz \
  /etc/elasticsearch \
  /etc/kibana

# Backup danych Elasticsearch (wymaga konfiguracji snapshot repository)
curl -X PUT "localhost:9200/_snapshot/my_backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/backup/elasticsearch"
  }
}'
```

---

## Co dalej podczas szkolenia?

Po zainstalowaniu ELK Stack uczestnicy mogą:

1. **Korzystać z Fleet/Integrations**
   - Dodawanie integracji (Nginx, Apache, System logs)
   - Zarządzanie Elastic Agents
   - Automatyczne zbieranie danych

2. **Tworzyć indeksy i dodawać dokumenty**
   - Nauka struktury danych w Elasticsearch
   - Operacje CRUD przez API
   - Mapping i typy danych

3. **Konfigurować Kibana**
   - Tworzenie Index Patterns/Data Views
   - Budowanie wizualizacji (wykresy, mapy)
   - Tworzenie interaktywnych dashboardów

4. **Integrować z innymi systemami**
   - Logstash - pipeline przetwarzania logów
   - Beats - lightweight data shippers
   - Prometheus/Grafana - korelacja metryk i logów

5. **Zarządzanie bezpieczeństwem**
   - Tworzenie użytkowników i ról
   - Kontrola dostępu do indeksów
   - Audit logs

6. **Łączyć Elasticsearch z Grafaną**
   - Elasticsearch jako Data Source w Grafanie
   - Korelacja metryk i logów w jednym dashboardzie
   - Alerting na podstawie danych z Elasticsearch

---

## Kompatybilność wersji

Skrypt został przetestowany z:
- **Elasticsearch**: 8.12.0+
- **Kibana**: 8.12.0+
- **Ubuntu**: 22.04 LTS, 24.04 LTS
- **Debian**: 11, 12

**UWAGA:** Wersje Elasticsearch i Kibana muszą być **identyczne** lub różnić się tylko patch version (np. 8.12.0 + 8.12.1).

---

## Różnice między wersjami

### Elasticsearch 7.x vs 8.x

- **8.x**: Domyślnie włączone zabezpieczenia (TLS, hasła) - **wymagane przez Fleet**
- **7.x**: Zabezpieczenia wyłączone domyślnie

**Ten skrypt (Elasticsearch 8.x):**
- Włącza security (`xpack.security.enabled: true`)
- Generuje hasła automatycznie
- Wyłącza SSL dla uproszczenia (środowisko szkoleniowe)
- Fleet/Integrations w pełni funkcjonalne

**UWAGA dla produkcji:** W środowisku produkcyjnym należy dodatkowo włączyć SSL!

---

## Podsumowanie

Skrypt automatyzuje instalację ELK Stack z włączonym security:
- ✅ Instalacja Elasticsearch z oficjalnych źródeł
- ✅ Włączenie X-Pack Security (wymagane przez Fleet)
- ✅ Automatyczne generowanie haseł dla użytkowników
- ✅ Instalacja Kibana z encryption keys
- ✅ Automatyczna konfiguracja połączenia między komponentami
- ✅ Fleet/Integrations w pełni funkcjonalne
- ✅ Wyłączenie SSL dla uproszczenia (środowisko szkoleniowe)
- ✅ Weryfikacja poprawności instalacji
- ✅ Możliwość całkowitej deinstalacji

**Różnica od wersji bez security:**
- **Fleet/Integrations:** Teraz działają! Możesz dodawać integracje dla różnych źródeł danych
- **Autentykacja:** Musisz się logować (user: elastic + wygenerowane hasło)
- **Produkcyjne podejście:** Skrypt przygotowuje środowisko bardziej zbliżone do produkcji

Idealne rozwiązanie do przygotowania środowiska szkoleniowego ELK Stack z pełną funkcjonalnością Fleet.
