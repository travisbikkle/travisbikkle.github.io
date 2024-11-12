# 使用 Coder 打造自己的雲開發環境（在中國大陸）


## 使用場景

1. 你需要在不同的開發環境中來回切換
2. 你有很多臺電腦，你需要在這些電腦上安裝完全一樣的開發環境
3. 你出去遊玩，手裏只有一臺平板電腦，但是你的項目出現問題需要緊急修改一部分代碼
4. 你是一名開發體驗提升工程師（這個崗位是我編的，一般指大公司裏面那些專門爲開發人員開發工具的開發人員），需要幫助用戶隨時按需創建完整的開發環境
5. 你希望瞭解類似於亞馬遜、騰訊雲是如何快速生成你需要的服務器或輕量服務器
6. 你單純的喜歡折騰

## Coder 是什麼

Coder 基於 [Terraform][1]，但是聲稱不需要深入瞭解 Terraform 就可以使用 Coder。

![img.png](/images/posts/coder/20240830-terraform.png)

Terraform 是什麼呢？它是一個使用少量配置文件就能在各種雲服務商（provider）生成節點的 Infrastructure as Code 工具。

它充當你和騰訊雲、亞馬遜雲之間的一箇中間人，你只需要提供配置（可複用，module），描述你需要什麼樣的節點，以及你的一些登錄憑證，它就能幫你創建實例。

在社區裏面有將近數百種種 provider，阿里雲、騰訊雲都在裏面。你也可以寫provider，對接任意服務商，包括本機部署的k8s或者僅僅一個docker也行，
當然，少不了本文的重點proxmox（pve）。

### 那我們爲什麼不去直接使用 Terraform？

