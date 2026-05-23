# TWRB 樹莓派 OS 燒錄

## 目的

將現成的 TWRB 系統映像檔燒錄到樹莓派使用的 SD 卡，讓車體端可以直接使用既有環境。

## 前置條件

- 已準備 TWRB 使用的 SD 卡。
- 已安裝 DiskGenius。
- 已取得映像檔 `TWRB_ID3_mini.img`。

## 操作步驟

### 1. 下載映像檔

映像檔放在 [Google Drive](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link)，檔名為 `TWRB_ID3_mini.img`。

### 2. 從映像檔還原 SD 卡

在 DiskGenius 中，對 SD 卡按右鍵，選擇從映像檔還原。

![](../assets/twrb%20樹梅派%20OS%20燒錄/file-20260114162559520.png)

### 3. 選擇映像檔

切換成所有檔案，選擇 `TWRB_ID3_mini.img`。

![](../assets/twrb%20樹梅派%20OS%20燒錄/file-20260114162705267.png)

### 4. 調整磁區大小

依照 SD 卡容量調整磁區大小。

![](../assets/twrb%20樹梅派%20OS%20燒錄/file-20260114162852728.png)

![](../assets/twrb%20樹梅派%20OS%20燒錄/file-20260114162906464.png)

### 5. 開始燒錄

確認目標磁碟是 SD 卡後，再按開始。

![](../assets/twrb%20樹梅派%20OS%20燒錄/file-20260114162937868.png)

## 驗證方式

燒錄完成後，將 SD 卡插回樹莓派。樹莓派需要與 PC 在同一網段，後續才能透過 SSH 連線。

## 注意事項

第一次連線樹莓派時有兩種方式：

- 使用外接螢幕與 mini HDMI 線直接操作。
- 將樹莓派接上路由器，用 [Angry IP Scanner](https://angryip.org/) 找到 IP，再透過 SSH 進去設定 NetworkManager、Wi-Fi 與固定 IP。
