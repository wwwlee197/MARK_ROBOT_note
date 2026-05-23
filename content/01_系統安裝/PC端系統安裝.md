# PC 端系統安裝

## 目的

PC 端主要負責開發、執行 ROS1 / ROS2、啟動 ros1_bridge，以及透過 SSH 操作 TWRB。為了穩定執行專題環境，本文件採用外接 SSD 安裝 Ubuntu，讓 Windows 與 Ubuntu 並存。

## 前置條件

- 先備份筆電重要資料。
- 準備 16GB 以上 USB，用於製作 Ubuntu 安裝碟。
- 準備外接 SSD，建議 256GB 以上。
- 準備 Balena Etcher 與 DiskGenius。

## 安裝方式

可行方式有兩種：

1. 直接分割筆電內建硬碟。此方法會動到已有資料的磁碟，風險較高。
2. 使用外接 SSD 作為 Ubuntu 系統硬碟。本文件採用此方法。

## 操作步驟

### 1. 製作 Ubuntu USB 安裝碟

16GB USB 用於燒錄 Ubuntu `.iso` 安裝映像檔。

- Ubuntu 22.04.5 LTS ISO：[下載 64-bit PC desktop image](https://releases.ubuntu.com/jammy/ubuntu-22.04.5-desktop-amd64.iso)
- Balena Etcher：[下載 balenaEtcher](https://github.com/balena-io/etcher/releases/download/v2.1.4/balenaEtcher-2.1.4.Setup.exe)

### 2. 分割外接 SSD

推薦使用 DiskGenius。分割區表類型建議轉換為 GUID 模式，再依序建立分割區。

![](../assets/PC端系統安裝/file-20260114121040697.png)

建議分割如下：

| 分割區 | 檔案系統 / 類型      | 建議容量 | 用途                                   |
| ------ | -------------------- | -------- | -------------------------------------- |
| ESP    | FAT32                | 1GB      | `/boot` 開機分割區，放置 `.efi` 開機檔 |
| swap   | Linux swap partition | 16GB     | Linux swap 交換空間                    |
| root   | EXT4                 | 64GB     | Linux `/` 目錄                         |
| home   | EXT4                 | 100GB    | Linux `/home` 目錄                     |

剩餘空間可保留成一般儲存空間使用。

參考連結：[CSDN 教學](https://blog.csdn.net/hypc9709/article/details/127941834)

### 3. 關閉 BitLocker 與 Secure Boot

使用外部硬碟或雙系統時，需要關閉安全啟動。Windows 為保護系統資料，可能會觸發 BitLocker。

建議提前關閉 BitLocker：

![](../assets/PC端系統安裝/file-20260114123654633.png)

如果忘記關閉，或因其他原因觸發 BitLocker：

![](../assets/PC端系統安裝/file-20260114123750915.png)

可到 [Microsoft 帳戶設定頁面](https://account.microsoft.com/?ref=MeControl&refd=www.microsoft.com) 查詢復原金鑰。

![](../assets/PC端系統安裝/file-20260114124305701.png)

![](../assets/PC端系統安裝/file-20260114124254237.png)

### 4. 設定 BIOS

1. 插上 Ubuntu 安裝 USB。
2. 若要安裝到外接 SSD，也先接上外接 SSD。
3. 進入 BIOS。
4. 關閉 Secure Boot。
5. 將 USB 安裝碟排到第一順位。
6. 重新開機進入 Ubuntu 安裝流程。

### 5. 首次安裝 Ubuntu

選擇鍵盤類型。注音或新酷音可以後續再新增。

![](../assets/PC端系統安裝/file-20260114124813073.png)

安裝選項中，`Download updates while installing Ubuntu` 可以取消勾選。

![](../assets/PC端系統安裝/file-20260114124836579.png)

安裝方式請選擇 `Something else`，手動指定分割區。

其他可能選項包含：

- 偵測到 Windows 後自動分割磁碟建立雙系統。
- 完全覆蓋磁碟，讓本機只剩 Ubuntu。這個選項不要選，會清掉原系統。

### 6. 指定安裝磁碟與分割區

這一步最重要：請依照容量與磁碟名稱確認哪一顆是外接 SSD，避免操作到 Windows 內建硬碟。

![](../assets/PC端系統安裝/file-20260114130005662.jpg)

範例中偵測到兩顆硬碟：

- `nvme0n1p6`：筆電內建硬碟。
- `sda`：外接 SSD。

筆電內建硬碟分割範例如下：

![](../assets/PC端系統安裝/file-20260114131003667.png)

將外接 SSD 的 Type 與 Mount 設定成安裝範例中的配置。若有保留一般隨身碟用途的磁區，該磁區不用調整 Type 與 Mount。

`Device for boot loader installation` 可選擇外接 SSD 整顆磁碟，例如 `sda`，或選擇 EFI 分割區，例如 `sda1_efi`。重點是確認選到外接 SSD，而不是 Windows 內建硬碟。

## 驗證方式

安裝完成後，重新開機並從外接 SSD 進入 Ubuntu。進入系統後建議先完成：

- VS Code 安裝。
- 新酷音輸入法或慣用輸入法設定。
- 常用開發工具與個人環境設定。

## OS 概念

開機流程可理解為：

```text
電源 -> EFI / UEFI -> 開機載入程式 -> 作業系統
```

本機 OS 安裝時，通常會將開機路徑，例如 `\EFI\Ubuntu\shimx64.efi`，註冊到主機板的 NVRAM Boot Entries。主機板因此知道要去哪裡找開機檔。

外接 OS 插到新電腦時，新電腦的 NVRAM 沒有該 OS 的註冊紀錄，因此無法透過標準開機選單啟動。這時需要依賴 UEFI 的 Fallback 機制。

### Fallback Path

UEFI 規範定義：當偵測到可移除裝置時，若 NVRAM 沒有紀錄，Firmware 會掃描該裝置 ESP 分割區中的預設路徑：

```text
\EFI\BOOT\BOOTX64.EFI
```

為了讓外接硬碟在不同電腦上都能啟動，必須確認 Bootloader，例如 GRUB，存在於上述位置。

### 無法開機時的補救方向

如果 Fallback Path 遺失導致無法開機，可使用 Ubuntu [Live USB](#1-製作-ubuntu-usb-安裝碟) 開機，選擇 `Try Ubuntu` 進入 Live 環境，再 mount 外接硬碟的 ESP 分割區，手動修復或複製 `.efi` 檔案。
