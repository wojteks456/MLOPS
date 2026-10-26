# Pracownia specjalistyczna 2: Uruchomienie usługi na systemie Linuksowym

## 1. Opis zadania i definicja ukończenia
* Uruchomienie gotowego modelu uczenia maszynowego na systemie Linuksowym w postaci pliku `llamafile` jako usługi `systemd`.
* Plik `llamafile` to jeden plik wykonywalny zawierający wagi modelu, silnik do jego uruchamiania oraz serwer HTTP z REST API.
* Pobranie pliku bezpośrednio na maszynę wirtualną za pomocą narzędzia `curl`.
* Uruchomienie usługi na osobnym użytkowniku systemowym (`llamafile`), a nie jako `root`.
* Konfiguracja automatycznego startu usługi wraz z systemem oraz automatycznego restartu po zabiciu procesu poleceniem `kill`.
* Ograniczenie dostępności REST API wyłącznie do adresu sieci Host-Only (`192.168.56.102:8080`) – brak dostępu z pozostałych interfejsów.

---

## 2. Kolejne kroki wykonania zadania (instrukcja odtworzenia na nowej maszynie)

### Krok 1: Sprawdzenie adresu IP interfejsu Host-Only
Sprawdzono adresację sieciową maszyny wirtualnej w celu zidentyfikowania adresu karty Host-Only (`192.168.56.102`):
```bash
ip -4 a
```

### Krok 2: Utworzenie użytkownika systemowego i katalogu docelowego
Utworzono dedykowanego użytkownika systemowego `llamafile` (bez katalogu domowego oraz bez możliwości logowania interaktywnego) oraz katalog `/opt/llamafile`:
```bash
sudo useradd --system --no-create-home llamafile
sudo mkdir -p /opt/llamafile
```

### Krok 3: Pobranie pliku `llamafile` przez `curl` i nadanie uprawnień
Zainstalowano narzędzie `curl`, pobrano gotowy model `TinyLlama-1.1B` bezpośrednio do `/opt/llamafile/model.llamafile`, nadano uprawnienia wykonywania (`+x`) oraz zmieniono właściciela plików na użytkownika `llamafile`:
```bash
sudo apt update && sudo apt install -y curl
sudo curl -L [https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q5_K_M.llamafile](https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q5_K_M.llamafile) -o /opt/llamafile/model.llamafile
sudo chmod +x /opt/llamafile/model.llamafile
sudo chown -R llamafile:llamafile /opt/llamafile
```

### Krok 4: Konfiguracja jednostki `systemd`
Utworzono plik jednostki `/etc/systemd/system/llamafile.service`:
```bash
sudo nano /etc/systemd/system/llamafile.service
```

Zawartość pliku `/etc/systemd/system/llamafile.service`:
```ini
[Unit]
Description=Llamafile LLM Service
After=network-online.target
Wants=network-online.target

[Service]
User=llamafile
Environment="HOME=/opt/llamafile"
ExecStart=/bin/sh -c "/opt/llamafile/model.llamafile --server --nobrowser --host 192.168.56.102 --port 8080"
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### Krok 5: Uruchomienie usługi, autostart i podgląd logów
Przeładowano konfigurację demona `systemd`, włączono automatyczne uruchamianie usługi po starcie maszyny i wystartowano usługę:
```bash
sudo systemctl daemon-reload
sudo systemctl enable llamafile
sudo systemctl start llamafile
```
Podgląd logów uruchamianego serwera w czasie rzeczywistym zweryfikowano poleceniem:
```bash
journalctl -u llamafile -f
```

---

## 3. Weryfikacja i dokumentacja działania usługi

### 3.1. Status usługi oraz zużycie pamięci RAM (`systemctl status` / `top`)
Sprawdzenie statusu oraz odczytanie zużycia pamięci przez usługę:
```bash
sudo systemctl status llamafile --no-pager
```

<img width="374" height="220" alt="image" src="https://github.com/user-attachments/assets/de54764d-81f0-4905-a8e5-08b5021fc4ec" />

* **Zużycie pamięci przez usługę (`Memory` / `RES` w `top`):** Usługa po załadowaniu wag modelu TinyLlama-1.1B do pamięci RAM zużywa **ok. 780 MB – 850 MB pamięci RAM** (dokładna wartość w wierszu `Memory` na powyższym zrzucie ekranu). Weryfikację w programie `top` można również wykonać komendą `top -u llamafile`.

---

### 3.2. Weryfikacja nasłuchiwania wyłącznie na adresie Host-Only (`ss -tlnp`)
Sprawdzenie otwartych gniazd sieciowych TCP:
```bash
sudo ss -tlnp
```
Wynik potwierdzający, że serwer HTTP nasłuchuje wyłącznie na adresie Host-Only (`192.168.56.102:8080`) i nie jest osiągalny przez pozostałe adresy maszyny:

<img width="610" height="151" alt="image" src="https://github.com/user-attachments/assets/cb4e1348-97e7-4405-bb95-12f0b973578a" />

---

### 3.3. Test automatycznego restartu po zabiciu procesu (`kill`)
Zweryfikowano działanie procesu na koncie użytkownika `llamafile`, a następnie celowo zabito proces poleceniem `kill`:
```bash
ps aux | grep llamafile
sudo kill -9 $(systemctl show --property MainPID --value llamafile)
sudo systemctl status llamafile --no-pager
```
Po zabiciu procesu `systemd` automatycznie uruchomił usługę ponownie (proces otrzymał nowy identyfikator PID).

---

### 3.4. Przykładowe zapytanie `curl` wraz z odpowiedzią modelu
Zapytanie wysłane do REST API na adres Host-Only (`192.168.56.102:8080`):
```bash
curl [http://192.168.56.102:8080/v1/chat/completions](http://192.168.56.102:8080/v1/chat/completions) \
-H "Content-Type: application/json" \
-d '{
  "model": "LLaMA_CPP",
  "messages": [
    {"role": "user", "content": "Czym jest system Linux? Odpowiedz w jednym zdaniu."}
  ]
}'
```

Odpowiedź zwrócona przez model (zrzut ekranu oraz pełny obiekt JSON):

<img width="914" height="238" alt="image" src="https://github.com/user-attachments/assets/4635f177-fd62-443b-957e-aa5d454c8015" />

```json
{"choices":[{"finish_reason":"stop","index":0,"message":{"content":"System Linux jest systemem operacyjnym, który jest używany w wielu urządzeniach komputerowych, takich jak notebooky, serwery, rakiety, a także na komputerach domowych. Jest to system operacyjny, który jest zbudowany z wielu programów, które są używane do zarządzania systemem i zarządzania plikami. System Linux jest używany w wielu środowiskach, takich jak oprogramowanie na rzecz systemu operacyjnego, systemy plików, aplikacje, a także systemy zarządzania plikami.</s>","role":"assistant"}}],"created":1793031756,"id":"chatcmpl-0QikmEudfEn1jOkNUmg96TKF83r7yUKk","model":"LLaMA_CPP","object":"chat.completion","usage":{"completion_tokens":148,"prompt_tokens":37,"total_tokens":185}}
```
