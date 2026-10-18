# MARK ROBOT / TWRB 交接文件

這個 repository 是 MARK ROBOT / TWRB 專題的交接文件網站，使用 [Quartz v4](https://quartz.jzhao.xyz/) 將 Markdown 筆記發布成靜態網站。

文件內容集中在 `content/`，包含 PC 系統安裝、TWRB 樹莓派 OS 燒錄、Arduino MEGA 程式燒錄、ROS1 / ROS2 / ros1_bridge 環境配置、執行流程、現行 PC 控制架構、Arduino 輪速閉迴路與 ROS 通訊協定。舊 `controller.py` 頁保留歷史說明及現行模式 2 的 PWM 對照。

2026/10/18 技術更新依據 `twrb_project` 主分支 commit [`98dba0d6`](https://github.com/wwwlee197/twrb_project/tree/98dba0d6e0f5945e81efe75ade10d51e94df9292)。模式 2 使用 PWM，其餘正式模式使用 RPS × 1000；正式 Arduino 燒錄版本與實車驗收仍待確認。差異、來源與待確認事項見[版本查核紀錄](content/99_附錄/程式版本更新查核紀錄.md)。

## 文件結構

```text
content/
  index.md              # 交接入口與閱讀順序
  01_系統安裝/          # PC、TWRB、Arduino 安裝與燒錄
  02_環境配置/          # ROS1、ROS2、ros1_bridge、bashrc
  03_執行流程/          # 實際啟動與連線流程
  04_程式碼說明/        # PC 架構／模式比較、Arduino 閉迴路、ROS 協定、舊控制器
  99_附錄/              # 版本查核與待確認事項、VNC 補充
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
