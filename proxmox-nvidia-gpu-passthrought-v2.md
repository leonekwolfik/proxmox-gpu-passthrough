```python
# Creating the comprehensive Markdown guide for the user based on their specific Proxmox gaming VM case.

markdown_content = """# Od zera do gamingowego VM w Proxmoxie: Przypadek AMD Ryzen 5800X + RTX 5060 Ti

## Wstęp i Specyfikacja Sprzętowa

Niniejszy poradnik opisuje proces konfiguracji w pełni wydajnej, gamingowej maszyny wirtualnej (VM) z systemem Windows 11 (Tiny11) na hiperwizerze Proxmox VE 9.x. Poradnik powstał na bazie rzeczywistych problemów i wyzwań technicznych napotkanych podczas konfiguracji specyficznej platformy AMD.

### Architektura sprzętowa hosta:
* **CPU:** AMD Ryzen AM4
* **GPU:** NVIDIA GeForce RTX 5000
* **Chipset:** AMD 500 Series (B550/X570)
* **Pamięć masowa:** Dysk NVMe (system Proxmox i LVM) + fizyczny dysk SATA 1 TB (`/dev/sda`)
* **Peryferia:** Kontroler, Mysz, Klawiature

---

## Rozdział 1: Pułapki Chipsetu AMD (Dlaczego Proxmox zamarzał?)

Podczas prób standardowego przekazania kontrolerów PCI (SATA i USB) na płytach głównych AMD z serii 500 dochodzi do tzw. **pętli zagłady** i zamrożenia serwera (komunikat `communication failure (0)` w GUI). 

Wynika to z faktu, że kontroler USB chipsetu (`02:00.0`), kontroler SATA (`02:00.1`) oraz główny mostek PCI szyny danych (`02:00.2`) dzielą domyślnie tę samą grupę IOMMU (Grupa 15). Próba odcięcia kontrolera SATA lub USB od hosta powoduje, że Proxmox odizolowuje również mostek łączący procesor z wbudowaną kartą sieciową (LAN). Skutkuje to natychmiastowym odcięciem sieci i błędem *Kernel Panic*.

Rozwiązaniem tego problemu, zastosowanym w tym poradniku, jest użycie mechanizmu **ACS Override** w celu sztucznego rozbicia grupy IOMMU oraz przekazywanie dysków jako urządzeń blokowych.

---

## Rozdział 2: Przygotowanie Hosta (Proxmox VE)

Wszystkie poniższe kroki wykonujemy z poziomu terminala (Shell) Proxmoxa jako użytkownik `root`.

### Krok 1: Włączenie IOMMU oraz ACS Override w GRUB
Musimy zmodyfikować parametry startowe jądra Linuksa, aby odblokować wirtualizację urządzeń, rozbić kłopotliwą Grupę 15 oraz zarezerwować pamięć dla Hugepages.

1. Otwórz plik konfiguracyjny GRUB:

```

```text
File successfully created: od_zera_do_gamingowego_vm_proxmox.md

```bash
   nano /etc/default/grub

```

2. Znajdź linijkę `GRUB_CMDLINE_LINUX_DEFAULT` i zmodyfikuj ją, aby wyglądała dokładnie tak:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction hugepages=12288"

```


*Znaczenie flag:*
* `amd_iommu=on` – Włącza obsługę IOMMU dla procesorów AMD.
* `iommu=pt` – Zapobiega dotykaniu przez hosta urządzeń, które mają być przekazane (tryb Passthrough).
* `pcie_acs_override=downstream,multifunction` – Rozbija współdzielone grupy IOMMU (rozdziela USB od SATA w chipsecie).
* `hugepages=12288` – Rezerwuje 12288 stron pamięci po 2MB każda (łącznie 24 GB pamięci RAM zablokowane na sztywno dla niskich opóźnień w VM).


3. Zapisz plik (`CTRL+O`, `Enter`, `CTRL+X`) i zaktualizuj konfigurację rozruchu:
```bash
update-grub

```



### Krok 2: Izolacja Karty Graficznej (Blacklist)

Musimy upewnić się, że Proxmox nie przejmie karty graficznej NVIDIA RTX 5000 dla własnych konsoli. **Krytyczne:** Nie wolno blokować sterowników USB (`xhci_hcd`), ponieważ uniemożliwi to działanie urządzeń podpiętych pod hosta.

1. Otwórz plik konfiguracyjny blokad (stworzony pod grafikę):
```bash
nano /etc/modprobe.d/blacklist-nvidia.conf

```


2. Upewnij się, że plik zawiera wyłącznie blokady dla GPU, a linie dotyczące `xhci` zostały usunięte lub zakomentowane:
```text
blacklist nouveau
blacklist nvidiafb
blacklist nvidia
# ZAKOMENTOWANE - host musi widzieć USB, aby przekazać je przez PCI!
# blacklist xhci_hcd
# blacklist xhci_pci

```


3. Zaktualizuj ramdysk startowy systemu:
```bash
update-initramfs -u

```


4. Zrestartuj cały serwer fizyczny:
```bash
reboot

```



### Krok 3: Weryfikacja podziału grup po restarcie

