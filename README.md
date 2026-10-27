# ThinkBook G7+ ASP EFI

OpenCore EFI для **Lenovo ThinkBook G7+ ASP** (AMD Ryzen AI, Strix Point, Radeon 880M) под **macOS Sequoia 15**.
Здесь хранятся версии EFI: каждая версия выкладывается в [Releases](../../releases) архивом `.zip`.

> [!WARNING]
> Это экспериментальная сборка для AMD-ноутбука. Она проверена на одной машине (macOS 15.8.1, загрузка с флешки).
> Держите рабочую загрузку (Linux/Windows или запасной EFI), чтобы откатиться.

## Железо

| | |
|---|---|
| Ноутбук | Lenovo ThinkBook G7+ ASP |
| CPU | AMD Ryzen AI (Strix Point, Zen 5) |
| GPU | AMD Radeon 880M (RDNA 3.5, GC 11.5.0, PCI `1002:150E`) |
| Экран | 3072x1920, 30–120 Гц (работает 120 Гц) |
| Wi-Fi / BT | MediaTek MT7925 |
| Тачпад | I2C HID `GXTP5100` (`\_SB.I2CA.TPDG`), прерывание через AMD GPIO `AMDI0030`, пин 8 |
| SMBIOS | `MacBookPro16,1` |

## Что работает

| | Статус |
|---|---|
| Загрузка macOS Sequoia 15 | ✅ |
| **Тачпад** | ✅ **только после холодной загрузки** (после перезагрузки из Windows/Linux может не определиться — выключите ноутбук полностью) |
| **USB Type-C и Type-A** | ✅ (`ItlXhciKick` убирает 2-секундные задержки мыши/клавиатуры на AMD xHCI) |
| **Wi-Fi** | ✅ в режиме «как Ethernet» (`itlmt7925`, legacy) — сеть задаётся в Info.plist кекста, см. ниже |
| Клавиатура | ✅ (VoodooPS2) |
| Батарея | ✅ (SMCBatteryManager + ECEnabler) |
| NVMe | ✅ (NVMeFix) |
| **Графика: StrixArc (эксперимент)** | ⚠️ Metal на Radeon 880M, WindowServer на GPU, родные 3K@120 (eDP-линк перетренирован на HBR3) |
| Bluetooth | ❌ |
| Звук | ❌ |
| Сон | ❌ / не проверялся |
| Внешние мониторы, подсветка экрана | ❌ |

## Экспериментальный транслятор StrixArc

