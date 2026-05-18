# Kompleksowy Poradnik PCI/GPU Passthrough w Proxmox VE (Intel/AMD & Windows/Linux)

## Wstęp i Cel Poradnika

Niniejszy poradnik opisuje proces konfiguracji w pełni wydajnej maszyny wirtualnej (VM) z bezpośrednim dostępem do sprzętu (passthrough) dedykowanej do gamingu, uczenia maszynowego lub ciężkiej pracy stacyjnej na hiperwizerze Proxmox VE 9.x. 

Głównym celem jest eliminacja narzutu wirtualizacji i osiągnięcie wydajności rzędu 98-99% czystego komputera (bare-metal) przy zachowaniu pełnej stabilności systemu-hosta.

---

## Rozdział 1: Architektura i Pułapki Grup IOMMU

Podstawą przekazywania urządzeń PCI jest technologia IOMMU (Intel VT-d lub AMD-Vi). Pozwala ona na odizolowanie fizycznego urządzenia na płycie głównej i przypisanie go bezpośrednio do pamięci maszyny wirtualnej.

### Problem Współdzielonych Grup IOMMU
Na wielu płytach głównych (szczególnie z chipsetami konsumenckimi, np. AMD z serii 500 czy Intel z serii B/Z) linie PCIe są współdzielone. Może dojść do sytuacji, w której kontroler USB, kontroler SATA oraz zintegrowana karta sieciowa (LAN) znajdują się w jednej **Grupie IOMMU**. 
* Próba przekazania jednego z tych urządzeń bez wcześniejszego rozbicia grupy odetnie hosta Proxmox od sieci lub dysków systemowych, powodując natychmiastowy zwis serwera (`communication failure (0)` w GUI) oraz błąd *Kernel Panic*.

Rozwiązaniem tego problemu jest aktywacja mechanizmu **ACS Override**, który sztucznie wymusza separację urządzeń na poziomie jądra Linuksa.

---

## Rozdział 2: Przygotowanie Hosta (Proxmox VE)

Wszystkie kroki w tym rozdziale wykonujemy z poziomu terminala (Shell) Proxmoxa jako użytkownik `root`.

### Krok 1: Włączenie IOMMU oraz ACS Override w GRUB / systemd-boot
Musimy zmodyfikować parametry startowe jądra Linuksa w zależności od posiadanego procesora oraz menedżera rozruchu hosta.

#### A. Konfiguracja dla systemu rozruchowego GRUB (Najczęstsza)
1. Otwórz plik konfiguracyjny:
   ```bash
   nano /etc/default/grub
   ```
2. Znajdź linijkę `GRUB_CMDLINE_LINUX_DEFAULT` i zmodyfikuj ją w zależności od platformy:
   * **Dla procesorów AMD:**
     ```text
     GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction hugepages=12288"
     ```
   * **Dla procesorów Intel:**
     ```text
     GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction hugepages=12288"
     ```
3. Zapisz plik (`CTRL+O`, `Enter`, `CTRL+X`) i zaktualizuj konfigurację bootloadera:
   ```bash
   update-grub
   ```

#### B. Konfiguracja dla systemu rozruchowego systemd-boot
1. Otwórz plik konfiguracyjny:
   ```bash
   nano /etc/kernel/cmdline
   ```
2. Dopisz odpowiednie flagi na końcu istniejącej linii (w jednej linii, po spacji):
   * `amd_iommu=on` lub `intel_iommu=on`
   * `iommu=pt`
   * `pcie_acs_override=downstream,multifunction`
   * `hugepages=12288` *(Wartość 12288 rezerwuje na sztywno 24 GB pamięci RAM w postaci wielkich stron 2MB. Dostosuj tę wartość do potrzeb wirtualki: Liczba_GB * 512)*

### Krok 2: Izolacja Karty Graficznej na Hoście (Blacklist)
Musimy zablokować sterownikom Proxmoxa możliwość przejęcia karty graficznej przeznaczonej dla VM.