Po ponownym uruchomieniu sprawdź, czy ACS Override poprawnie rozbił Grupę 15:

```bash
for d in $(find /sys/kernel/iommu_groups/ -type l); do echo "${d#*/iommu_groups/}: $(basename $(readlink $d)) ($(lspci -nns $(basename $(readlink $d))))"; done | grep -E "02:00"

```

Powinieneś zobaczyć, że kontroler USB (`02:00.0`) i SATA (`02:00.1`) mają teraz **osobne numery grup** (np. Grupa 15 i Grupa 16).

---

## Rozdział 3: Konfiguracja Maszyny Wirtualnej (Hardware & Wydajność)

Konfiguracja sprzętowa w pliku `/etc/pve/qemu-server/100.conf` została zoptymalizowana pod kątem maksymalnej wydajności w grach (Bare-Metal Performance).

### 1. Optymalizacja Procesora (CPU)

* **Liczba rdzeni (`cores: 12`):** Ryzen ma 16 wątków. Przypisanie wszystkich 16 do VM powoduje mikroprzycięcia (stuttering), ponieważ host Proxmox nie ma wolnych zasobów na obsługę emulacji pamięci i dysków. Pozostawienie 4 wątków (2 rdzeni fizycznych) dla hosta całkowicie eliminuje ten problem.
* **Typ procesora (`cpu: host`):** Przekazuje instrukcje procesora 1:1, pozwalając systemowi Windows na pełne wykorzystanie architektury Zen 3.
* **NUMA (`numa: 1`):** Włączenie emulacji architektury NUMA jest bezwzględnym wymaganiem Proxmoxa do prawidłowej alokacji Hugepages.

### 2. Optymalizacja Pamięci RAM & Hugepages

Dzięki wcześniejszemu zarezerwowaniu stron w GRUB, w konfiguracji maszyny aktywujemy:

```text
hugepages: 2
memory: 24576

```

Pamięć RAM o wielkości 24 GB zostaje zmapowana w paczkach po 2 MB, drastycznie zmniejszając narzut cache procesora na translację adresów pamięci, co drastycznie podnosi minimalny FPS w grach (1% low FPS).

### 3. Migracja dysku z IDE do VirtIO SCSI (Klucz do wydajności NVMe)

Domyślne emulowane kontrolery IDE drastycznie ograniczają prędkość dysków SSD. Przeniesienie dysku na szynę VirtIO SCSI z dedykowanym wątkiem wejścia/wyjścia pozwala osiągnąć pełną prędkość fizycznego NVMe.

**Procedura migracji:**

1. Uruchom maszynę wirtualną z wpiętym obrazem ISO sterowników VirtIO (`virtio-win-*.iso`).
2. Zainstaluj w Windows sterowniki poprzez uruchomienie instalatora `virtio-win-gt-x64.exe`.
3. Wyłącz VM. W zakładce **Hardware** zaznacz dysk systemowy (np. `ide1`) i kliknij **Detach**. Dysk przejdzie na dół jako `Unused Disk`.
4. Kliknij dwukrotnie na `Unused Disk`, zmień interfejs na **SCSI**, numer gniazda na **0**.
5. W opcjach zaawansowanych zaznacz/ustaw:
* **Discard:** `Zaznaczone` (włącza obsługę TRIM, chroniąc dysk SSD przed zużyciem).
* **SSD emulation:** `Zaznaczone` (Windows optymalizuje pracę pod kątem pamięci Flash, wyłączając defragmentację).
* **IO Thread:** `Zaznaczone` (operacje dyskowe dostają dedykowany wątek CPU).
* **Cache:** `Write back` (używa pamięci RAM hosta jako ultraszybkiego bufora zapisu).


6. W zakładce **Options -> Boot Order** ustaw `scsi0` jako pierwsze urządzenie rozruchowe.

### 4. Bezpieczne przekazanie fizycznego dysku 1 TB na gry

Zamiast niebezpiecznego przekazywania całego kontrolera SATA (co zawieszało serwer), przekazujemy sam fizyczny dysk jako urządzenie blokowe bezpośrednio poleceniem CLI:

```bash
qm set 100 -scsi2 /dev/sda,ssd=1

```

Dysk pojawia się w Windows 11 jako czyste urządzenie SCSI o zerowym narzucie wydajnościowym.

---

## Rozdział 4: Passthrough Urządzeń PCI (GPU i USB)

### Krok 1: Przekazanie karty RTX 5000

W zakładce **Hardware -> Add -> PCI Device** dodajemy:

1. Kartę graficzną (`0000:08:00`) – zaznaczamy opcję **Primary GPU** oraz **PCI-Express**.
2. Dźwięk z karty graficznej (`0000:01:00`).

### Krok 2: Przekazanie Kontrolerów USB (Rozwiązanie problemu z padem GameSir)

Przekazywanie pojedynczych urządzeń USB (USB Device Passthrough) generuje ogromny input lag i blokuje niskopoziomowe zapytania, przez co aplikacje takie jak *GameSir Nexus* nie wykrywają padów. Konieczne jest przekazanie całych kontrolerów PCI.

