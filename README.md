# 💻 Proxmox iGPU Passthrough — Intel N100 / N355 (Alder Lake-N / Twin Lake)

Poradnik konfiguracji passthrough zintegrowanego GPU (iGPU) dla procesorów Intel N100 i N355 w systemie **Proxmox VE**.  
Dzięki temu GPU hosta może być przekazane bezpośrednio do maszyny wirtualnej (np. Windows 10 lub 11).

---

## 🔧 1. Wymagania BIOS / UEFI

W BIOS włącz:
- **VT-x / VMX**
- **VT-d / IOMMU**
- **ACS / ARI Support** *(jeśli dostępne)*
- **Above 4G Decoding** *(opcjonalnie, zalecane)*
- Zostaw włączoną zintegrowaną grafikę (iGPU).

Zapisz zmiany i uruchom ponownie.

---

## 🧱 2. Aktualizacja Proxmox i jądra

```bash
uname -r
apt update
apt dist-upgrade
reboot
uname -r
```

---

## 💿 3. Utworzenie VM z Windows

1. Wgraj ISO instalacyjne Windows i ISO sterowników VirtIO:
   - [VirtIO ISO (ze strony Proxmox)](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers#Using_the_ISO)
2. Ustawienia VM:
   - **System**
     - Machine: `q35`
     - BIOS: `OVMF (UEFI)`
     - Graphic Card: `Default`
     - QEMU Agent: ✓
     - Dodaj **TPM 2.0**
   - **Dysk**
     - Typ: `SCSI` (VirtIO SCSI)
     - `Discard` + `SSD emulation`
     - Rozmiar: np. `100G`
   - **CPU**
     - 4 rdzenie, typ: `Host`
   - **RAM**
     - np. `12G`, wyłącz ballooning

3. Uruchom instalator Windows:
   - Załaduj sterowniki VirtIO (SCSI).
   - Po instalacji Windows zaktualizuj brakujące urządzenia w Menedżerze urządzeń.
   - Wyłącz VM.

---

## ⚙️ 4. Włącz IOMMU w systemie

Edytuj plik `/etc/default/grub`:

```bash
nano /etc/default/grub
```

Dodaj w linii `GRUB_CMDLINE_LINUX_DEFAULT`:

```
intel_iommu=on iommu=pt intel_pstate=disable
```

Zapisz i zaktualizuj GRUB:

```bash
update-grub
reboot
```

> 💡 Jeśli Twój Proxmox używa `systemd-boot`, edytuj `/etc/kernel/cmdline`  
> i użyj `proxmox-boot-tool refresh` zamiast `update-grub`.

---

## 🔍 5. Sprawdzenie działania IOMMU

Po restarcie uruchom:

```bash
dmesg | grep -e DMAR -e IOMMU
```

Oczekiwany wynik:
```
DMAR: IOMMU enabled
DMAR: Intel(R) Virtualization Technology for Directed I/O
DMAR: Using Queued invalidation
DMAR-IR: Enabled IRQ remapping in x2apic mode
```

Dodatkowo sprawdź obsługę remappingu:
```bash
dmesg | grep remapping
```

---

## 🧩 6. Sprawdzenie grup IOMMU

```bash
pvesh get /nodes/{nodename}/hardware/pci --pci-class-blacklist ""
```

Znajdź urządzenie GPU (`00:02.0`).  
Musi znajdować się w osobnej grupie IOMMU lub przynajmniej bez krytycznych urządzeń.

---

## 🚫 7. Zablokowanie sterownika `i915`

```bash
echo "blacklist i915" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u -k all
reboot
```

Po restarcie:
```bash
lspci -nnk | grep -A 2 VGA
```
Powinno być:
```
Kernel driver in use: vfio-pci
```

---

## 🧱 8. Pobranie i dodanie pliku ROM

Dla Twojego CPU użyj odpowiedniego ROM-u:

### ✅ Intel N100
```bash
wget https://github.com/lixiaoliu666/intel6-14rom/releases/download/v2.0-20250622-100999/12-n100-q10.rom
cp 12-n100-q10.rom /usr/share/kvm/
chmod 644 /usr/share/kvm/12-n100-q10.rom
```

### ✅ Intel N355
```bash
wget https://github.com/LongQT-sea/intel-igpu-passthru/releases/download/v0.1/ADL-N_TWL_GOPv21_igd.rom
cp ADL-N_TWL_GOPv21_igd.rom /usr/share/kvm/
chmod 644 /usr/share/kvm/ADL-N_TWL_GOPv21_igd.rom
```

---

## 🧩 9. Modyfikacja pliku VM `.conf`

Otwórz konfigurację VM:

```bash
cd /etc/pve/qemu-server
nano <VMID>.conf
```

Dodaj poniższe (dopasuj nazwę pliku ROM do CPU):

```text
args: -set device.hostpci0.bus=pcie.0       -set device.hostpci0.addr=0x02.0       -set device.hostpci0.x-igd-gms=0x6       -set device.hostpci0.x-igd-opregion=on

cpu: host,hidden=1

hostpci0: 0000:00:02.0,pcie=1,x-vga=1,romfile=/usr/share/kvm/<nazwa_romu>.rom
```

---

## 🧷 10. (Opcjonalnie) Przekazanie urządzeń USB

```bash
lsusb
qm set <VMID> -usb0 host=<device_id>
```

---

## ▶️ 11. Uruchomienie i testy

- Uruchom VM — host może utracić obraz (to normalne).  
- Sprawdź w Windows Menedżerze urządzeń — GPU powinno być widoczne.  
- Jeśli występuje błąd **Code 43** — upewnij się, że `hidden=1` i ROM jest zgodny.  
- Możesz zmieniać wartość `x-igd-gms` w zależności od ustawień DVMT w BIOS  
  (np. `0x2`, `0x4`, `0x6`, `0xa`).

---

## ⚠️ 12. Uwagi i typowe problemy

- iGPU passthrough dla serii **Alder Lake-N / Twin Lake** może działać różnie w zależności od płyty głównej i BIOS.  
- Jeśli host nie ma drugiej karty graficznej, ekran po starcie VM będzie czarny — połączenie przez sieć SSH / web UI.  
- ROM musi być zgodny z konkretną wersją iGPU (inne wersje GOP mogą powodować błędy lub czarny ekran).  
- Po aktualizacji jądra Proxmoxa czasem trzeba ponownie wykonać blacklist i `update-initramfs`.  
- Jeśli GPU nadal jest używane przez i915, można wymusić przełączenie:
  ```bash
  echo "options vfio-pci ids=8086:46d3" >> /etc/modprobe.d/vfio.conf
  update-initramfs -u -k all
  reboot
  ```
- **[VM z Linuxem]** Jeśli chcesz przekazać GPU do maszyny wirtulnej z Linuxem (np. do TrueNAS Scale) to dodawanie pliku ROM oraz modyfikacja pliku .conf nie są konieczne.

---

## 📚 Źródła i dokumentacja

- [Proxmox PCI Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
- [Proxmox Admin Guide – PCI Passthrough](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)
- [automation-avenue / proxmox-igpu-passthrough-n100](https://github.com/automation-avenue/proxmox-igpu-passthrough-n100)
- [LongQT-sea / intel-igpu-passthru](https://github.com/LongQT-sea/intel-igpu-passthru)
- [lixiaoliu666 / intel n100 rom](https://github.com/lixiaoliu666/intel6-14rom/releases/download/v2.0-20250622-100999/12-n100-q10.rom)
- [Automation Avenue / Proxmox GPU pasthrought tutorial](https://www.youtube.com/watch?v=Pjjyuk4YU78&t)
- [Automation Avenue / Run Windows 11 VM on Proxmox](https://www.youtube.com/watch?v=2zqqzMDorMw)

---

✅ **Poradnik działa dla:**
- Intel N100 (Alder Lake-N)
- Intel N355 (Twin Lake)
- Innych procesorów z iGPU Intel UHD / Xe-LP generacji 12–14 (po dopasowaniu ROM)