Coder 在 terraform 基礎上做了一些對人類友好的工作，比如一個簡單的web界面（你可以通過配置文件自定義web界面的菜單、輸入窗口），
一些用戶、權限管理（企業版支持）、一些[模板](#Templates)、一個[工作空間](#Workspace)管理界面，你還可以直接打開命令行窗口、code
server（在線版的vscode）。

![img.png](/images/posts/coder/240830-coder-showcase.png)

### 個人開發者爲什麼不選擇dev containers？

首先，其實 coder 支持 dev containers。

其次，在一些你無法安裝 docker 的場景，或者你不希望在自己漂亮的 mac 或 win 系統中安裝docker，那麼 coder 就很適合你。

### Coder 的概念

你可以根據 Templates 創建 Workspace，Workspace 就是你的開發環境。

你在 Templates 中可以定義開發環境將部署在 aws 或是你的一臺 pve 上，並且還能定義參數，在創建 Workspace 的時候輸入 cpu
個數或內存大小等。

甚至還能定義 Workspace 的健康狀態檢查，並顯示到前臺界面上。

#### Templates

![img.png](/images/posts/coder/templates.png)

#### Workspace

![img.png](/images/posts/coder/workspace.png)

## 結論在前

* Coder 仍然有[很多不足](#不足之處)的地方。
* Coder 在中國大陸使用比較困難（這也是本文主要要解決的問題），在中國有一些免費的替代品可供選擇，比如騰訊雲的 code studio，但是能夠自己搭建這樣的工具永遠是一個可靠的選擇。
* Coder 適合中小型企業搭建安全可靠的私有云開發環境，僅適合部分有類似需求的個人開發者嘗試。

## 如何安裝

Coder 包含一個執行文件，和一個 postgres 數據庫。

但是 Coder 開始編輯模板之後，需要下載一些 provider，而這些 provider 在中國是下載不下來的。

因此我建議用[這個倉庫](https://github.com/dev-easily/coder-templates)的 Dockerfile 和 docker-compose.yaml 文件來自己構建鏡像安裝。

那麼問題來了，在中國大陸 docker 根本不能用，怎麼辦呢？

### 如何在中國大陸使用 docker 的提示

我只能給出一點提示，不能全盤托出，不然都沒得用，請注意看下面的每句話：

1. docker 是一個下載、構建、上傳鏡像的軟件，它本身是免費的
2. 它可以指向任何鏡像源，比如你在網上搜索到的各種大學、公司的鏡像源。當然，截止到2024年8月，這些鏡像應該全部不能用了
3. 鏡像源可以私人部署，也可以搜搜看有服務商提供免費實例，可以上傳300個鏡像
4. 使用 docker login 可以登錄到你自己的私有源，你可以用 docker push 將你常用的鏡像上傳上去
5. 利用 github action，docker login，docker push 將外網鏡像**自動**推送到你自己的鏡像源
   github 上有這樣的 action，你只需要填寫自己的鏡像源登錄憑證，在 issue 中按照固定格式寫 issue，就可以自動推送到你的鏡像源。然後你就可以在
   任何地方使用你自己的鏡像源了。
   並且 pull 下來鏡像之後，你可以通過 docker tag 改名，讓它假冒成從 docker hub 下載下來的鏡像。

### 開始安裝 Coder

終於進入正題了。

你需要一臺安裝有 docker compose 的 linux 環境，本文使用 ubuntu 22.04 lts。

如果你已經有 linux 虛擬機，可以參考以下步驟做一些修改，安裝 docker compose（注意docker-compose 和 docker compose 不是一個東西）。

```bash
# 卸載 ubuntu2204 舊的 docker, 安裝新 docker 和 docker compose
sudo apt-get remove docker.io containerd runc
sudo curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository &#34;deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable&#34; -y
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
sudo systemctl start docker
sudo systemctl status docker
sudo systemctl enable docker
sudo chmod 777 /var/run/docker.sock # 偷懶
docker --version

## 如果你打算不下載任何公共鏡像，自己的私有倉庫已經足夠，下面的步驟可以省略
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json &lt;&lt;-&#39;EOF&#39;
{
  &#34;registry-mirrors&#34;: [&#34;你搜索到的mirror，注意私有部署的源不要寫在這裏，這裏只是官方鏡像的mirror備份，私有源應該使用 docker login&#34;]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 安裝 Coder

現在 docker compose 已經就緒了，你可以下載[這個倉庫](https://github.com/dev-easily/coder-templates)。

```bash
git clone git@github.com:dev-easily/coder-templates.git
cd coder-templates
# 構建鏡像 這一步仍然需要下載 ghcr.io/coder/coder:latest，你應該按照上面我說的方法，推送到自己的倉庫
docker compose -f ./docker-compose.yaml build

# 開始運行
export CODER_VERSION=&#34;v2.14.2&#34;
export POSTGRES_USER=&#34;eucoder&#34; # 可以不改，數據庫用戶
export POSTGRES_PASSWORD=&#34;eucoderp&#34; # 可以不改，數據庫密碼
export POSTGRES_DB=&#34;eucoder&#34; # 可以不改，db名稱
export CODER_ACCESS_URL=&#34;http://192.168.0.96:7080&#34; # 請修改，建議先修改成你部署 coder 的機器的地址

docker compose -f ./docker-compose.yaml up -d
```

在瀏覽器訪問你配置的 CODER_ACCESS_URL 地址，設置一個你喜歡的用戶名，郵箱
（有些模板會使用這兩個值作爲 git 的 user.name 和 user.email，你應該知道我在說什麼）和密碼即可。

## 使用 docker 或 pve provider

推薦使用 docker，docker 鏡像構建完成後，下次啓動，速度會非常快。

docker 配置起來也非常簡單，適合體驗。

如果你像我一樣將一臺商用服務器擺在家裏，並且還安裝了 pve，那你就選擇 pve。

這兩個 provider，我都已經放到了 registry.terraform.io 文件夾中，並映射到了鏡像如下位置：

```
/home/coder/.terraform.d/plugins/registry.terraform.io
```

### 創建 docker 開發環境
1. 將本倉庫的 docker 模板壓縮爲 tar 壓縮包。
   ```bash
   cd coder-templates/templates/docker
   tar cvf docker.tar * # 不要選擇 zip，zip 會卡住
   ```
2. 點擊上傳模板：
   ![img.png](/images/posts/coder/create_template.png)

3. 填寫模板信息
   ![img.png](/images/posts/coder/template-upload-success.png)

   建議選擇和我選擇的一樣的圖標，因爲你需要一點運氣。
   
   如果你在這一步沒有報錯，那你有30%的概率是天命人。

4. 點擊 `create workspace` 來試試創建一個 workspace 吧：
   {{&lt; notice tip &gt;}}
   在進行下一步創建 workspace 之前，爲了提升速度，減少遇到的錯誤，建議你先參考[這裏](#可能出現的問題) 構建出鏡像。
   {{&lt; /notice &gt;}}

   ![img.png](/images/posts/coder/create-workspace.png)

5. 使用你的全新 workspace：

   ![img.png](/images/posts/coder/create-workspace-success.png)

6. 一些提示
   在 coder 界面右上角，點擊自己的頭像-Account-SSH KEYS，將公鑰添加到 GITHUB 等網站，即可直接下載代碼。

   注意在容器中使用的 git，是特製的，沒有使用 .ssh 文件夾。

#### 可能出現的問題

整個過程中你可能遇到最多的問題，是網絡問題。

1. 如果你的 docker 鏡像無法構建成功
   你可以先手動在 coder 安裝的機器 build 好鏡像：
   ```bash
   cd templates/docker/build
   docker build --build-arg USER=&#34;改成你的登錄用戶名&#34; .
   docker build --build-arg USER=&#34;zhaoyu&#34; .
   ```
   你也可以在安裝後登錄上這個 workspace 手動安裝 code-server
   ![img.png](/images/posts/coder/install-code-server-manually.png)
   記得端口改成和 main.tl 裏面的一樣，你可以搜索 `13337`
   你也可以自己修改這個模板，修改完成後點擊 build，然後 publish
   ![img.png](/images/posts/coder/edit-template.png)
   安裝 code-server 的代碼在 main.tf 的 line 39-55

2. 如果你的 docker 鏡像可以構建成功，但是創建 workspace 卡住
   去 coder 所在的機器查看容器日誌吧：
   ```bash
   docker ps -a
   docker logs -f coder-zhaoyu-1 # 更換爲你的容器名稱
   ```
   
   這些日誌一看就是用 go 語言寫的：
   ![img.png](/images/posts/coder/check-docker-logs.png)

3. 如果你的CODER_ACCESS_URL使用了https，並且使用了自簽名或者letsEncrypt證書，workspace將無法啓動
   容器將卡在下載 coder-agent 一步，並且vscode遠程將無法連接。
   
   自簽名就不說了，letsEncrypt也有問題，原理就是雖然瀏覽器信任了letsEncrypt，但是ubuntu和macos等os還是沒有信任它的。
   
   將證書添加進系統信任列表即可（macos雙擊倉庫中 templates/docker/build/letsEncrypt.crt，linux執行以下命令）。

   ```bash
   sudo mkdir -p /usr/local/share/ca-certificates/extra/
   sudo cp /tmp/letsEncrypt.crt /usr/local/share/ca-certificates/extra/
   sudo update-ca-certificates
   ```
   

漂亮！現在你完成了 docker 開發環境快速配置！點一下 workspace 上的幾個按鈕試試？

![img.png](/images/posts/coder/install-workspace-success.png)

### 創建 pve 開發環境

1. 在 pve 中創建一個虛擬機模板

   你可以在 pve 的 ui 中創建，也可以參考如下命令，創建一個 ubuntu 22.04 的 vm 模板。
   
   在 pve 機器上，使用 root 用戶，執行以下命令：

   ```bash
   cd /root
   wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img # 我是在裸金屬服務器安裝的pve。你可能需要選擇 kvm，請在此查看 https://cloud-images.ubuntu.com/jammy/current/
   VM_ID=110 ## 改成一個不存在的ID，本例以 110 爲例
   STORAGE_ID=data ## 你的可能是local-lvm，請修改
   qm create $VM_ID --cores 2 --memory 4096 --name ubuntu2204vm --net0 virtio,bridge=vmbr0 # 注意這裏的名字 ubuntu2204vm，將會在下一步用到
   qm set $VM_ID --scsi0 ${STORAGE_ID}:0,import-from=/root/ubuntu-22.04-server-cloudimg-amd64.img # 
   qm set $VM_ID --scsi1 ${STORAGE_ID}:cloudinit
   qm set $VM_ID --boot c --bootdisk scsi0
   qm set $VM_ID --ciuser root --cipassword 123 # 密碼可改
   # qm set $VM_ID  --ipconfig0 ip=10.0.10.123/24,gw=10.0.10.1,ip6=dhcp # 這種寫法是靜態 ip
   qm set $VM_ID --ipconfig0 ip=dhcp,ip6=dhcp # 這種寫法是 dhcp
   ```
2. 啓動虛擬機，注意安裝過程中可以選擇 ubuntu 的源倉庫，建議選擇（需要手動輸入）。
3. 安裝成功後，在 pve 中將該 vm 轉換爲模板。
4. 修改 main.tf 中的變量
   在 main.tf 中搜索&#34;請修改&#34;字樣，請將這些字樣全部替換。
   
   | 修改位置                | 說明                                                                                          |
   |---------------------|---------------------------------------------------------------------------------------------|
   | pm_api_url          | pve api地址，如https://192.168.0.3:8006/api2/json                                               |
   | pm_api_token_id     | pve token id，如root@pam!changeme，你可以在pve Server View中點擊 Data Center-Permissions-Api Tokens添加 |
   | pm_api_token_secret | pve token secret, 如 11111-xxxxx-yyy                                                         |
   | host                | 搜索&#34;請修改：pve主機地址&#34;，你安裝 pve 的機器，用於 ssh                                                          |
   | user                | pve 的機器ssh用戶名                                                                               |
   | password            | pve 的機器ssh密碼                                                                                |
   | target_node         | 請修改：pve 的節點（node）                                                                           |
   | clone               | 請修改：要克隆的虛擬機名稱                                                                               |
   | storage             | 請修改：硬盤存儲id，如local-lvm                                                                       |
5. 上傳模板
   ```bash
   cd coder-templates/templates/pve
   tar cvf pve.tar * # 不要選擇 zip，zip 會卡住
   ```
   ![img.png](/images/posts/coder/upload-pve-template.png)
6. 創建 workspace
   這一次和 docker 不一樣，你必須要選擇 cpu，內存，硬盤大小。  
   注意硬盤大小必須大於你在 pve 中創建 vm 時選擇的硬盤大小，否則將無法啓動。

## JetBrains GateWay
我是 JetBrains 的粉絲，JetBrains 同樣也支持遠程開發，
不過你需要在容器裏面下載一個 Idea，我測試幾回，很遺憾沒有一次下載成功。

## 開發 android，flutter 等需要界面的應用
普通的 web 開發，coder 可以幫助你通過 proxy 訪問頁面。舉例你在寫一個 docusaurus 的網站，監聽地址是 3000：
```
# 你應該將它啓動到全0監聽，否則將會顯示 connect ECONNREFUSED 0.0.0.0:3000
npm run start -- --host 0.0.0.0
```

但是 flutter 這樣的開發，coder 可能就無能爲力了。

不過這一切等待我測試後再下結論。

## 不足之處

1. 錯誤提醒不夠明顯，比如上傳zip模板的時候，會一直卡住，後臺docker logs沒有任何輸出
   ![img.png](/images/posts/coder/stuck-when-uploading.png)
   又比如在創建 docker workspace 的時候，如果沒有事先 build 好 docker 鏡像，將只會看到 `docker_image.main: Still creating... [3m30s elapsed]` 而看不到任何 docker build 信息。
2. 沒有批量刪除 workspace 界面操作
3. 沒有導出模板的功能
4. 沒有明顯的命令行提示
5. 沒有中文
6. 界面功能很簡陋

## 我爲什麼寫這篇文章

凡是需要聯網下載的東西，對於中國的程序員來說都很困難，翻牆不能解決所有的問題，因爲一些命令行、CI任務、遠程機器並不是總適合使用代理軟件。
每隔一段時間，中國程序員往往都要耗費一到兩天的時間解決下載問題，不過我們慶幸，總有路沒有被堵死。

coder 顯然還沒有中國大陸的用戶，因爲它的界面沒有中文，也沒有中文的文檔。

最重要的是使用過程中需要下載的各種依賴，在中國是無法下載的。本文旨在爲有興趣的個人開發者提供一個能夠跑通的示例，希望能有更多的人使用 coder。

我沒有從 coder 接受任何贊助，本文也不是一個嚴謹的指導文檔。

[1](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/infrastructure-as-code）


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/08/coder-how-to/  

