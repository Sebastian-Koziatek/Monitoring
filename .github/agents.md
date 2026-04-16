# Instrukcje dla agenta AI - Repozytorium Monitoring

## Kontekst repozytorium

To repozytorium zawiera kompleksowe materiały szkoleniowe dotyczące **monitoringu systemów IT** z wykorzystaniem **Grafana, Prometheus, Zabbix i Loki**. Materiały są przygotowane w formacie Markdown i podzielone na moduły tematyczne.

---

## ⚠️ ZASADY PRACY Z GIT I GITHUB

**KRYTYCZNE**: Nie wolno wykonywać ŻADNYCH operacji git/GitHub bez wyraźnej prośby użytkownika:

### ZABRONIONE bez wyraźnego polecenia:
- ❌ `git push` - wypychanie zmian na GitHub
- ❌ `git commit` - commitowanie zmian
- ❌ `git add` - dodawanie plików do stage
- ❌ Tworzenie/usuwanie branchy
- ❌ Merge/rebase
- ❌ Operacje na remote repository

### WYMAGANE POSTĘPOWANIE:
1. **Po wykonaniu zmian w plikach** - ZATRZYMAJ SIĘ
2. **Zapytaj użytkownika**: "Czy mam zacommitować i wypchnąć te zmiany na GitHub?"
3. **Poczekaj na odpowiedź** - NIE zakładaj zgody
4. **Dopiero po wyraźnej zgodzie** wykonaj operacje git
5. **Nawet po zgodzie** - zapytaj o potwierdzenie przed `git push`

**Przykład prawidłowego workflow:**
- ✅ Zmieniłem pliki X, Y, Z. Czy mam zacommitować te zmiany?
- ✅ [czekam na odpowiedź]
- ✅ [po zgodzie] Czy wypchać zmiany na GitHub? (git push)

**NIGDY:**
- ❌ "Zatwierdź zmiany do git?" [i od razu robię git add/commit/push]
- ❌ Zakładanie że użytkownik chce push
- ❌ Automatyczne push po commit

---

## Struktura repozytorium

```
Monitoring/
├── Moduł 1 - Podstawy Grafany i architektura/
├── Moduł 2 - Podłączanie źródeł danych/
├── Moduł 3 - Prometheus – wprowadzenie i obsługa/
├── Moduł 4 - Podstawy Zabbix/
├── Moduł 5 - Loki – log management w Grafanie/
├── Moduł 6 - Dashboardy i wizualizacje w Grafanie/
├── Moduł 7 - Dynamiczne dashboardy i zmienne Grafany/
├── Moduł 8 - Alertowanie i powiadomienia/
├── Moduł 9 - Administracja i bezpieczeństwo w Grafanie/
├── Moduł 10 - Rozszerzenia i projekt końcowy/
└── grafiki/
```

---

## Styl pisania i formatowania

### 1. Struktura dokumentów

Każdy plik `.md` powinien zawierać:

- **Nagłówek główny H1** (`#`) - tytuł tematu
- **Wprowadzenie** - krótki opis tematyki (2-4 zdania)
- **Sekcje H2** (`##`) - podział na logiczne części
- **Podsekcje H3** (`###`) - szczegółowe zagadnienia
- **Przykłady kodu** - zawsze w blokach ````bash`, ```yaml`, ```json`, ```PromQL` itp.
- **Podsumowanie** - na końcu dokumentu (gdy pasuje)

### 2. Formatowanie kodu

#### Komendy bash/shell:
```bash
# Komentarz wyjaśniający co robi komenda
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

#### Pliki konfiguracyjne:
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

#### Zapytania PromQL:
```PromQL
# Komentarz wyjaśniający zapytanie
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[15m])) * 100)
```

**WAŻNE**: Po każdym bloku kodu dodawaj **wyjaśnienie** kluczowych elementów

### 3. Wyjaśnienia i opisy

- Używaj list wypunktowanych dla enumeracji:
  - Punkty rozpoczynaj wielką literą
  - Kończ kropką, jeśli to pełne zdanie
  - Nie dodawaj kropki po krótkich wyliczeniach

- Dla parametrów/opcji używaj list wypunktowanych z **pogrubieniem**:
  - **nazwa_parametru**: Opis działania parametru. Domyślnie: wartość.

### 4. Tabele porównawcze

Używaj tabel Markdown do porównań:

```markdown
| Prometheus | Zabbix |
|------------|--------|
| Pull-based monitoring | Push/Pull monitoring |
| PromQL | SQL-like queries |
```

### 5. Grafiki i obrazy

- Zapisuj grafiki w katalogu `grafiki/`
- Używaj formatu: `![Opis grafiki](../grafiki/nazwa-pliku.png)`
- Nazewnictwo: małe litery, myślniki zamiast spacji
- Format obrazków: PNG dla diagramów, JPG dla zrzutów ekranu

### 6. Sekcje specjalne

#### Najlepsze praktyki:
```markdown
## Najlepsze praktyki

### 1. **Optymalizacja wydajności**
Opis praktyki...

### 2. **Bezpieczeństwo**
Opis praktyki...
```

