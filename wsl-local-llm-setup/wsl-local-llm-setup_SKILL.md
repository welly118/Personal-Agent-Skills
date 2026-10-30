---
name: wsl-local-llm-setup
description: 在 Windows 的 WSL2（Debian）裡部署本地 LLM 的完整步驟與排錯紀錄，涵蓋 WSL 遷移、Docker、NVIDIA Container Toolkit、Ollama、Open WebUI、Hugging Face GGUF 模型下載與 Modelfile、Cloudflare Tunnel 對外連線。只要使用者提到在 WSL 安裝或更新 Ollama / Open WebUI、本地模型部署、GPU 在 WSL 或 Docker 裡抓不到、Open WebUI 連不到 Ollama、自訂 Modelfile、Cloudflare Tunnel 對外開放，或模型跑不起來，就使用這份文件，即使使用者沒有明說「部署」。
---

# 在 WSL2 部署本地模型

- 最後更新：2026-10-30
- 作者：welly118
- 編輯：Claude Sonnet5.5

## 適用環境

- Windows 版本：Windows 10 22H2
- WSL 版本與發行版：WSL2 + Debian 12
- 顯卡與 VRAM：RTX 5060 Ti 16GB
- 顯卡驅動版本：（請補上，Windows PowerShell 執行 `nvidia-smi` 可查；RTX 50 系列需要較新的驅動）
- 使用工具：Ollama + Open WebUI（Docker）
- 儲存空間：C 槽為 SATA SSD、E 槽為 M.2 SSD（WSL 與模型放 E 槽）

## 文件慣例

- 標示 **PowerShell** 的指令在 Windows 的 PowerShell 執行。
- 標示 **Debian** 的指令在 WSL 的 Debian 終端機執行。
- `<角括號>` 內是要換成自己內容的部分。
- 本文的指令與網址取自撰寫當下的官方文件，Docker、NVIDIA、Cloudflare 的套件庫位置偶爾會變動，失敗時請先對照官方文件。

## 前置檢查

1. 確認 WSL 是 v2（PowerShell）：`wsl -l -v`
2. 確認 Windows 端已安裝顯卡驅動（PowerShell）：執行 `nvidia-smi`，若沒有輸出，先到 Windows 安裝驅動。驅動只裝在 Windows，**不要在 WSL 裡再裝一次**。
3. 在 WSL 內執行 `nvidia-smi`，能看到顯卡才代表 GPU 通了。（完成步驟 1 之後再檢查）

## 安裝步驟

### 步驟 1：安裝 Debian 並遷移到 E 槽

**PowerShell**

```powershell
wsl --install -d Debian
```

首次啟動會要求建立使用者名稱與密碼，完成後回到 PowerShell 繼續：

```powershell
mkdir E:\wsl\debian
wsl --terminate Debian
wsl --export Debian E:\wsl\debian\debian.tar
wsl --unregister Debian
wsl --import Debian E:\wsl\debian E:\wsl\debian\debian.tar --version 2
```

`wsl --import` 的參數順序是「發行版名稱、安裝目錄、tar 檔路徑」。確認能正常進入後，`debian.tar` 可以刪除以節省空間。

如果 `wsl --version` 顯示版本夠新，也可以改用一行完成：`wsl --manage Debian --move E:\wsl\debian`。

### 步驟 2：設定預設使用者並啟用 systemd

匯入後預設會以 root 登入，所以先在 Debian 設定預設使用者，並同時啟用 systemd（後面所有 `systemctl` 都依賴它）。

**Debian**

```bash
nano /etc/wsl.conf
```

內容如下：

```ini
[boot]
systemd=true

[user]
default=<你的使用者名稱>
```

在 nano 中依序按 `Ctrl+O`、`Enter`、`Ctrl+X` 儲存並離開。

**PowerShell**（重啟 WSL，設定才會生效）

```powershell
wsl --shutdown
wsl -d Debian
```

驗證：進入後 `whoami` 應顯示你的使用者名稱，`systemctl is-system-running` 應顯示 `running` 或 `degraded`。

### 步驟 3：安裝 Docker

**Debian**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

新增套件庫：

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

安裝 Docker Engine：

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

把自己加入 docker 群組，之後不用每次打 sudo：

```bash
sudo usermod -aG docker $USER
```

接著在 PowerShell 執行 `wsl --shutdown` 後重新進入 Debian（群組變更需要重新登入才生效）。