1. Otwórz (lub utwórz) plik czarnej listy sterowników:
   ```bash
   nano /etc/modprobe.d/pve-blacklist.conf
   ```
2. Dopisz sterowniki odpowiadające Twojej karcie graficznej:
   * **Dla kart NVIDIA:**
     ```text
     blacklist nouveau
     blacklist nvidiafb
     blacklist nvidia
     ```
   * **Dla kart AMD Radeon:**
     ```text
     blacklist radeon
     blacklist amdgpu
     ```
   * **Dla zintegrowanych kart Intel:**
     ```text
     blacklist i915
     ```
   
   ⚠️ **Krytyczna uwaga:** Jeśli w plikach konfiguracyjnych znajdują się wpisy `blacklist xhci_hcd` lub `blacklist xhci_pci`, **usuń je lub zakomentuj**. Ich zablokowanie uniemożliwi poprawne przekazywanie fizycznych kontrolerów USB do maszyn wirtualnych.

3. Zaktualizuj ramdysk startowy i zrestartuj serwer:
   ```bash
   update-initramfs -u
   reboot
   ```

### Krok 3: Weryfikacja podziału grup IOMMU
Po restarcie upewnij się, czy grupy zostały pomyślnie rozbite przez ACS Override. Wywołaj poniższy skrypt:
```bash
for d in $(find /sys/kernel/iommu_groups/ -type l); do echo "${d#*/iommu_groups/}: $(basename $(readlink $d)) ($(lspci -nns $(basename $(readlink $d))))"; done | sort -V
```
Sprawdź, czy Twoja karta graficzna (GPU) oraz docelowy kontroler USB posiadają teraz unikalne, samodzielne numery grup IOMMU.

---

## Rozdział 3: Tworzenie i Wydajnościowa Konfiguracja Maszyny Wirtualnej

Maszynę wirtualną tworzymy w GUI Proxmoxa, wybierając typ maszyny `q35` oraz BIOS `OVMF (UEFI)`. Następnie wprowadzamy kluczowe modyfikacje wydajnościowe.

### 1. Optymalizacja Procesora (CPU)
* **Liczba rdzeni/wątków (`cores`):** Nigdy nie przypisuj wszystkich fizycznych wątków procesora do maszyn wirtualnych (np. 16 wątków dla VM na procesorze 8C/16T). Zawsze pozostaw przynajmniej 2 lub 4 wątki (1-2 rdzenie) wolne dla systemu Proxmox. W przeciwnym razie host będzie dławił wirtualkę podczas obsługi operacji sieciowych i dyskowych, wywołując silny *stuttering* (mikroprzycięcia).
* **Typ procesora:** Ustaw na **`host`**. Przekazuje instrukcje procesora 1:1 do wnętrza VM.
* **NUMA:** **Włącz (ptaszek przy NUMA)**. Jest to rygorystyczne wymaganie jądra Linuksa do aktywacji Hugepages.

### 2. Aktywacja Wielkich Stron Pamięci (Hugepages)
Hugepages blokują przydzielony RAM w dużych blokach (2MB zamiast standardowych 4KB). Zapobiega to ciągłej translacji adresów pamięci przez hiperwizor i dramatycznie podnosi minimalny FPS w grach oraz aplikacjach renderujących.

Po włączeniu opcji NUMA, otwórz konfigurację danej maszyny na hoście (gdzie `ID_VM` to numer Twojej wirtualki, np. 100):
```bash
nano /etc/pve/qemu-server/ID_VM.conf
```
Dopisz w wolnej linijce:
```text
hugepages: 2
```

### 3. Optymalizacja Pamięci Masowej (Storage)
Tradycyjne szyny emulowane (SATA/IDE) ograniczają prędkość nowoczesnych dysków SSD/NVMe. Główny dysk wirtualny powinien działać na parawirtualizowanej szynie **VirtIO SCSI**.

