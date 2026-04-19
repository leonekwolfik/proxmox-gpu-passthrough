# 🖥️ Proxmox NVIDIA GPU Passthrough — RTX 2060 / Linux & Windows

Poradnik konfiguracji passthrough dedykowanego GPU NVIDIA (RTX 2060 SUPER) w systemie **Proxmox VE** z procesorem **Intel**, **AMD**.  
Dzięki temu karta graficzna hosta może być przekazana bezpośrednio do maszyny wirtualnej z Ubuntu lub Windows.

---

## 🔧 1. Wymagania BIOS / UEFI

W BIOS włącz:
- **VT-x / VMX** — wirtualizacja procesora
- **VT-d / IOMMU** — wirtualizacja I/O (kluczowe dla passthrough)
- **Above 4G Decoding** *(opcjonalnie, zalecane)*

Zapisz zmiany i uruchom ponownie.

---

## 🧱 2. Aktualizacja Proxmox i jądra

```bash
apt update
apt dist-upgrade
reboot
```

---

## ⚙️ 3. Włącz IOMMU w bootloaderze

```bash
nano /etc/default/grub
```

Znajdź linię `GRUB_CMDLINE_LINUX_DEFAULT` i dodaj parametry:

```
# Intel
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# AMD
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

> 💡 Dla AMD zamień `intel_iommu=on` na `amd_iommu=on`

Zapisz i zaktualizuj GRUB:

```bash
update-grub
reboot
```

> 💡 Jeśli Proxmox używa `systemd-boot`, edytuj `/etc/kernel/cmdline`  
> i użyj `proxmox-boot-tool refresh` zamiast `update-grub`.

---

## 🔍 4. Sprawdzenie działania IOMMU

Po restarcie:

Intel
```bash
dmesg | grep -e DMAR -e IOMMU
```

AMD:
```bash
dmesg | grep -e AMD-Vi -e IOMMU
```

Oczekiwany wynik:
```
# Intel
DMAR: IOMMU enabled
DMAR: Intel(R) Virtualization Technology for Directed I/O
DMAR: Using Queued invalidation
DMAR-IR: Enabled IRQ remapping in x2apic mode

# AMD
AMD-Vi: IOMMU enabled
AMD-Vi: Interrupt remapping enabled
```

---

## 🧩 5. Załaduj moduły VFIO

```bash
nano /etc/modules
```

Dodaj:
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

---

## 🔍 6. Znajdź ID karty GPU

```bash
lspci -nn | grep -i nvidia
```

Przykładowy wynik dla RTX 2060 SUPER:
```
01:00.0 VGA compatible controller [10de:1f06]
01:00.1 Audio device              [10de:10f9]
01:00.2 USB controller            [10de:1ada]
01:00.3 Serial bus controller     [10de:1adb]
```

> ⚠️ Nowsze karty NVIDIA mogą mieć 4 urządzenia (GPU, Audio, USB, UCSI) — zapisz ID wszystkich.

---

## 🚫 7. Przypisz GPU do VFIO i zablokuj sterowniki

Utwórz plik konfiguracyjny VFIO:

```bash
nano /etc/modprobe.d/vfio.conf
```

Dodaj (wstaw swoje ID z kroku 6):
```
options vfio-pci ids=10de:1f06,10de:10f9,10de:1ada,10de:1adb
```

Zablokuj sterowniki NVIDIA:

```bash
nano /etc/modprobe.d/blacklist.conf
```

Dodaj:
```
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist i2c_nvidia_gpu
```

> ⚠️ Nie blacklistuj `xhci_hcd` — to ogólny sterownik USB, jego zablokowanie może wyłączyć wszystkie porty USB w systemie.

Zaktualizuj initramfs i zrestartuj:

```bash
update-initramfs -u -k all
reboot
```

---

## ✅ 8. Sprawdzenie czy VFIO przejął GPU

```bash
lspci -nnk | grep -A 3 -i nvidia
```

Oczekiwany wynik dla głównego GPU i Audio:
```
01:00.0 VGA compatible controller [10de:1f06]
        Kernel driver in use: vfio-pci
01:00.1 Audio device [10de:10f9]
        Kernel driver in use: vfio-pci
