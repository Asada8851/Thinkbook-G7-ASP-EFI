# ThinkBook G7+ ASP EFI

OpenCore EFI для **Lenovo ThinkBook G7+ ASP** (AMD Ryzen AI, Strix Point, Radeon 880M) под **macOS Sequoia 15**.
Здесь хранятся версии EFI: каждая версия выкладывается в [Releases](../../releases) архивом `.zip`.

> [!WARNING]
> Это экспериментальная сборка для AMD-ноутбука. Она проверена на одной машине (macOS 15.8.1 24H32, загрузка с флешки).
> Держите рабочую загрузку (Linux/Windows или запасной EFI), чтобы откатиться.

## Железо

| | |
|---|---|
| Ноутбук | Lenovo ThinkBook G7+ ASP |
| CPU | AMD Ryzen AI (Strix Point, Zen 5) |
| GPU | AMD Radeon 880M (RDNA 3.5, GC 11.5.0, PCI `1002:150E`) |
| Экран | 3072x1920, 30–120 Гц (работает 120 Гц) |
| Wi-Fi / BT | MediaTek MT7925 (Wi-Fi `14C3:7925`, Bluetooth USB `0489:E111`) |
| Звук | Realtek ALC257 на AMD HD Audio `1022:15E3` (`PciRoot(0x0)/Pci(0x8,0x1)/Pci(0x0,0x6)`) |
| Тачпад | I2C HID `GXTP5100` (`\_SB.I2CA.TPDG`), прерывание через AMD GPIO `AMDI0030`, пин 8 |
| SMBIOS | `MacBookPro16,1` |

## Что работает

| | Статус |
|---|---|
| Загрузка macOS Sequoia 15 | ✅ |
| **Тачпад** | ✅ **только после холодной загрузки** (после перезагрузки из Windows/Linux может не определиться — выключите ноутбук полностью) |
| **USB Type-C и Type-A** | ✅ (`ItlXhciKick` убирает 2-секундные задержки мыши/клавиатуры на AMD xHCI) |
| **Wi-Fi** | ✅ **родное меню Wi-Fi**: список сетей, подключение из меню (WPA2), сохранённые сети, автоподключение, DHCP (`MtkWlan` + порт `MtkWlanNative` под 15.8.1) |
| **Bluetooth** | ✅ сопряжение и подключение устройств: клавиатура (Magic Keyboard, с уровнем заряда), наушники со звуком (`MtkBt` + `BlueToolFixup`) |
| **Звук** | ✅ динамики через `AppleALC` (layout 11), звук в Bluetooth-наушники |
| Клавиатура | ✅ (VoodooPS2) |
| Батарея | ✅ (SMCBatteryManager + ECEnabler) |
| NVMe | ✅ (NVMeFix) |
| **Графика: StrixArc (эксперимент)** | ⚠️ Metal на Radeon 880M, WindowServer на GPU, родные 3K@120 (eDP-линк перетренирован на HBR3) |
| Встроенный микрофон | ❌ цифровой микрофон подключён к AMD Audio Coprocessor, драйвера для macOS нет |
| Наушники в разъём, микрофон гарнитуры | ❓ не проверялись |
| AirDrop / Continuity | ❌ (у MT7925 нет AWDL) |
| Сон | ❌ / не проверялся |
| Внешние мониторы, подсветка экрана | ❌ |

## Wi-Fi и Bluetooth (MediaTek MT7925)

