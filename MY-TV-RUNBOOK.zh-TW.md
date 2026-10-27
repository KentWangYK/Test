# 我的電視：本機執行清單（方案 B）

這個 repo 是 [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)（MIT 授權）的完整複本，只改了一個地方：

| 檔案 | 改動 |
| --- | --- |
| `launcher/app/src/main/res/values/config.xml` | `dock_apps` 改為 YouTube → Netflix → Disney+ → Hami Video；`columns` 維持 4（Dock 4 格） |

其餘的 `tools/`、`docs/`、`examples/` 和安全檢查都保持上游原樣。

雲端工作階段**沒辦法**做下面兩件事，所以都要在本機完成：

- 連上電視：雲端不在你家的網路上。
- 建置 APK：雲端的網路政策擋掉了 Android SDK 的下載。

## 0. 準備（在和電視同一個 Wi-Fi 的電腦上）

```bash
git clone https://github.com/kentwangyk/test.git cleanyourtv
cd cleanyourtv
git checkout claude/android-tv-debloat-launcher-j3y7ws
python3 -m venv tools/venv && tools/venv/bin/pip install -r tools/requirements.txt
```

建置啟動器需要 JDK 17 和 Android SDK（compileSdk 34）。裝好 Android Studio 就有，或自行設定 `ANDROID_HOME`。

電視端：設定 → 裝置偏好設定 → 關於 → 在「版本號」按 OK 7 次 → 返回 → 開發人員選項 → 開啟 USB 偵錯（有「網路／ADB 偵錯」的話也一起打開）。

> 有些機種開了 USB 偵錯後，會切斷 USB 埠的供電。如果電視是透過 **USB 有線網路轉接器**上網，請一次做完全部流程，不要分兩次。

## 1. 交給本機的 Claude Code

在上面那個資料夾裡打開 Claude Code，把 [`PROMPT.md`](PROMPT.md) 分隔線之間的提示詞貼進去，並做三件事：

- 把 `<MODEL NAME HERE>` 換成**電視型號**（這次還沒提供）。
- 說明「我已 clone 這個 repo，請沿用 `tools/` 和 `launcher/`；`config.xml` 的 Dock 已經設定好，不用再問 Dock」。
- 提醒它先檢查 `examples/` 裡有沒有同型號的資料夾。

它會依照提示詞依序執行：

1. 連線（ADB 5555 → 無線偵錯配對 → `tools/relay.py`），並告訴你何時在電視上按「允許 USB 偵錯」。
2. 把完整套件清單、已停用套件、`getprop` 和所有設定備份到 `backup/`，並寫好還原腳本。
3. 把套件分類，然後**用勾選清單問你**要保留哪些串流 app、語音助理、Chromecast、韌體更新。
4. 用 `tools/disable.sh` 分批停用，每批之間截圖確認。最後一批完成後等 15 秒再重開機，然後比對 `pm list packages -d`。
5. 檢查 SDK（需 ≥ 26）和能否側載 APK，再用 `launcher/build-and-install.sh` 建置並安裝啟動器、設為預設主畫面。
6. 驗證：用遙控器的按鍵注入搭配截圖，以及 `dumpsys activity activities | grep mFocusedApp`。
7. 收尾：關閉 USB 偵錯和網路偵錯、撤銷授權、重開機，然後用 `tools/adb-closed.py <電視 IP>` 確認埠已拒絕連線。

## 2. 確認 Hami Video 的套件名稱

Dock 第 4 格填的是 `hami.androidtv`。這是 Hami Video **電視版**的套件名稱，但只從第三方網站查到，還沒在實機上確認過。連上電視後請先執行：

```bash
TV_HOST=<電視 IP> tools/venv/bin/python tools/tv.py 'pm list packages | grep -i hami'
```

如果名稱不一樣（例如 `com.cht.ottPlayer`），把 `config.xml` 裡的 `hami.androidtv` 改成實際名稱再建置。即使沒改也不會壞：裝置上沒有的 app 會被略過，空出來的 Dock 格子會自動補上 app 格的第一個 app（`fill_dock`）。

## 3. 之後要改 Dock 或桌布

- **Dock 順序**：編輯 `config.xml` 的 `dock_apps`，然後重新執行 `TV_HOST=<IP> launcher/build-and-install.sh`。
- **不想重新建置**：用主題來改，`tools/theme.sh themes/<name>`（`config.json` 的鍵名和 `config.xml` 相同，並可附桌布，詳見 `themes/README.md`）。
- **重新啟用某個 app**：`tools/restore.sh <當初用的停用清單>`，或執行 `tools/tv.py 'pm enable <套件名稱>'`。