Dodajemy dwa kontrolery w zakładce **Hardware -> Add -> PCI Device**:

1. `0000:0a:00.3` – Kontroler Matisse osadzony bezpośrednio w procesorze Ryzen (obsługuje wybrane porty z tyłu płyty).
2. `0000:02:00.0` – Kontroler z chipsetu AMD (obsługuje front panel i resztę portów, odizolowany dzięki ACS Override).

⚠️ **KRYTYCZNA REGULA DLA AMD USB:** Dla obu kontrolerów USB w oknie dodawania **MUSISZ ODZNACZYĆ opcję PCI-Express** oraz **All Functions**. Kontrolery USB od AMD emulowane jako urządzenia PCIe zgłaszają w Windows błąd *Kod 10*. Przekazanie ich jako klasyczne szyny PCI (bez flagi `pcie=1`) sprawia, że działają w 100% stabilnie.

---

## Rozdział 5: Wzorcowy plik konfiguracyjny VM (`100.conf`)

Oto jak wygląda kompletny, ostateczny plik konfiguracyjny maszyny wirtualnej zlokalizowany w `/etc/pve/qemu-server/100.conf`:

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
meta: creation-qemu=11.0.0,ctime=1778949610
name: tiny11
net0: virtio=BC:24:11:A1:DD:72,bridge=vmbr0,firewall=1
numa: 1
ostype: win11
parent: first
scsi0: local-lvm:vm-100-disk-1,cache=writeback,discard=on,iothread=1,size=48G,ssd=1
scsi2: /dev/sda,size=976762584K,ssd=1
scsihw: virtio-scsi-single
smbios1: uuid=ccd7c8f9-273e-415a-ad12-6543e0be724b
sockets: 1
tpmstate0: local-lvm:vm-100-disk-2,size=4M,version=v2.0
vga: none
vmgenid: 3a25e253-1035-4723-ae70-1a0069ba49e3

```

---

## Rozdział 6: Tweak wewnątrz Windowsa 11 (MSI Interrupts)

Po uruchomieniu Windowsa musimy przełączyć kartę graficzną RTX 5060 Ti z archaicznego trybu przerwań liniowych na tryb MSI (Message Signaled Interrupts), co całkowicie eliminuje mikroprzycięcia obrazu.

1. Pobierz bezpieczną paczkę narzędzia z GitHub (np. z repozytorium `Sathango/Msi-Utility-v3`).
2. Rozpakuj archiwum, kliknij prawym przyciskiem myszy na plik `MSI_util_v3.exe` i wybierz **Uruchom jako Administrator**.
3. Znajdź na liście pozycję **NVIDIA GeForce RTX 5060 Ti**.
4. Zaznacz pole wyboru (checkbox) w kolumnie **MSI**.
5. W kolumnie **Interrupt Priority** zmień wartość z *Undefined* na **High**.
6. Kliknij **Apply** w prawym górnym rogu.
7. Zrestartuj system Windows 11.

---

## Podsumowanie i Efekt Końcowy

Po wykonaniu powyższych kroków maszyna wirtualna osiąga pełną niezależność i stabilność:

1. Przedni panel obudowy oraz porty z tyłu działają natywnie w Windowsie z zerowym opóźnieniem.
2. Aplikacja **GameSir Nexus** bez problemu wykrywa kontroler, pozwalając na aktualizację firmware i kalibrację.
3. Dysk NVMe i fizyczny dysk WD 1 TB działają z pełną, natywną prędkością dzięki szynie VirtIO SCSI i trybowi `writeback`.
4. Brak narzutu na pamięć RAM i procesor dzięki alokacji 12 vCPU oraz Hugepages gwarantuje stabilne klatki i brak stutteringu w najbardziej wymagających grach.
"""

# Writing the markdown content to a file

file_name = "od_zera_do_gamingowego_vm_proxmox.md"
with open(file_name, "w", encoding="utf-8") as f:
f.write(markdown_content)

print(f"File successfully created: {file_name}")

```
Twoja dedykowana instrukcja w formie pliku Markdown (`.md`) jest gotowa. Uwzględniłem w niej całą naszą drogę techniczną – od kłopotów z zamrażaniem chipsetu, przez izolację grup za pomocą ACS Override i prawidłowe mapowanie specyficznych kontrolerów USB od AMD, aż po zaawansowane parametry wydajnościowe (vCPU, VirtIO SCSI, Hugepages, MSI Utility). 

Poradnik został ustrukturyzowany w profesjonalny sposób, dokładnie opisując logiczne przyczyny błędów oraz konkretne komendy i pliki konfiguracyjne z Twojego serwera, które doprowadziły nas do sukcesu.

Twój plik Markdown (MD) jest gotowy
[file-tag: code-generated-file-0-1779117390512125220]

Możesz teraz pobrać wygenerowany plik i zachować go w swoim homelabowym archiwum jako gotową dokumentację na przyszłość (np. w przypadku reinstalacji środowiska). Guide idealnie pasuje do wrzucenia na własne repozytorium GitHub czy do lokalnego notatnika (Obsidian/Notion). Powodzenia w dalszych projektach i miłego grania bez przycięć! 🎮🚀

```