- **Wi-Fi:** драйвер [MtkWlan](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi) 0.12.1 и родное меню
  `MtkWlanNative` 0.4.13. Оригинальный
  [MtkWlanNative](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi-native) собран только под macOS 26.7.1
  (25G241) и на Sequoia не загружается. **В этом EFI лежит порт под macOS 15.8.1 (24H32)**: заголовки ABI
  IO80211/Skywalk пересобраны с ядра 24H32, раскладка событий и запросов взята из дизассемблера 24H32 и
  [ga403wm-hackintosh](https://github.com/thanhnndev/ga403wm-hackintosh). Пока есть подключение, радио сканирует не чаще
  раза в минуту: macOS просит скан каждые ~5 с, пока открыто меню Wi-Fi, и полный обход каналов ронял связь.
  Работает **только на 15.8.1 (24H32)**: на другой сборке kext сам отказывается загружаться, и MtkWlan переходит в
  режим «как Ethernet». Исходники порта будут опубликованы после согласования с автором оригинала.
- **Bluetooth:** [MtkBt](https://github.com/watchthelight-labs/mt7925-hackintosh-bluetooth) 0.2.4 загружает прошивку,
  дальше работает родной `bluetoothd` с `BlueToolFixup`. Нужные NVRAM-ключи (`bluetoothExternalDongleFailed`,
  `bluetoothInternalControllerInfo`) уже прописаны в `config.plist`. Кнопка «Подключить» у iPhone, как и на настоящем
  Mac, ничего не подключает, пока на телефоне не включён режим модема.
- Прежний драйвер `itlmt7925` (Wi-Fi «как Ethernet») остался в EFI выключенным — как запасной вариант.
- Лицензии этих кекстов лежат в архиве рядом с папкой `EFI`, в `LICENSES-kexts/`.

## Звук

`AppleALC` 1.9.8, кодек ALC257, **layout задан boot-arg'ом `alcid=11`**. `layout-id` из `DeviceProperties` до
контроллера на этом ноутбуке не доходил, и без layout AppleHDA не поднимался; `alcid` задаёт его напрямую.
Если звук работает не полностью, другой layout выбирается сменой `alcid=` в boot-args (для ALC257 есть 11, 18, 86,
96, 97, 99, 100, 101).

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
- **Родное разрешение 3K**: `UEFI > Output > Resolution = Max`. Прежний конфиг с 1920x1200 — `EFI/OC/config-1920x1200.plist`
  (старый: в нём Wi-Fi через `itlmt7925`, без Bluetooth и звука).
- **AMFI ослаблен** (`AMFIPass`, `amfi=0x80`, `amfi_get_out_of_my_way=0x1`), **SIP выключен** (`csr-active-config = 7F080000`),
  `SecureBootModel = Disabled` — это нужно для неподписанных кекстов и Metal-плагина StrixArc.
- VRAM на этом APU — кусок обычной ОЗУ (carve-out по PA `0x780000000–0x87FFFFFFF`, в E820 reserved), BAR0 — 256 МиБ.
- Логирование включено (`Misc > Debug > Target = 67`), логи пишутся в корень флешки.

### boot-args

```
keepsyms=1 debug=0x100 mtkfe=native mtknativeif=1 itlmt_m1=0 itlmt_m2=1 itlmt_m3=1 itlmt_fast=6
phxdebug=1 -phxexperimental amfi=0x80 amfi_get_out_of_my_way=0x1 -amfipassbeta ipc_control_port_options=0 alcid=11
```

`mtkfe=native mtknativeif=1` включают родное меню Wi-Fi (без них MtkWlan работает «как Ethernet»), `alcid=11` — layout
звука. `itlmt_*` относятся к выключенному `itlmt7925`.

## Состав EFI

**ACPI:** `SSDT-EC-USBX-LAPTOP`, `SSDT-PLUG-ALT`
**Drivers:** `OpenRuntime`, `OpenHfsPlus`, `ResetNvramEntry`

| Кекст | Версия | Назначение |
|---|---|---|
| Lilu | 1.7.2 | основа для плагинов |
| AppleALC | 1.9.8 | звук ALC257 |
| BlueToolFixup | 2.7.2 | `bluetoothd` принимает сторонний контроллер |
| MtkBt | 0.2.4 | загрузка прошивки Bluetooth MT7925 |
| AMFIPass | 1.4.1 | работа с ослабленным AMFI |
| VirtualSMC | 1.3.8 | эмуляция SMC |
| SMCBatteryManager | 1.3.8 | батарея |
| ECEnabler | 1.0.6 | чтение полей EC (батарея) |
| NVMeFix | 1.1.3 | NVMe |
| RestrictEvents | 1.1.6 | косметика / ограничения процессов |
| AppleMCEReporterDisabler | 1.2 | убирает панику MCE на AMD |
| RealtekRTL8111 | 3.0.0 | Ethernet Realtek (USB/док) |
| VoodooPS2Controller (+ VoodooInput, Keyboard, Mouse, Trackpad) | 2.3.7 | клавиатура |
| VoodooI2C (+ VoodooI2CServices, VoodooGPIO) | 2.9.1 | шина I2C, AMD GPIO для прерывания тачпада |
| VoodooI2CHID | 1.0 | тачпад GXTP5100 |
| ItlXhciKick | 0.1.0 | опрос событий AMD xHCI каждую 1 мс |
| MtkWlan | 0.12.1 | драйвер Wi-Fi MT7925 — **включён** |
| MtkWlanNative | 0.4.13 (порт под 24H32) | родное меню Wi-Fi — **включён** |
| itlmt7925 / itlmt7925Native | 0.1.0 | прежний драйвер Wi-Fi — выключены (запасной вариант) |
| PhxBringup (StrixArc) | 1.0.0 | запуск GPU Radeon 880M — экспериментально |

## Установка

1. Скачайте архив из [Releases](../../releases) и распакуйте папку `EFI` в EFI-раздел флешки или диска.
2. **Сгенерируйте свои SMBIOS** (`MacBookPro16,1`) через [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) и впишите
   `MLB`, `SystemSerialNumber`, `SystemUUID`, `ROM` в `PlatformInfo > Generic` — в архиве стоят заглушки.
3. **Wi-Fi:** ничего вписывать не нужно — подключайтесь к сети из меню Wi-Fi. Если раньше карта работала «как Ethernet»,
   переименуйте старую службу: `networksetup -renamenetworkservice "Ethernet 2" "Wi-Fi"`.
4. В BIOS: выключите Secure Boot. Загружайтесь **с холодного старта** (тачпад, Bluetooth).
5. Если новые boot-args не применились (OpenCore не перезаписывает уже существующие), один раз сделайте Reset NVRAM в
   меню OpenCore.

## Планы

- [x] Wi-Fi: родное меню Wi-Fi вместо режима «как Ethernet», стабильность
- [x] Bluetooth (MT7925)
- [x] Звук (динамики)
- [ ] Звук: наушники в разъём и микрофон гарнитуры (проверить, при необходимости подобрать layout)
- [ ] Wi-Fi: скорость, счётчики трафика, сон; исследовать monitor mode MT7925 (задел под AWDL)
- [ ] Тачпад до идеала: работа после перезагрузки (не только с холодного старта), жесты, отклик
- [ ] Сон / пробуждение
- [ ] Подсветка экрана и её регулировка
- [x] StrixArc: 120 Гц (перетренировка eDP-линка на HBR3)
- [ ] StrixArc: видео, внешние мониторы
- [ ] Карта USB-портов, веб-камера, кард-ридер, прочие мелочи

## Версии

| Версия | Что внутри |
|---|---|
| [v0.2.0](../../releases/tag/v0.2.0) | Wi-Fi с родным меню (MtkWlan + порт MtkWlanNative под 24H32), Bluetooth (MtkBt + BlueToolFixup), звук (AppleALC, `alcid=11`) |
| [v0.1.0](../../releases/tag/v0.1.0) | первая выложенная версия: тачпад (холодный старт), USB-C/A, Wi-Fi (legacy), StrixArc PhxBringup |

## Благодарности

[Acidanthera](https://github.com/acidanthera) (OpenCore, Lilu, VirtualSMC, AppleALC, BrcmPatchRAM/BlueToolFixup и др.),
[AMD OS X](https://github.com/AMD-OSX) (AMD Vanilla патчи), [VoodooI2C](https://github.com/VoodooI2C),
[nullarc](https://github.com/luuumi-dev/nullarc), [watchthelight-labs](https://github.com/watchthelight-labs)
(MtkWlan, MtkWlanNative, MtkBt), [ga403wm-hackintosh](https://github.com/thanhnndev/ga403wm-hackintosh)
(itlmt7925, разбор IO80211 на 24H32).