驗證：`docker ps` 能正常列出（空清單也算成功）。

### 步驟 4：安裝 NVIDIA Container Toolkit

這一步讓 Docker 容器能使用 GPU。（注意：這裡裝的是 Container Toolkit，不是完整的 CUDA toolkit。）

**Debian**

新增套件庫：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

安裝並設定 Docker runtime：

```bash
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

驗證（RTX 50 系列請使用 CUDA 12.8 以上的映像；若該 tag 不存在，到 Docker Hub 的 nvidia/cuda 頁面找較新的 base tag）：

```bash
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu22.04 nvidia-smi
```

能印出顯卡資訊就代表容器能用 GPU。

### 步驟 5：安裝 Ollama

**Debian**

```bash
sudo apt install -y git curl zstd
curl -fsSL https://ollama.com/install.sh | sh
```

`zstd` 是 Ollama 安裝腳本解壓縮需要的套件（注意套件名稱是 `zstd`，不是 `zst`）。

接著**必須**讓 Ollama 監聽所有網卡，否則 Docker 裡的 Open WebUI 連不到它（Ollama 預設只聽 127.0.0.1）：

```bash
sudo systemctl edit ollama
```

在編輯器開啟的空白處加入下面兩行（要寫在註解區塊的上方範圍內，照編輯器提示的位置貼上），儲存離開：

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

套用設定：

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

驗證：`ollama ls` 能執行（目前沒有模型是正常的）。

### 步驟 6：安裝 Open WebUI

**Debian**

建立專用資料夾並編寫 compose 檔：

```bash
mkdir -p ~/open-webui && cd ~/open-webui
nano docker-compose.yaml
```

`docker-compose.yaml` 內容：

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: open-webui
    ports:
      - "3000:8080"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
      - WEBUI_SECRET_KEY=<自行產生一串隨機字串>
    volumes:
      - open-webui:/app/backend/data
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: always

volumes:
  open-webui:
```

`WEBUI_SECRET_KEY` 可用 `openssl rand -hex 32` 產生。啟動：

```bash
docker compose up -d
```

### 步驟 7：測試 Open WebUI

1. 在 Windows 瀏覽器開啟 `http://localhost:3000`。
2. 第一個註冊的帳號會自動成為管理員，請立刻註冊自己的管理員帳號。
3. 註冊完成後，到管理員設定關閉「新使用者註冊」，避免陌生人自行註冊（尤其是之後會開放 Cloudflare Tunnel 對外）。

### 步驟 8：安裝 Hugging Face CLI

**Debian**

```bash
curl -LsSf https://hf.co/cli/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

驗證：`hf --help` 有輸出。（新版指令名稱是 `hf`，舊的 `huggingface-cli` 已淘汰。）若要下載需要授權的模型，另外執行 `hf auth login`。

## 下載與執行模型

### 1. 選擇模型

到 Hugging Face 註冊帳號，並在個人設定中填入自己的顯卡型號，模型頁面就會依 VRAM 顯示相容性建議。16GB VRAM 通常選 Q4 ～ Q5 量化的 14B 以下模型；模型檔大小加上上下文所需空間，必須能放進 VRAM，否則會被切到 CPU，速度大幅下降。

### 2. 下載 GGUF

1. 在模型頁面右下角的區塊選擇 Quantizations，挑選合適的量化版本。
2. 進入該 GGUF 的頁面後，參考作者的品質說明與右上角的平台推薦。
3. 選好後點右上角 `Use this model`，再選 `Download with hf-cli`，複製指令。
4. 回到 Debian 貼上執行，**並加上 `--local-dir ~/models`**，讓檔案落在固定位置（預設會放在 `~/.cache/huggingface/hub/` 底下很深的目錄）：

```bash
mkdir -p ~/models
hf download <作者>/<倉庫名稱> <檔名>.gguf --local-dir ~/models
```

最後檔案路徑會是 `/home/<使用者名稱>/models/<檔名>.gguf`，請記下來。

### 3. 製作 Modelfile

```bash
mkdir -p ~/mdf && cd ~/mdf
nano <自訂模型簡稱>.mdf
```

內容如下，**所有行都要頂格寫、不能縮排**（`TEMPLATE` 的三引號內若有縮排，空白會被送進 prompt）：

```
FROM /home/<使用者名稱>/models/<檔名>.gguf

