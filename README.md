# Pracownia specjalistyczna 2: Uruchomienie usługi na systemie Linuksowym

## 1. Opis zadania
Uruchomienie gotowego modelu językowego (`llamafile`) jako usługi systemowej `systemd` działającej na osobnym użytkowniku systemowym (`llamafile`) oraz udostępnienie interfejsu HTTP REST API wyłącznie przez adres sieci Host-Only (`192.168.56.102:8080`).

---

## 2. Kolejne kroki wykonania zadania (instrukcja odtworzenia)

### Krok 1: Utworzenie użytkownika systemowego i katalogu docelowego
Utworzono użytkownika systemowego `llamafile` bez katalogu domowego oraz katalog `/opt/llamafile`:
```bash
sudo useradd --system --no-create-home llamafile
sudo mkdir -p /opt/llamafile
```

### Krok 2: Pobranie pliku `llamafile` i nadanie uprawnień
Zainstalowano narzędzie `curl`, pobrano gotowy model `TinyLlama-1.1B` bezpośrednio na maszynę wirtualną, nadano uprawnienia wykonywania oraz zmieniono właściciela plików:
```bash
sudo apt update && sudo apt install -y curl
sudo curl -L [https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q5_K_M.llamafile](https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q5_K_M.llamafile) -o /opt/llamafile/model.llamafile
sudo chmod +x /opt/llamafile/model.llamafile
sudo chown -R llamafile:llamafile /opt/llamafile
```

### Krok 3: Konfiguracja jednostki `systemd`
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

### Krok 4: Uruchomienie usługi i włączenie autostartu
Przeładowano demona `systemd`, włączono automatyczne uruchamianie usługi po starcie maszyny i wystartowano usługę:
```bash
sudo systemctl daemon-reload
sudo systemctl enable llamafile
sudo systemctl start llamafile
```

---

## 3. Weryfikacja i dokumentacja działania usługi

### 3.1. Status usługi oraz zużycie pamięci (`systemctl status`)
Polecenie:
```bash
sudo systemctl status llamafile --no-pager
```
Wynik (zużycie pamięci widoczne w wierszu `Memory`):
```text
[WKLEJ TUTAJ WYNIK Z SYSTEMCTL STATUS]
```

### 3.2. Weryfikacja nasłuchiwania wyłącznie na adresie Host-Only (`ss -tlnp`)
Polecenie:
```bash
sudo ss -tlnp
```
Wynik potwierdzający nasłuchiwanie wyłącznie na `192.168.56.102:8080`:
```text
[WKLEJ TUTAJ WYNIK Z SS -TLNP]
```

### 3.3. Test automatycznego restartu po zabiciu procesu (`kill`)
Wyszukano PID procesu, zakończono go sygnałem `kill -9` i zweryfikowano ponowne automatyczne uruchomienie przez `systemd`:
```bash
ps aux | grep llamafile
sudo kill -9 $(systemctl show --property MainPID --value llamafile)
sudo systemctl status llamafile --no-pager
```
Usługa automatycznie podniosła się po 3 sekundach z nowym numerem PID.

### 3.4. Przykładowe zapytanie `curl` wraz z odpowiedzią modelu
Zapytanie wysłane na adres Host-Only (`192.168.56.102:8080`):
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

Odpowiedź modelu:
```json
{"choices":[{"finish_reason":"stop","index":0,"message":{"content":"System Linux jest systemem operacyjnym, który jest używany w wielu urządzeniach komputerowych, takich jak notebooky, serwery, rakiety, a także na komputerach domowych. Jest to system operacyjny, który jest zbudowany z wielu programów, które są używane do zarządzania systemem i zarządzania plikami. System Linux jest używany w wielu środowiskach, takich jak oprogramowanie na rzecz systemu operacyjnego, systemy plików, aplikacje, a także systemy zarządzania plikami.</s>","role":"assistant"}}],"created":1793031756,"id":"chatcmpl-0QikmEudfEn1jOkNUmg96TKF83r7yUKk","model":"LLaMA_CPP","object":"chat.completion","usage":{"completion_tokens":148,"prompt_tokens":37,"total_tokens":185}}
```
