# MARK ROBOT / TWRB 交接文件

這個 repository 是 MARK ROBOT / TWRB 專題的交接文件網站，使用 [Quartz v4](https://quartz.jzhao.xyz/) 將 Markdown 筆記發布成靜態網站。

文件內容集中在 `content/`，包含 PC 系統安裝、TWRB 樹莓派 OS 燒錄、Arduino MEGA 程式燒錄、ROS1 / ROS2 / ros1_bridge 環境配置、執行流程，以及 `controller.py` 控制邏輯說明。

## 文件結構

```text
content/
  index.md              # 交接入口與閱讀順序
  01_系統安裝/          # PC、TWRB、Arduino 安裝與燒錄
  02_環境配置/          # ROS1、ROS2、ros1_bridge、bashrc
  03_執行流程/          # 實際啟動與連線流程
  04_程式碼說明/        # 控制程式邏輯說明
  99_附錄/              # 非必要補充文件
  assets/               # 文件圖片素材
```

## 快速開始

安裝依賴：

```bash
npm install
```

本機預覽文件網站：

```bash
npx quartz build --serve
```

檢查格式與 TypeScript：

```bash
npm run check
```

## 維護方式

- 新增交接內容時，優先放到對應的 `content/` 子資料夾。
- 每篇文件建議維持「目的、前置條件、操作步驟、驗證方式、注意事項」的結構。
- 圖片素材放在 `content/assets/`，並使用相對路徑引用。
- 修改後建議執行 `npm run check` 與 `npx quartz build` 確認網站可正常產出。

## 外部資源

- [範例程式碼 Google Drive](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link)
- [範例影片](https://youtu.be/L9N0wVhpF3s)