# 盡量把所有層放在 GPU 上
PARAMETER num_gpu 99
# 單次對話的上下文長度
PARAMETER num_ctx 8192
# 溫度與 top_p，請依模型作者的建議調整
PARAMETER temperature 0.6
PARAMETER top_p 0.95

# 對話截斷標籤（ChatML 規範）
PARAMETER stop "<|im_start|>"
PARAMETER stop "<|im_end|>"
PARAMETER stop "<|endoftext|>"

# Prompt 模板（ChatML 格式）
TEMPLATE """{{ if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}{{ if .Prompt }}<|im_start|>user
{{ .Prompt }}<|im_end|>
{{ end }}<|im_start|>assistant
{{ .Response }}<|im_end|>"""
```

> **注意**：上面的 `stop` 與 `TEMPLATE` 只適用於使用 ChatML 格式的模型（例如 Qwen 系列）。Llama、Gemma 等模型的格式不同，必須換成該模型自己的模板，否則會出現亂碼或停不下來。不確定時，請看模型頁面的 chat template 說明。

存檔離開 nano 後，建立 Ollama 模型：

```bash
ollama create <自訂Ollama模型名稱> -f <自訂模型簡稱>.mdf
```

看到 `success` 就完成了，重新整理 Open WebUI，下拉選單就會出現新模型。`ollama create` 會把模型複製進 Ollama 自己的儲存區，之後原本的 GGUF 檔可以刪除以節省空間。

**替代做法**：如果作者的 GGUF 已內建模板，可以略過 Modelfile，直接拉取，模板會自動處理：

```bash
ollama run hf.co/<作者>/<倉庫名稱>:<量化,例如 Q4_K_M>
```

### 4. 確認真的有用到 GPU

載入模型並對話一句後：

```bash
ollama ps
nvidia-smi
```

- `ollama ps` 的 `PROCESSOR` 欄位應顯示 `100% GPU`；若顯示 `xx% CPU / xx% GPU`，代表模型沒有完全放進 VRAM。
- `nvidia-smi` 應能看到 VRAM 使用量上升。

## Open WebUI 使用

1. 檢查 Ollama 連線：管理員設定 > 連線，Ollama API 位址應為 `http://host.docker.internal:11434`，按下驗證按鈕確認通過。
2. 可在管理員設定中，為所有模型設定通用的系統提示詞。
3. 進入 設定 > 模型，可個別修改每個模型的圖示、暱稱、參數、可使用的工具與能力。
4. 可新增管理員或一般使用者。

## Cloudflare Tunnel 設定

讓外部網路透過自己的網域連到 Open WebUI，不需要在路由器開 port。

### 步驟 1：準備網域並交給 Cloudflare 託管

1. 註冊免費網域（dpdns.org）。
2. 在 Cloudflare 註冊帳號，新增這個網域。
3. 回到網域服務商，把 nameserver 改成 Cloudflare 提供的兩組，等待 Cloudflare 顯示網域狀態為 Active。

**這一步要先完成**，後面 `tunnel login` 時才選得到網域。

### 步驟 2：安裝 cloudflared

**Debian**

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
```

如果套件庫失敗，改用 .deb 直接安裝：

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
```

### 步驟 3：登入 Cloudflare

```bash
cloudflared tunnel login
```

指令會印出一串網址，用 Windows 瀏覽器開啟、選擇你的網域並授權。（不要加 `sudo`，憑證會存到 `~/.cloudflared/cert.pem`。）

### 步驟 4：建立 tunnel

```bash
cloudflared tunnel create <tunnel名稱>
```

成功後會顯示 tunnel ID，並在 `~/.cloudflared/` 產生 `<tunnel-id>.json`。把設定檔放到系統目錄：

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/<tunnel-id>.json /etc/cloudflared/
sudo nano /etc/cloudflared/config.yml
```

`config.yml` 內容（`service` 的 port 要和 Open WebUI 對外的 3000 一致）：

```yaml
tunnel: <tunnel-id>
credentials-file: /etc/cloudflared/<tunnel-id>.json

ingress:
  - hostname: llm.xxx.dpdns.org
    service: http://localhost:3000
  - service: http_status:404
