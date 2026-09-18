# Zadania z Grafany

## 1. Skasowanie źródła danych Zabbix i ponowne dodanie:
- Skasuj źródło danych Zabbix w Grafanie i spróbuj dodać je ponownie.
- Odpowiedz na poniższe pytania:
  - Jak działa pobieranie danych z wirtualnej maszyny?
  - Gdzie te dane są przechowywane?
  - Jak dane są przekazywane do Grafany?
  - Jaki plugin należy zainstalować, aby te dane dodać do źródeł danych w Grafanie?

## 2. Stworzenie nowego folderu dla zadań na dashboardzie:
- Stwórz nowy folder na zadania w dashboardzie.
- Stwórz dashboard, który wyświetli dwie tabele:
  1. **Pierwsza tabela:**
     - Użycie CPU
     - Użycie RAM
     - Użycie dysku twardego
  2. **Druga tabela:**
     - Użycie sieci – ruch przychodzący
     - Użycie sieci – ruch wychodzący

## 3. Dashboard z danymi z MySQL (Geolokalizacja):
- Stwórz dashboard, który pokaże dane z MySQL dotyczące geolokalizacji dla danych z Europy.
- Zastosuj znacznik punktowy z ikoną samolotu, aby zobrazować lokalizację.

## 4. Testowy dashboard `.md`:
- Stwórz dashboard, który będzie testem pliku `.md`.

## 5. Utworzenie playlisty:
- Utwórz playlistę zawierającą stworzone dashboardy, która będzie przełączać się co 5 minut między nimi.

## 6. Stworzenie snapshotu:
- Stwórz snapshot wybranego dashboarda.

## 7. Przegląd metryk w sekcji **Explore**:
- Wejdź do sekcji **Explore** i przejrzyj dostępne metryki z Zabbix.

## 8. Zapoznanie się z metrykami Prometheusa:
- Wejdź do sekcji **Metrics** i zapoznaj się z danymi dla Prometheusa.

## 9. Tworzenie Contact Point do Discorda:
- Stwórz **Contact Point** do Discorda, używając webhooka.

## 10. Tworzenie alertów:
- Stwórz alert na użycie CPU i sprawdź, czy będzie on działał poprawnie.

## 11. Dodanie tagu do alertu:
- Dodaj do alertu swój tag, np. `severity`.

## 12. Notification template:
- Utwórz **Notification Template** ze swoją wiadomością dla danego alertu. Możesz wygenerować wiadomość, używając języka `Go Template`.

## 13. Dodanie template do alertu:
- Dodaj template do alertu, który został utworzony, i przetestuj zmiany.

## 14. Eksportowanie dashboardów:
- Wyeksportuj jeden z utworzonych wcześniej dashboardów do pliku JSON.
- Odpowiedz na pytania:
  - W jaki sposób Grafana przechowuje definicje dashboardów?
  - Jak można zaimportować dashboard do innej instancji Grafany?

## 15. Zarządzanie użytkownikami, organizacją i zespołami:
- **Dodanie nowego użytkownika:**
  - Dodaj użytkownika z uprawnieniami Viewer i przypisz go do istniejącego zespołu.
  - Sprawdź, jakie opcje są dostępne w ustawieniach użytkownika.

- **Tworzenie zespołu:**
  - Utwórz nowy zespół o nazwie "Monitoring" i przypisz do niego kilku użytkowników.
  - Skonfiguruj domyślny folder i dashboard dla tego zespołu.

- **Zmiana ról w organizacji:**
  - Zmień rolę jednego z użytkowników z Viewer na Editor.
  - Sprawdź, jak zmieniają się jego możliwości dostępu do dashboardów.

- **Dodanie drugiej organizacji:**
  - Dodaj nową organizację w Grafanie.
  - Przypisz użytkownika do tej organizacji i ustaw jego rolę jako Admin.
  - Przełącz się między organizacjami i zobacz różnice w konfiguracji.
