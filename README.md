# dgx-spark-fan-thermald

讓 DGX Spark 的風扇 **溫度 ≥ 80 °C 自動全速、降到 ≤ 70 °C 自動交回原廠曲線**。

只有兩個檔案：

| 檔案 | 作用 |
|---|---|
| `nvfancontrol-thermald` | 每 2 秒讀一次溫度，決定要 `max` 還是 `auto` |
| `nvfancontrol-thermald.service` | systemd 服務，開機自動啟動 |

實際控制風扇的是 [dgx-spark-fan-override](https://github.com/Z841973620/dgx-spark-fan-override) 提供的 `nvfancontrol` 指令，所以要先裝它（第 1 步）。

---

## 安裝（在 DGX Spark 上，照順序整段貼上）

所有指令都在 **DGX Spark 上**執行。

### 0. 安裝編譯工具並下載這個 repo

```bash
sudo apt update
sudo apt install -y git dkms build-essential linux-headers-$(uname -r)
cd ~
git clone https://github.com/WeilJimmer/dgx-spark-fan-thermald.git
```

### 1. 安裝 `nvfancontrol`（風扇控制驅動）

```bash
cd ~
git clone https://github.com/Z841973620/dgx-spark-fan-override.git
cd dgx-spark-fan-override
chmod 0755 nvfancontrol/DEBIAN/postinst nvfancontrol/DEBIAN/prerm nvfancontrol/DEBIAN/postrm nvfancontrol/usr/sbin/nvfancontrol
dpkg-deb --build --root-owner-group nvfancontrol nvfancontrol.deb
sudo apt install -y ./nvfancontrol.deb
```

> `chmod` 那行不能省：上游 repo 裡這幾個檔案沒有執行權限，不加的話 `dpkg-deb` 會直接報錯。

確認驅動可用（兩個指令都要印出 `OK`）：

```bash
sudo nvfancontrol max
```

```bash
sudo nvfancontrol auto
```

第一行執行後風扇會明顯變大聲，第二行執行後會慢慢降下來。

### 2. 安裝自動溫控服務

```bash
cd ~/dgx-spark-fan-thermald
sudo install -m 0755 nvfancontrol-thermald /usr/local/sbin/nvfancontrol-thermald
sudo install -m 0644 nvfancontrol-thermald.service /etc/systemd/system/nvfancontrol-thermald.service
sudo systemctl daemon-reload
sudo systemctl enable --now nvfancontrol-thermald.service
```

### 3. 確認有在跑

```bash
systemctl status nvfancontrol-thermald --no-pager
```

看到 `active (running)` 就完成了，之後每次開機都會自動啟動。

---

## 運作規則

- 最熱的 thermal zone **≥ 80 °C** → 風扇全速（`nvfancontrol max`）
- 降到 **≤ 70 °C** → 交回 EC 原廠溫控曲線（`nvfancontrol auto`）
- 70–80 °C 之間維持目前狀態，避免風扇來回切換
- 讀不到任何合理的溫度 → 一律全速（寧可吵，不要過熱）
- 停止服務時自動恢復 `auto`

## 看即時紀錄

```bash
journalctl -u nvfancontrol-thermald -f
```

每次切換都會印一行，例如 `Switching to max: thermal_zone0(cpu-thermal) at 81.0 C`。按 `Ctrl+C` 離開。

## 改門檻溫度（選用）

```bash
sudo systemctl edit nvfancontrol-thermald.service
```

在編輯器裡貼上要改的值，存檔離開：

```ini
[Service]
Environment=MAX_AT_CELSIUS=85
Environment=AUTO_AT_CELSIUS=75
```

```bash
sudo systemctl restart nvfancontrol-thermald.service
```

`AUTO_AT_CELSIUS` 必須小於 `MAX_AT_CELSIUS`，否則服務會拒絕啟動。

## 只看 CPU 溫度（選用）

預設會取**所有** thermal zone 的最高溫。服務啟動時會把每個 zone 的名稱和溫度印在 log 裡：

```bash
journalctl -u nvfancontrol-thermald -b --no-pager | head -30
```

找到 CPU 的 zone 名稱後（例如 `cpu-thermal`），用上一節的方法加一行：

```ini
[Service]
Environment=ZONE_TYPE_PATTERN=cpu
```

## 解除安裝

```bash
sudo systemctl disable --now nvfancontrol-thermald.service
sudo rm /usr/local/sbin/nvfancontrol-thermald /etc/systemd/system/nvfancontrol-thermald.service
sudo systemctl daemon-reload
```

風扇會回到原廠 `auto`。如果連驅動也要移除：

```bash
sudo apt remove -y nvfancontrol
```

## 注意事項

- 這個服務和 `nvfancontrol` 套件自帶的 `nvfancontrol.service`（開機就固定全速）互相衝突，啟用其中一個時另一個會被停掉。
- 如果 `nvfancontrol` 某次請求失敗，服務會等 30 秒後再試，不會連續重送。
- 換 kernel 之後，DKMS 會自動重新編譯驅動，前提是新 kernel 的 headers 已經安裝。
