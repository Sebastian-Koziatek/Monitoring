# Dashboard i Wizualizacja

## Stwórz poniższą konfigurację
#### 1. Dashboard z wizualizacją CPU, RAM, dysku i karty sieciowej -

### Panel 1: CPU Usage (Time Series)
- **Typ wizualizacji**: Time Series
- **Nazwa:** Użycie CPU 
- **Opis**: Panel ten będzie pokazywał wykorzystanie pamięci RAM w postaci wykresu słupkowego.
- **Konfiguracja**:
  - **Źródło danych**: Zabbix
  - **Item Tag**: `cpu`
  - **Item**: `cpu utilization`

### Panel 2: RAM Usage (Bar Chart)
- **Typ wizualizacji**:: Bar Chart
- **Nazwa:** Użycie RAM
- **Opis**: Panel ten będzie pokazywał wykorzystanie pamięci RAM w postaci wykresu słupkowego.
- **Konfiguracja**:
  - **Źródło danych**: Zabbix
  - **Item Tag**: `memory`
  - **Item**: `memory utilization`

### Panel 3: Relacje między CPU, RAM, dyskiem twardym, a kartą sieciową
- **Typ wizualizacji**: Bar guage
- **Opis**: Panel ten pokazuje relacje między różnymi zasobami systemowymi:
  - **CPU Usage**: Zużycie CPU w procentach
  - **RAM Usage**: Zużycie pamięci RAM
  - **Disk Usage**: Zużycie dysku twardego
  - **Network Speed**: Prędkość pobierania i wysyłania danych z karty sieciowej
- **Konfiguracja**:
  - **Źródło danych**: Zabbix
  - **Item Tags**:
    - CPU: `cpu`
    - RAM: `memory`
    - Dysk: `disk`
    - Karta sieciowa: `network`
  - **Item**:
    - CPU: `cpu utilization`
    - RAM: `memory utilization`
    - Dysk: `FS [/]: Space: Used, in %`
    - Karta sieciowa: `Interface ens18: Bits sent`
    - Karta sieciowa `Interface ens18: Bits recived`

---

## 2. Dashboard z dokumentacją w formie strony .md

Ten dashboard będzie zawierał stronę `.md` jako część panelu, która będzie zawierała dokumentację lub informacje o monitorowanych zasobach.

### Panel 1: Strona .md
- **Typ wizualizacji**: Text Panel (Markdown)
- **Opis**: Panel będzie zawierał dokumentację w formie pliku `.md` lub tekstu markdown, który będzie wyświetlał informacje o monitorowanych zasobach systemowych, takich jak CPU, RAM, dysk twardy i karta sieciowa.
- **Konfiguracja**:
  - **Typ**: Markdown
  - **Treść**: Możesz dodać opis monitorowania zasobów, np.:

    ```
    # Dokumentacja systemu

    ## CPU
    Zużycie CPU monitorowane jest za pomocą metryki `cpu utilization`. Wartości pokazują średnie zużycie CPU w czasie.

    ## RAM
    Zużycie RAM monitorowane jest za pomocą metryki `memory utilization`. Wartości przedstawiają procentowe zużycie pamięci w systemie.

    ## Dysk
    Zużycie dysku jest monitorowane poprzez metrykę `disk usage`, która pokazuje, jaką część dostępnego miejsca na dysku zajmuje system.

    ## Karta sieciowa
    Szybkość pobierania i wysyłania danych jest monitorowana za pomocą metryki `network speed`.
    ```

## 3. Dashboard z Geolokacja

Używając bazy danych do lokalizacji, stwórz dashboard z geolokacja wyświetlającą USA w panelu.

