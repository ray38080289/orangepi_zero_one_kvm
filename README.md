# Orange Pi Zero One-KVM 教學

## 硬體配置

### 板子
- Orange Pi Zero 256MB

### SD 卡
- Samsung PRO Endurance microSDXC 64GB
> 因為會用到實體 swap，建議盡量使用耐寫的卡。

### 擷取裝置
- 支援 UVC

---

## 系統 image 選擇

- 使用 Johan Gunnarsson 提供的 Debian Bookworm
(這個 image 沒有 wifi)
image：
  [下載連結](https://sd-card-images.johang.se/boards/orange_pi_zero.html)
- 參考 Reddit 討論：
  [Orange Pi Zero as USB Gadget](https://www.reddit.com/r/OrangePI/comments/16mcrqu/orange_pi_zero_as_usb_gadget/)
  > mradermacher_hf 表示 armbian 設備樹存在缺陷使用otg需手動修改，這也是為什麼所有 orangepi zero 都需要使用修改後的版本
- 為什麼不用 jacobbar/fruity-pikvm 的 image？
  > 因為該 image 容易系統凍結（armbian 問題），即使手動固定 CPU 頻率依舊會發生。
  > 詳見 [Armbian 討論串](https://forum.armbian.com/topic/12647-orange-pi-zero-freezing-randomly/)

---

## 開始教學

### 1. 合併映像檔
下載 `debian-bookworm-armhf-wi3soo.bin.gz` 與 `boot-orange_pi_zero.bin.gz`。

> Windows 可用 WSL 或 VM 執行下列指令

```bash
zcat boot-orange_pi_zero.bin.gz debian-bookworm-armhf-wi3soo.bin.gz > sd-card.img
```

![合併映像檔](image-1.png)

---

### 2. 使用 balenaEtcher 燒錄
選取 `sd-card.img` 進行燒錄。

![balenaEtcher 操作](image-2.png)

---

### 3. 連線與登入
燒錄完成後接上網路線，預設為 DHCP，從路由器查詢 IP。

- 使用 SSH 連線
- 帳號：`root`  
- 密碼：`wi3soo`（密碼見下載網頁）

> 若出現憑證錯誤，需刪除舊的 key（詳情可問 GPT）

![SSH 連線](image-3.png)

正常會出現接受新 key 提示：

![接受新 key](image-4.png)

本教學全程以 root 操作，所有命令不須加 sudo。

![root 操作](image-5.png)

---

### 4. 更改密碼

```bash
passwd
```

![更改密碼](image-6.png)

---

### 5. 更新軟體源

```bash
apt update
```

![更新軟體源](image-7.png)

---

### 6. 擴大分區空間

> 這個 image 分區不會自動擴大，需手動操作。

安裝 cloud-guest-utils：

```bash
apt install cloud-guest-utils -y
```

![安裝 cloud-guest-utils](image-9.png)

查看 root 對應的 device（如 `/dev/mmcblk0`）與 part（如 2）：

![查看分區](image-8.png)

執行：

```bash
growpart /dev/mmcblk0 2
resize2fs /dev/mmcblk0p2
```

---

### 7. 啟用 zram

預設情況：

![zram 預設](image-16.png)

安裝 zram-tools：

```bash
apt install zram-tools
```

![安裝 zram-tools](image-14.png)

裝完預設會是 lz4 256MB，但 CPU 太弱不適合全壓，因此建議改成 50%。

編輯 `/etc/default/zramswap`：

```bash
nano /etc/default/zramswap
```

![編輯 zramswap](image-15.png)

改完後重啟服務：

```bash
systemctl restart zramswap
```

![重啟 zramswap](image-17.png)

---

### 8. 加上實體 swap

建立 swap file：

```bash
fallocate -l 1G /swapfile && chmod 600 /swapfile &&  mkswap /swapfile && swapon /swapfile
```

![建立 swap file](image-18.png)

開機自動啟用：

```bash
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab && tail /etc/fstab
```

重啟：

```bash
reboot
```

---

### 9. 安裝 Docker

```bash
apt install curl -y && curl -fsSL https://get.docker.com | bash
```

![安裝 Docker](image-19.png)
![Docker 安裝完成](image-20.png)

---

### 10. 啟動 one kvm docker 並插入擷取裝置

> VIDEONUM 依具體 id 設定

```bash
docker run -itd --name kvmd --privileged=true -v /lib/modules:/lib/modules:ro -v /dev:/dev -v /sys/kernel/config:/sys/kernel/config -v /root/kvmd_config:/etc/kvmd -e OTG=1 -e USERNAME=admin -e PASSWORD=admin -e VIDEONUM=0 -e AUDIONUM=0 -p 8080:8080 -p 4430:4430 -p 5900:5900 -p 623:623 silentwind0/kvmd
```

![啟動 kvmd](image-21.png)
![kvmd 執行中](image-26.png)
![kvmd 執行中2](image-27.png)

---

### 11. 查詢 docker logs

```bash
docker logs kvmd
```

若出現下圖，表示擷取裝置 id 錯誤或未插入：

![擷取裝置錯誤](image-22.png)

將 docker 停止並刪除，並刪除設定檔（否則會導致使用相同擷取裝置）：

```bash
docker stop kvmd && docker rm kvmd && rm -rf /root/kvmd_config
```

![刪除 kvmd 設定](image-25.png)