В EFI включён `PhxBringup.kext` из проекта **[StrixArc](https://github.com/Asada8851/StrixArc)** — порт
[nullarc](https://github.com/luuumi-dev/nullarc) (Radeon 780M) на Strix Point. Он поднимает GPU (PSP с TMR, SMU, RLC,
MES + KIQ, очереди gfx/compute), а Metal работает поверх Vulkan (Mesa RADV).

- В EFI лежит только `PhxBringup.kext`. Остальное (PhxAccel в Auxiliary KC, Metal-плагин, загрузочный демон `phxboot`)
  ставится из macOS скриптами StrixArc — см. [README StrixArc](https://github.com/Asada8851/StrixArc/blob/main/README_RU.md).
- Кадр на экран копирует SDMA в один из двух буферов, экран переключается по vsync: 3K@120 без разрывов; eDP-линк перетренирован на HBR3.
- Аварийное выключение драйвера: boot-arg **`-phxbringupoff`** (или убрать `-phxexperimental`).
- Без установленной части StrixArc macOS работает на фреймбуфере прошивки (без ускорения).

## Особенности этой сборки

- **Модифицированный OpenCore с разделением MMIO** (`lenovo-mmio-split`): в `Booter > MmioWhitelist` оставлена в
  runtime область `0xFEC00000–0xFFFFFFFF`, плюс `DevirtualiseMmio`, `RebuildAppleMemoryMap`, `ResizeAppleGpuBars = -1`.
  Без этого Mac на этом ноутбуке не стартовал (долго разбирались с MMIO и разметкой VRAM). Стандартный `OpenCore.efi`
  сюда **не подходит**.
- **AMD Vanilla патчи** ядра (algrey и др.), `ProvideCurrentCpuInfo`, `DummyPowerManagement`; ядра в патче
  `cpuid_cores_per_package` выставлены под этот процессор.
- **Родное разрешение 3K**: `UEFI > Output > Resolution = Max`. Прежний конфиг с 1920x1200 — `EFI/OC/config-1920x1200.plist`.
- **AMFI ослаблен** (`AMFIPass`, `amfi=0x80`, `amfi_get_out_of_my_way=0x1`), **SIP выключен** (`csr-active-config = 7F080000`),
  `SecureBootModel = Disabled` — это нужно для неподписанных кекстов и Metal-плагина StrixArc.
- VRAM на этом APU — кусок обычной ОЗУ (carve-out по PA `0x780000000–0x87FFFFFFF`, в E820 reserved), BAR0 — 256 МиБ.
- Логирование OpenCore включено (`Misc > Debug > Target = 67`), логи `opencore-*.txt` пишутся в корень флешки.

### boot-args

```
keepsyms=1 debug=0x100 mtkfe=native mtknativeif=1 itlmt_m1=0 itlmt_m2=1 itlmt_m3=1 itlmt_fast=6
phxdebug=1 -phxexperimental amfi=0x80 amfi_get_out_of_my_way=0x1 -amfipassbeta ipc_control_port_options=0
```

## Состав EFI

**ACPI:** `SSDT-EC-USBX-LAPTOP`, `SSDT-PLUG-ALT`
**Drivers:** `OpenRuntime`, `OpenHfsPlus`, `ResetNvramEntry`

| Кекст | Версия | Назначение |
|---|---|---|
| Lilu | 1.7.2 | основа для плагинов |
| AMFIPass | 1.4.1 | работа с ослабленным AMFI |
| VirtualSMC | 1.3.8 | эмуляция SMC |
| SMCBatteryManager | 1.3.8 | батарея |
| ECEnabler | 1.0.6 | чтение полей EC (батарея) |
| NVMeFix | 1.1.3 | NVMe |
| RestrictEvents | 1.1.6 | косметика / ограничения процессов |
| AppleMCEReporterDisabler | — | убирает панику MCE на AMD |
| RealtekRTL8111 | 3.0.0 | Ethernet Realtek (USB/док) |
| VoodooPS2Controller (+ VoodooInput, Keyboard, Mouse, Trackpad) | 2.3.7 | клавиатура |
| VoodooI2C (+ VoodooI2CServices, VoodooGPIO) | 2.9.1 | шина I2C, AMD GPIO для прерывания тачпада |
| VoodooI2CHID | 1.0 | тачпад GXTP5100 |
| ItlXhciKick | 0.1.0 | опрос событий AMD xHCI каждую 1 мс |
| itlmt7925 | 0.1.0 | Wi-Fi MT7925 как Ethernet — **включён** |
| itlmt7925Native | 0.1.0 | родное меню Wi-Fi для 15.8.1 — выключен |
| MtkWlan / MtkWlanNative | 0.12.1 / 0.4.13 | альтернативный драйвер MT7925 — выключены |
| PhxBringup (StrixArc) | 1.0.0 | запуск GPU Radeon 880M — экспериментально |

## Установка

1. Скачайте архив из [Releases](../../releases) и распакуйте папку `EFI` в EFI-раздел флешки или диска.
2. **Сгенерируйте свои SMBIOS** (`MacBookPro16,1`) через [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) и впишите
   `MLB`, `SystemSerialNumber`, `SystemUUID`, `ROM` в `PlatformInfo > Generic` — в архиве стоят заглушки.
3. **Wi-Fi:** впишите имя сети и пароль в `EFI/OC/Kexts/itlmt7925.kext/Contents/Info.plist`
   (`itl_ssid` / `itl_psk`, есть слоты 2–4 для запасных сетей). В архиве они пустые.
4. В BIOS: выключите Secure Boot. Загружайтесь **с холодного старта**, чтобы работал тачпад.

## Планы

- [ ] Wi-Fi: родное меню Wi-Fi вместо режима «как Ethernet», стабильность и скорость
- [ ] Bluetooth (MT7925)
- [ ] Тачпад до идеала: работа после перезагрузки (не только с холодного старта), жесты, отклик
- [ ] Звук
- [ ] Сон / пробуждение
- [ ] Подсветка экрана и её регулировка
- [x] StrixArc: 120 Гц (перетренировка eDP-линка на HBR3)
- [ ] StrixArc: видео, внешние мониторы
- [ ] Карта USB-портов, веб-камера, кард-ридер, прочие мелочи

## Версии

| Версия | Что внутри |
|---|---|
| [v0.1.0](../../releases/tag/v0.1.0) | первая выложенная версия: тачпад (холодный старт), USB-C/A, Wi-Fi (legacy), StrixArc PhxBringup |

## Благодарности

[Acidanthera](https://github.com/acidanthera) (OpenCore, Lilu, VirtualSMC и др.), [AMD OS X](https://github.com/AMD-OSX)
(AMD Vanilla патчи), [VoodooI2C](https://github.com/VoodooI2C), [nullarc](https://github.com/luuumi-dev/nullarc),
авторы драйверов MT7925.
