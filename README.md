# dgx-spark-fan-thermald

[English](#english) | [繁體中文](#繁體中文)

---

## English

Temperature-driven fan control for NVIDIA DGX Spark: **full speed at ≥ 80 °C, back to the EC's stock curve at ≤ 70 °C**.

Just two files:

| File | Purpose |
|---|---|
| `nvfancontrol-thermald` | Reads temperatures every 2 s and decides between `max` and `auto` |
| `nvfancontrol-thermald.service` | systemd service, starts on boot |

### Credits

The actual fan control is done by **[dgx-spark-fan-override](https://github.com/Z841973620/dgx-spark-fan-override)** by [Z841973620](https://github.com/Z841973620), which reverse-engineered the DGX Spark EC fan override and provides the `nvfancontrol` kernel module and command. This repo only adds a small daemon on top of it. All credit for the hard part goes to the original author.

Backup fork (in case the original becomes unavailable): <https://github.com/WeilJimmer/dgx-spark-fan-override>

### Install (on the DGX Spark, paste each block in order)

#### 1. Install `nvfancontrol` (the original author's .deb)

```bash
sudo apt update
sudo apt install -y git dkms build-essential linux-headers-$(uname -r)
cd /tmp
wget https://github.com/Z841973620/dgx-spark-fan-override/releases/download/1.0.0/nvfancontrol_1.0.0_arm64.deb
echo "c1174bc888810287a4ecc27db3a3a6fb8553a8282d0649415c5d43779f9e202c  nvfancontrol_1.0.0_arm64.deb" | sha256sum -c
sudo apt install -y ./nvfancontrol_1.0.0_arm64.deb
```

The `sha256sum -c` line must print `OK`. If it doesn't, stop and do not install.

<details>
<summary>Original release gone? Build the same package from the backup fork instead</summary>

The fork has no release, so build the .deb from source (same commit as the original 1.0.0 release):

```bash
sudo apt update
sudo apt install -y git dkms build-essential linux-headers-$(uname -r)
cd ~
git clone https://github.com/WeilJimmer/dgx-spark-fan-override.git
cd dgx-spark-fan-override
chmod 0755 nvfancontrol/DEBIAN/postinst nvfancontrol/DEBIAN/prerm nvfancontrol/DEBIAN/postrm nvfancontrol/usr/sbin/nvfancontrol
dpkg-deb --build --root-owner-group nvfancontrol nvfancontrol.deb
sudo apt install -y ./nvfancontrol.deb
```

The `chmod` line is required: these files are not executable in git, and `dpkg-deb` refuses to build without it.

</details>

Check the driver works (both commands must print `OK`):

```bash
sudo nvfancontrol max
```

```bash
sudo nvfancontrol auto
```

After the first one the fans get noticeably louder; after the second they slowly calm down.

#### 2. Install the temperature service

```bash
cd ~
git clone https://github.com/WeilJimmer/dgx-spark-fan-thermald.git
cd dgx-spark-fan-thermald
sudo install -m 0755 nvfancontrol-thermald /usr/local/sbin/nvfancontrol-thermald
sudo install -m 0644 nvfancontrol-thermald.service /etc/systemd/system/nvfancontrol-thermald.service
sudo systemctl daemon-reload
sudo systemctl enable --now nvfancontrol-thermald.service
```

#### 3. Check it is running

```bash
systemctl status nvfancontrol-thermald --no-pager
```

`active (running)` means you're done. It starts automatically on every boot.

### How it behaves

- Hottest thermal zone **≥ 80 °C** → fans at full speed (`nvfancontrol max`)
- Drops to **≤ 70 °C** → back to the EC stock curve (`nvfancontrol auto`)
- Between 70 and 80 °C the current state is kept, so the fans don't flip back and forth
- No plausible temperature reading at all → full speed (loud beats overheated)
- Stopping the service restores `auto`

### Live log

```bash
journalctl -u nvfancontrol-thermald -f
```

Every switch prints a line such as `Switching to max: thermal_zone0(cpu-thermal) at 81.0 C`. Press `Ctrl+C` to exit.

### Change the thresholds (optional)

```bash
sudo systemctl edit nvfancontrol-thermald.service
```

Paste the values you want in the editor, then save and exit:

```ini
[Service]
Environment=MAX_AT_CELSIUS=85
Environment=AUTO_AT_CELSIUS=75
```

```bash
sudo systemctl restart nvfancontrol-thermald.service
```

`AUTO_AT_CELSIUS` must be lower than `MAX_AT_CELSIUS`, otherwise the service refuses to start.

### Only watch the CPU (optional)

By default the hottest of **all** thermal zones is used. On start the service logs every zone's name and temperature:

```bash
journalctl -u nvfancontrol-thermald -b --no-pager | head -30
```

Once you know the CPU zone name (e.g. `cpu-thermal`), add a line the same way as above:

```ini
[Service]
Environment=ZONE_TYPE_PATTERN=cpu
```

### Uninstall

```bash
sudo systemctl disable --now nvfancontrol-thermald.service
sudo rm /usr/local/sbin/nvfancontrol-thermald /etc/systemd/system/nvfancontrol-thermald.service
sudo systemctl daemon-reload
```

The fans return to stock `auto`. To remove the driver as well:

```bash
sudo apt remove -y nvfancontrol
```

### Notes

- This service conflicts with the `nvfancontrol.service` shipped in the original package (which pins the fans at full speed on boot). Enabling one stops the other.
- If an `nvfancontrol` request fails, the service waits 30 s before trying again instead of resending immediately.
- After a kernel upgrade, DKMS rebuilds the driver automatically as long as the new kernel's headers are installed.

---

## 繁體中文

讓 DGX Spark 的風扇 **溫度 ≥ 80 °C 自動全速、降到 ≤ 70 °C 自動交回原廠曲線**。

只有兩個檔案：

| 檔案 | 作用 |
|---|---|
| `nvfancontrol-thermald` | 每 2 秒讀一次溫度，決定要 `max` 還是 `auto` |
| `nvfancontrol-thermald.service` | systemd 服務，開機自動啟動 |

### 致謝

實際控制風扇的是 [Z841973620](https://github.com/Z841973620) 的 **[dgx-spark-fan-override](https://github.com/Z841973620/dgx-spark-fan-override)**。原作逆向分析了 DGX Spark EC 的風扇覆蓋機制，並提供 `nvfancontrol` 核心模組與指令。這個 repo 只是在上面加了一個小常駐程式，最困難的部分都歸功於原作者。

備援 fork（以防原作失效）：<https://github.com/WeilJimmer/dgx-spark-fan-override>

### 安裝（在 DGX Spark 上，照順序整段貼上）

#### 1. 安裝 `nvfancontrol`（原作的 .deb）

```bash
sudo apt update
sudo apt install -y git dkms build-essential linux-headers-$(uname -r)
cd /tmp
wget https://github.com/Z841973620/dgx-spark-fan-override/releases/download/1.0.0/nvfancontrol_1.0.0_arm64.deb
echo "c1174bc888810287a4ecc27db3a3a6fb8553a8282d0649415c5d43779f9e202c  nvfancontrol_1.0.0_arm64.deb" | sha256sum -c
sudo apt install -y ./nvfancontrol_1.0.0_arm64.deb
```

`sha256sum -c` 那行必須印出 `OK`。沒有印出 `OK` 就停下來，不要安裝。

<details>
<summary>原作的 Release 不見了？改從備援 fork 自己打包同一個套件</summary>

fork 沒有 Release，所以要從原始碼打包 .deb（和原作 1.0.0 Release 是同一個 commit）：

```bash
sudo apt update
sudo apt install -y git dkms build-essential linux-headers-$(uname -r)
cd ~
git clone https://github.com/WeilJimmer/dgx-spark-fan-override.git
cd dgx-spark-fan-override
chmod 0755 nvfancontrol/DEBIAN/postinst nvfancontrol/DEBIAN/prerm nvfancontrol/DEBIAN/postrm nvfancontrol/usr/sbin/nvfancontrol
dpkg-deb --build --root-owner-group nvfancontrol nvfancontrol.deb
sudo apt install -y ./nvfancontrol.deb
```

`chmod` 那行不能省：這幾個檔案在 git 裡沒有執行權限，不加的話 `dpkg-deb` 會拒絕打包。

</details>

確認驅動可用（兩個指令都要印出 `OK`）：

```bash
sudo nvfancontrol max
```

```bash
sudo nvfancontrol auto
```

第一個指令執行後風扇會明顯變大聲，第二個執行後會慢慢降下來。

#### 2. 安裝自動溫控服務

```bash
cd ~
git clone https://github.com/WeilJimmer/dgx-spark-fan-thermald.git
cd dgx-spark-fan-thermald
sudo install -m 0755 nvfancontrol-thermald /usr/local/sbin/nvfancontrol-thermald
sudo install -m 0644 nvfancontrol-thermald.service /etc/systemd/system/nvfancontrol-thermald.service
sudo systemctl daemon-reload
sudo systemctl enable --now nvfancontrol-thermald.service
```

#### 3. 確認有在跑

```bash
systemctl status nvfancontrol-thermald --no-pager
```

看到 `active (running)` 就完成了，之後每次開機都會自動啟動。

### 運作規則

- 最熱的 thermal zone **≥ 80 °C** → 風扇全速（`nvfancontrol max`）
- 降到 **≤ 70 °C** → 交回 EC 原廠溫控曲線（`nvfancontrol auto`）
- 70–80 °C 之間維持目前狀態，避免風扇來回切換
- 讀不到任何合理的溫度 → 一律全速（寧可吵，不要過熱）
- 停止服務時自動恢復 `auto`

### 看即時紀錄

```bash
journalctl -u nvfancontrol-thermald -f
```

每次切換都會印一行，例如 `Switching to max: thermal_zone0(cpu-thermal) at 81.0 C`。按 `Ctrl+C` 離開。

### 改門檻溫度（選用）

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

### 只看 CPU 溫度（選用）

預設會取**所有** thermal zone 的最高溫。服務啟動時會把每個 zone 的名稱和溫度印在 log 裡：

```bash
journalctl -u nvfancontrol-thermald -b --no-pager | head -30
```

找到 CPU 的 zone 名稱後（例如 `cpu-thermal`），用上一節的方法加一行：

```ini
[Service]
Environment=ZONE_TYPE_PATTERN=cpu
```

### 解除安裝

```bash
sudo systemctl disable --now nvfancontrol-thermald.service
sudo rm /usr/local/sbin/nvfancontrol-thermald /etc/systemd/system/nvfancontrol-thermald.service
sudo systemctl daemon-reload
```

風扇會回到原廠 `auto`。如果連驅動也要移除：

```bash
sudo apt remove -y nvfancontrol
```

### 注意事項

- 這個服務和原作套件自帶的 `nvfancontrol.service`（開機就固定全速）互相衝突，啟用其中一個時另一個會被停掉。
- 如果 `nvfancontrol` 某次請求失敗，服務會等 30 秒後再試，不會連續重送。
- 換 kernel 之後，DKMS 會自動重新編譯驅動，前提是新 kernel 的 headers 已經安裝。