#### Ostrzeżenia/Uwagi:
```markdown
**Uwaga:** Tekst ostrzeżenia...

**WAŻNE**: Krytyczna informacja...
```

#### Przykłady konfiguracji:
```markdown
### Przykładowa konfiguracja

```yaml
# Pełny przykład
```

**Wyjaśnienie:**
- Punkt 1
- Punkt 2
```

### 7. Ćwiczenia praktyczne

Pliki z ćwiczeniami powinny zawierać:

1. **Cel ćwiczenia** - co uczestnik osiągnie
2. **Wymagania wstępne** - co jest potrzebne
3. **Kroki do wykonania** - numerowana lista
4. **Weryfikacja** - jak sprawdzić poprawność
5. **Rozwiązanie** - opcjonalne dla trudniejszych zadań

---

## Zakres tematyczny modułów

### Moduł 1-2: Podstawy Grafany
- Architektura i komponenty
- Instalacja i konfiguracja
- Źródła danych (Data Sources)

### Moduł 3: Prometheus
- Architektura Prometheusa
- **Plik konfiguracyjny** (prometheus.yml)
- Web UI - Targets, Rules, Alerts
- Eksportery (node_exporter, blackbox_exporter)
- Język zapytań PromQL

### Moduł 4: Zabbix
- Instalacja i konfiguracja
- Agenty i monitorowanie
- Integracja z Grafaną

### Moduł 5: Loki
- Log aggregation i management
- LogQL - język zapytań
- Integracja z Promtail

### Moduł 6-7: Dashboardy
- Struktura i organizacja dashboardów
- Typy wizualizacji (Time series, Gauge, Stat, Table, etc.)
- Zmienne i filtry dynamiczne

### Moduł 8: Alertowanie
- Reguły alertów
- Notification channels
- Alertmanager
- Polityki powiadomień

### Moduł 9: Administracja
- Zarządzanie użytkownikami
- Role i uprawnienia
- Bezpieczeństwo (HTTPS, OAuth)
- Backup i restore

### Moduł 10: Projekt końcowy
- Kompleksowy system monitoringu
- Best practices
- Troubleshooting

---

## Wskazówki dla agenta AI

### ✅ CO ROBIĆ:

1. **Zachowuj spójność stylu** z istniejącymi materiałami
2. **Dodawaj praktyczne przykłady** - każda teoria powinna mieć zastosowanie
3. **Wyjaśniaj szczegółowo** - materiały są dla osób uczących się
4. **Używaj komentarzy w kodzie** - każda komenda/konfiguracja powinna być zrozumiała
5. **Linkuj powiązane tematy** - wskazuj na inne moduły jeśli są powiązane
6. **Testuj komendy** - przed dodaniem komendy, upewnij się że działa (jeśli to możliwe)
7. **Dodawaj aktualne informacje** - sprawdzaj wersje oprogramowania

### ❌ CZEGO UNIKAĆ:

1. **Nie pomijaj wyjaśnień** - każdy parametr/komenda wymaga opisu
2. **Nie używaj skrótów myślowych** - materiały muszą być przystępne
3. **Nie zakładaj wiedzy** - wyjaśniaj podstawy
4. **Nie dodawaj kodu bez kontekstu** - zawsze opisz co robi
5. **Nie twórz zbyt długich dokumentów** - dziel na mniejsze sekcje
6. **Nie używaj slangu technicznego** bez wyjaśnienia

---

## Terminologia

Stosuj polskie terminy tam gdzie to możliwe, ale zachowuj angielskie nazwy własne:

- ✅ "Dashboard Grafany"
- ✅ "Źródło danych (Data Source)"
- ✅ "Panel typu Time series"
- ✅ "Język zapytań PromQL"
- ❌ "Deska rozdzielcza" (za Dashboard)
- ❌ "Seria czasowa" (używaj Time series)

---

## Środowisko testowe

W materiałach zakładamy dostęp do:

- **System operacyjny**: Ubuntu 24.04 LTS (głównie) / Debian / RHEL
- **Grafana**: wersja 10.x lub nowsza
- **Prometheus**: wersja 2.47.0 lub nowsza
- **Node Exporter**: najnowsza stabilna
- **Loki**: najnowsza stabilna
- **Serwer testowy**: Test-grafana (10.123.3.12)

---

## Sprawdzanie poprawności

Przed zakończeniem pracy nad materiałem sprawdź:

- [ ] Czy wszystkie bloki kodu mają określony język?
- [ ] Czy wszystkie parametry są wyjaśnione?
- [ ] Czy struktura jest logiczna i czytelna?
- [ ] Czy brakuje grafik do wizualizacji?
- [ ] Czy dodano przykłady praktyczne?
- [ ] Czy tekst jest zrozumiały dla osoby początkującej?
- [ ] Czy zachowano spójność z innymi modułami?

---

## Kontakt z użytkownikiem

Jeśli:
- Brakuje Ci kontekstu technicznego
- Nie jesteś pewien zakresu materiału
- Potrzebujesz dostępu do serwera testowego
- Materiał jest zbyt obszerny i wymaga podziału

→ **ZAPYTAJ użytkownika** przed kontynuowaniem pracy