```

檢查設定檔，並自動建立 DNS 紀錄：

```bash
cloudflared tunnel ingress validate
cloudflared tunnel route dns <tunnel名稱> llm.xxx.dpdns.org
```

`route dns` 會自動在 Cloudflare 建立 CNAME。**不要**再到網頁手動新增同名的 CNAME，兩者會衝突。

### 步驟 5：建立 cloudflared 系統服務

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

驗證：用手機（關掉 Wi-Fi 走行動網路）開啟 `https://llm.xxx.dpdns.org`，應該看到 Open WebUI 登入頁。

## 開機後自動保持運作

Windows 重開機後 WSL 不會自己啟動，WSL 在沒有終端機連線時也可能被閒置關閉，tunnel 就會斷線。解法是用「工作排程器」建立一個登入時執行的工作：

- 觸發程序：使用者登入時
- 動作：啟動程式 `wsl.exe`，引數填 `-d Debian -- sleep infinity`
- 勾選「隱藏」，避免一直跳出視窗

WSL 啟動後，systemd 會自動帶起 Docker、Ollama、cloudflared，而 compose 裡設定了 `restart: always`，Open WebUI 也會跟著起來。

## 更新方式

- Ollama：重新執行安裝腳本 `curl -fsSL https://ollama.com/install.sh | sh`
- Open WebUI：在 `~/open-webui` 執行 `docker compose pull && docker compose up -d`，之後可用 `docker image prune` 清掉舊映像
- cloudflared：`sudo apt update && sudo apt install --only-upgrade cloudflared`

## 常見問題

| 症狀 | 可能原因 | 解法 |
| --- | --- | --- |
| WSL 內 `nvidia-smi` 找不到指令或無輸出 | Windows 驅動未安裝或過舊；誤在 WSL 內裝了 Linux 驅動 | 更新 Windows 端驅動；移除 WSL 內的 Linux 顯卡驅動 |
| `docker ps` 出現 permission denied | 尚未加入 docker 群組，或群組變更未生效 | 執行 `sudo usermod -aG docker $USER`，再 `wsl --shutdown` 重新進入 |
| `docker: Cannot connect to the Docker daemon` | systemd 沒啟用或 Docker 沒啟動 | 確認 `/etc/wsl.conf` 有 `systemd=true` 且已 `wsl --shutdown`；`sudo systemctl start docker` |
| `docker run --gpus all` 失敗 | Container Toolkit 未設定 runtime，或 CUDA 映像 tag 不存在 | 重新執行 `nvidia-ctk runtime configure` 並重啟 docker；改用較新的 tag |
| Open WebUI 下拉選單沒有模型 | Ollama 只監聽 127.0.0.1，容器連不到 | 依步驟 5 設定 `OLLAMA_HOST=0.0.0.0` 並重啟 Ollama |
| 模型輸出亂碼或一直停不下來 | Modelfile 的模板與 stop 不符合該模型格式 | 換成模型自己的模板，或改用 `ollama run hf.co/...` 直接拉取 |
| 回應很慢，`ollama ps` 顯示有 CPU 比例 | 模型加上上下文放不進 VRAM | 改選較小量化或較小模型，降低 `num_ctx` |
| 模型載入很慢 | 模型檔放在 `/mnt/c` 等 Windows 路徑 | 放在 WSL 內的 Linux 檔案系統（如 `~/models`） |
| Cloudflare 顯示 1033 或 502 | WSL 已被關閉、cloudflared 沒在跑，或 `config.yml` 的 port 不對 | 確認排程器工作有執行；`systemctl status cloudflared`；確認 port 是 3000 |
| `tunnel route dns` 說紀錄已存在 | 先前手動建立了同名 CNAME | 到 Cloudflare 網頁刪除該紀錄後重跑 |

## 安全性提醒

- Tunnel 一開就是公開在網路上，務必先建立好管理員帳號並關閉開放註冊（見「步驟 7」）。
- 設定 `WEBUI_SECRET_KEY`，不要使用預設值。
- 需要更高安全性時，可在 Cloudflare Zero Trust 為該網域加上 Access 驗證。
- Ollama 的 11434 port 不要透過 tunnel 對外開放。

## 注意事項

- 模型檔放在 WSL 的 Linux 檔案系統（vhdx）內，不要放 `/mnt/c`，否則讀取很慢。
- vhdx 檔案只會長大不會自動縮小，刪除模型後空間不會立刻還給 Windows，需要另外壓縮。
- 工具更新很快，若指令與實際畫面對不上，請先確認版本，再對照各工具的官方文件。