Podczas edycji lub dodawania wirtualnego dysku zaznacz następujące opcje:
* **Discard:** `Włączone` (Zapewnia obsługę komendy **TRIM** – chroni fizyczny dysk SSD przed degradacją zapisu).
* **SSD emulation:** `Włączone` (System gościa optymalizuje harmonogram zadań pod kątem pamięci Flash).
* **IO Thread:** `Włączone` (Operacje wejścia/wyjścia dysku otrzymują dedykowany, oddzielny wątek procesora).
* **Cache:** Ustaw na **`Write back`** (Wykorzystuje wolną pamięć RAM hosta jako ultraszybki bufor zapisu danych).

---

## Rozdział 4: Passthrough Urządzeń PCI (GPU i USB)

W zakładce **Hardware -> Add -> PCI Device** maszyny wirtualnej dodajemy fizyczne komponenty.

### Krok 1: Karta Graficzna (GPU)
1. Wybierz adres swojej karty graficznej (np. `0000:08:00.0`).
2. Zaznacz opcję **Primary GPU** (jeśli ma to być jedyna/główna karta systemu, wyłączająca domyślną grafikę Proxmoxa).
3. Zaznacz opcję **PCI-Express**.
4. Zaznacz opcję **ROM-Bar** (zapewnia ładowanie vBIOS karty podczas rozruchu).
5. Dodaj jako drugie urządzenie PCI dźwięk z karty graficznej (zazwyczaj ten sam adres z końcówką `.1`).

### Krok 2: Fizyczne Kontrolery USB
Przekazywanie pojedynczych urządzeń (USB Device) wprowadza wysokie opóźnienia (*input lag*) i uniemożliwia aplikacjom producentów (np. oprogramowaniu do padów, myszek) bezpośrednią komunikację sprzętową. Prawidłowym rozwiązaniem jest przekazanie **całego kontrolera PCI USB**.

⚠️ **Złota zasada dla kontrolerów USB (AMD/Intel):** Podczas dodawania kontrolera USB jako urządzenia PCI, **MUSISZ ODZNACZYĆ opcję PCI-Express** oraz **All Functions**. Wiele zintegrowanych kontrolerów USB na poziomie sprzętowym sypie błędami inicjalizacji, gdy są emulowane na szynie PCIe systemu wirtualnego. Przekazanie ich jako standardowe, klasyczne urządzenia PCI eliminuje ten problem.

---

## Rozdział 5: Konfiguracja wewnątrz Systemu Gościa (Windows vs Linux)

W zależności od tego, jaki system instalujesz wewnątrz maszyny wirtualnej, musisz zastosować odpowiednie tweaki optymalizacyjne.

### Środowisko Windows (Guest OS)
1. **Sterowniki VirtIO:** Windows nie posiada natywnych sterowników dla szyny VirtIO SCSI. Podczas instalacji systemu musisz zamontować oficjalny obraz ISO `virtio-win.iso`, wskazać folder ze sterownikiem dysku, a po uruchomieniu zainstalować pełną paczkę narzędzi poprzez plik `virtio-win-gt-x64.exe`.
2. **Eliminacja mikroprzycięć (MSI Interrupts):** * Pobierz narzędzie **MSI Utility v3** (np. ze sprawdzonego źródła na GitHub).
   * Uruchom program jako Administrator.
   * Znajdź swoją kartę graficzną na liście, zaznacz pole wyboru w kolumnie **MSI**.
   * W kolumnie **Interrupt Priority** zmień wartość na **High**.
   * Kliknij *Apply* i zrestartuj Windowsa. Przełączy to kartę w tryb przerwań opartych na komunikatach, likwidując opóźnienia renderowania szyny PCIe.