```

> 💡 `01:00.2` (USB) może nadal używać `xhci_hcd` — to normalne, można ten kontroler pominąć przy passthrough.

---

## 🖥️ 9. Konfiguracja VM w Proxmox GUI

W Proxmox GUI → wybierz VM → **Hardware** → **Add → PCI Device**

Dodaj **jeden wpis**:

| Pole | Wartość |
|------|---------|
| PCI Device | `0000:01:00.0` |
| All Functions | ✅ |
| PCI-Express | ✅ |

> ⚠️ Użyj **tylko opcji All Functions** na `01:00.0` — automatycznie przekaże wszystkie funkcje (`.0`, `.1`, `.2`, `.3`). Nie dodawaj ich osobno — spowoduje to błąd `device assigned more than once`.

Pozostałe ustawienia VM:

| Opcja | Wartość |
|-------|---------|
| CPU type | `host` |
| Display | `None` (po instalacji sterowników) |
| Machine | `i440fx` (SeaBIOS) lub `q35` (UEFI) |

---

## 🖱️ 10. Przekazanie USB (mysz, klawiatura)

Sprawdź podłączone urządzenia:

```bash
lsusb
```

W Proxmox GUI → VM → **Hardware** → **Add → USB Device** → **Use USB Vendor/Device ID**

Przykład dla Logitech Unifying Receiver (obsługuje mysz + klawiaturę):

| Urządzenie | Vendor:Device ID |
|-----------|-----------------|
| Logitech Unifying Receiver | `046d:c52b` |

> 💡 Jeden Unifying Receiver przekazuje zarówno myszkę jak i klawiaturę Logitech.  
> ⚠️ Po przekazaniu urządzenia do VM przestaje ono działać na hoście Proxmox.

---

## 🐧 11a. Ubuntu VM — instalacja sterowników NVIDIA

Po uruchomieniu VM:

```bash
sudo apt update
sudo apt install nvidia-driver-535
reboot
```

Weryfikacja:

```bash
nvidia-smi
```

Oczekiwany wynik:
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.x    Driver Version: 535.x    CUDA Version: 12.x            |
|-------------------------------+----------------------+----------------------+
| GPU  0  GeForce RTX 2060 ...  | 00000000:01:00.0 Off |                  N/A |
+-----------------------------------------------------------------------------+
```

---

## 🪟 11b. Windows VM — instalacja sterowników NVIDIA

1. Pobierz sterowniki ze strony [nvidia.com/drivers](https://www.nvidia.com/drivers)
2. Zainstaluj normalnie przez instalator `.exe`
3. Jeśli pojawi się błąd **Code 43** — upewnij się że w konfiguracji VM masz:
   - CPU type: `host,hidden=1`
   - W pliku `/etc/pve/qemu-server/<VMID>.conf` dodaj: `args: -cpu host,hidden=1`

---

## ⚠️ Typowe problemy

| Problem | Rozwiązanie |
|---------|-------------|
| `IOMMU enabled` nie pojawia się w dmesg | Włącz VT-d w BIOS |
| `Kernel driver in use: nouveau` zamiast `vfio-pci` | Sprawdź blacklist.conf i ponów `update-initramfs` |
| `device assigned more than once` | Usuń osobne wpisy `.1` `.2` `.3`, użyj tylko `All Functions` na `.0` |
| Code 43 w Windows | Dodaj `hidden=1` do CPU w konfiguracji VM |
| Brak obrazu po starcie VM | Normalny jeśli Display ustawiony na `None` — połącz się przez SSH lub VNC tymczasowo |
| USB nie działa po blackliście | Usuń `xhci_hcd` z blacklist.conf |

---

## 📚 Źródła i dokumentacja

- [Proxmox PCI Passthrough — oficjalna dokumentacja](https://pve.proxmox.com/wiki/PCI_Passthrough)
- [Proxmox Admin Guide — PCI Passthrough](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)
- [NVIDIA Driver Downloads](https://www.nvidia.com/drivers)

---

✅ **Poradnik działa dla:**
- NVIDIA GeForce RTX 2060 / 2060 SUPER / 2070 / 2080 (seria Turing TU106/TU104)
- Procesorów Intel z VT-d
- Proxmox VE 7.x / 8.x
- VM z Ubuntu (SeaBIOS) i Windows (SeaBIOS lub UEFI)