### Środowisko Linux (Guest OS - np. Ubuntu, Arch, Pop!_OS)
1. **Brak potrzeby instalacji sterowników dysku:** Jądro Linuksa (`kernel`) posiada natywnie wbudowane i zoptymalizowane sterowniki dla urządzeń VirtIO (SCSI, Network, Serial). System zainstaluje się i uruchomi bez konieczności montowania dodatkowych płyt ISO.
2. **Przerwania MSI:** Linux automatycznie i natywnie konfiguruje nowoczesne karty graficzne w tryb MSI/MSI-X. Nie potrzebujesz żadnych zewnętrznych narzędzi do konfiguracji przerwań.
3. **Zarządzanie energią procesora (CPU Governor):**
   Wewnątrz wirtualnego Linuksa domyślny zarządca procesora może próbować oszczędzać energię, co obniża wydajność w grach. Wymuś tryb maksymalnej wydajności:
   ```bash
   sudo apt install cpufrequtils   # Dla systemów Debian/Ubuntu
   echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
   ```
4. **Wyłączenie kompozytora okien (Dla graczy):**
   Środowiska graficzne Linuksa (GNOME, KDE, XFCE) używają kompozycji obrazu, co generuje lekki input lag.
   * Jeśli używasz systemu **X11**, włącz opcję "Allow applications to block compositing" w ustawieniach ekranu lub wyłączaj kompozytor skrótem klawiszowym (np. `Shift + Alt + F12` w KDE) przed uruchomieniem gry.
   * Jeśli używasz **Waylanda**, mechanizm ten jest zoptymalizowany, ale upewnij się, że gra uruchamia się w trybie *Exclusive Fullscreen*.

---

## Rozdział 6: Przekazywanie Dodatkowych Fizycznych Dysków

Jeśli posiadasz oddzielny, fizyczny dysk SSD/HDD (SATA lub NVMe) przeznaczony wyłącznie na dane lub gry, najwydajniejszą metodą jest przekazanie go jako surowe urządzenie blokowe z poziomu CLI hosta Proxmox.

Wpisz w Shellu Proxmoxa:
```bash
qm set ID_VM -scsi2 /dev/sdX,ssd=1
```
*Gdzie:*
* `ID_VM` – Numer maszyny wirtualnej.
* `/dev/sdX` – Ścieżka do fizycznego dysku na hoście (np. `/dev/sda` lub `/dev/nvme1n1`).
* `ssd=1` – Flaga informująca system gościa (zarówno Windows, jak i Linux), że urządzenie jest dyskiem półprzewodnikowym.

---

## Rozdział 7: Wzorcowy Plik Konfiguracyjny Maszyny (`/etc/pve/qemu-server/ID_VM.conf`)

Oto jak powinien wyglądać docelowy, czysty i w pełni zoptymalizowany plik konfiguracyjny maszyny wirtualnej w Proxmoxie:

```text
agent: 1
args: -cpu host,kvm=off
bios: ovmf
boot: order=scsi0;net0
cores: 12
cpu: host
efidisk0: local-lvm:vm-100-disk-0,efitype=4m,ms-cert=2023k,pre-enrolled-keys=1,size=4M
hostpci0: 0000:08:00,pcie=1
hostpci1: 0000:01:00
hostpci2: 0000:02:00.0
hostpci3: 0000:0a:00.3
hugepages: 2
machine: pc-q35-11.0
memory: 24576
name: gaming-station
net0: virtio=BC:24:11:A1:DD:72,bridge=vmbr0,firewall=1
numa: 1
ostype: win11
scsi0: local-lvm:vm-100-disk-1,cache=writeback,discard=on,iothread=1,size=48G,ssd=1
scsi2: /dev/sda,size=976762584K,ssd=1
scsihw: virtio-scsi-single
sockets: 1
tpmstate0: local-lvm:vm-100-disk-2,size=4M,version=v2.0
vga: none
```
*(Uwaga: W przypadku maszyny z Linuksem, jedyną różnicą w tym pliku będzie wartość parametru `ostype: l26`, odpowiadająca jądru Linuksa serii 2.6/3.x/4.x/5.x/6.x).*