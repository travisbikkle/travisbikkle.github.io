# 使用 Coder 打造自己的云开发环境（在中国大陆）


## 使用场景

1. 你需要在不同的开发环境中来回切换
2. 你有很多台电脑，你需要在这些电脑上安装完全一样的开发环境
3. 你出去游玩，手里只有一台平板电脑，但是你的项目出现问题需要紧急修改一部分代码
4. 你是一名开发体验提升工程师（这个岗位是我编的，一般指大公司里面那些专门为开发人员开发工具的开发人员），需要帮助用户随时按需创建完整的开发环境
5. 你希望了解类似于亚马逊、腾讯云是如何快速生成你需要的服务器或轻量服务器
6. 你单纯的喜欢折腾

## Coder 是什么

Coder 基于 [Terraform][1]，但是声称不需要深入了解 Terraform 就可以使用 Coder。

![img.png](/images/posts/coder/20240830-terraform.png)

Terraform 是什么呢？它是一个使用少量配置文件就能在各种云服务商（provider）生成节点的 Infrastructure as Code 工具。

它充当你和腾讯云、亚马逊云之间的一个中间人，你只需要提供配置（可复用，module），描述你需要什么样的节点，以及你的一些登录凭证，它就能帮你创建实例。

在社区里面有将近数百种种 provider，阿里云、腾讯云都在里面。你也可以写provider，对接任意服务商，包括本机部署的k8s或者仅仅一个docker也行，
当然，少不了本文的重点proxmox（pve）。

### 那我们为什么不去直接使用 Terraform？

Coder 在 terraform 基础上做了一些对人类友好的工作，比如一个简单的web界面（你可以通过配置文件自定义web界面的菜单、输入窗口），
一些用户、权限管理（企业版支持）、一些[模板](#Templates)、一个[工作空间](#Workspace)管理界面，你还可以直接打开命令行窗口、code
server（在线版的vscode）。

![img.png](/images/posts/coder/240830-coder-showcase.png)

### 个人开发者为什么不选择dev containers？

首先，其实 coder 支持 dev containers。

其次，在一些你无法安装 docker 的场景，或者你不希望在自己漂亮的 mac 或 win 系统中安装docker，那么 coder 就很适合你。

### Coder 的概念

你可以根据 Templates 创建 Workspace，Workspace 就是你的开发环境。

你在 Templates 中可以定义开发环境将部署在 aws 或是你的一台 pve 上，并且还能定义参数，在创建 Workspace 的时候输入 cpu
个数或内存大小等。

甚至还能定义 Workspace 的健康状态检查，并显示到前台界面上。

#### Templates

![img.png](/images/posts/coder/templates.png)

#### Workspace

![img.png](/images/posts/coder/workspace.png)

## 结论在前

* Coder 仍然有[很多不足](#不足之处)的地方。
* Coder 在中国大陆使用比较困难（这也是本文主要要解决的问题），在中国有一些免费的替代品可供选择，比如腾讯云的 code studio，但是能够自己搭建这样的工具永远是一个可靠的选择。
* Coder 适合中小型企业搭建安全可靠的私有云开发环境，仅适合部分有类似需求的个人开发者尝试。

## 如何安装

Coder 包含一个执行文件，和一个 postgres 数据库。

但是 Coder 开始编辑模板之后，需要下载一些 provider，而这些 provider 在中国是下载不下来的。

因此我建议用[这个仓库](https://github.com/dev-easily/coder-templates)的 Dockerfile 和 docker-compose.yaml 文件来自己构建镜像安装。

那么问题来了，在中国大陆 docker 根本不能用，怎么办呢？

### 如何在中国大陆使用 docker 的提示

我只能给出一点提示，不能全盘托出，不然都没得用，请注意看下面的每句话：

1. docker 是一个下载、构建、上传镜像的软件，它本身是免费的
2. 它可以指向任何镜像源，比如你在网上搜索到的各种大学、公司的镜像源。当然，截止到2024年8月，这些镜像应该全部不能用了
3. 镜像源可以私人部署，也可以搜搜看有服务商提供免费实例，可以上传300个镜像
4. 使用 docker login 可以登录到你自己的私有源，你可以用 docker push 将你常用的镜像上传上去
5. 利用 github action，docker login，docker push 将外网镜像**自动**推送到你自己的镜像源
   github 上有这样的 action，你只需要填写自己的镜像源登录凭证，在 issue 中按照固定格式写 issue，就可以自动推送到你的镜像源。然后你就可以在
   任何地方使用你自己的镜像源了。
   并且 pull 下来镜像之后，你可以通过 docker tag 改名，让它假冒成从 docker hub 下载下来的镜像。

### 开始安装 Coder

终于进入正题了。

你需要一台安装有 docker compose 的 linux 环境，本文使用 ubuntu 22.04 lts。

如果你已经有 linux 虚拟机，可以参考以下步骤做一些修改，安装 docker compose（注意docker-compose 和 docker compose 不是一个东西）。

```bash
# 卸载 ubuntu2204 旧的 docker, 安装新 docker 和 docker compose
sudo apt-get remove docker.io containerd runc
sudo curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository &#34;deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable&#34; -y
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
sudo systemctl start docker
sudo systemctl status docker
sudo systemctl enable docker
sudo chmod 777 /var/run/docker.sock # 偷懒
docker --version

## 如果你打算不下载任何公共镜像，自己的私有仓库已经足够，下面的步骤可以省略
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json &lt;&lt;-&#39;EOF&#39;
{
  &#34;registry-mirrors&#34;: [&#34;你搜索到的mirror，注意私有部署的源不要写在这里，这里只是官方镜像的mirror备份，私有源应该使用 docker login&#34;]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 安装 Coder

现在 docker compose 已经就绪了，你可以下载[这个仓库](https://github.com/dev-easily/coder-templates)。

```bash
git clone git@github.com:dev-easily/coder-templates.git
cd coder-templates
# 构建镜像 这一步仍然需要下载 ghcr.io/coder/coder:latest，你应该按照上面我说的方法，推送到自己的仓库
docker compose -f ./docker-compose.yaml build

# 开始运行
export CODER_VERSION=&#34;v2.14.2&#34;
export POSTGRES_USER=&#34;eucoder&#34; # 可以不改，数据库用户
export POSTGRES_PASSWORD=&#34;eucoderp&#34; # 可以不改，数据库密码
export POSTGRES_DB=&#34;eucoder&#34; # 可以不改，db名称
export CODER_ACCESS_URL=&#34;http://192.168.0.96:7080&#34; # 请修改，建议先修改成你部署 coder 的机器的地址

docker compose -f ./docker-compose.yaml up -d
```

在浏览器访问你配置的 CODER_ACCESS_URL 地址，设置一个你喜欢的用户名，邮箱
（有些模板会使用这两个值作为 git 的 user.name 和 user.email，你应该知道我在说什么）和密码即可。

## 使用 docker 或 pve provider

推荐使用 docker，docker 镜像构建完成后，下次启动，速度会非常快。

docker 配置起来也非常简单，适合体验。

如果你像我一样将一台商用服务器摆在家里，并且还安装了 pve，那你就选择 pve。

这两个 provider，我都已经放到了 registry.terraform.io 文件夹中，并映射到了镜像如下位置：

```
/home/coder/.terraform.d/plugins/registry.terraform.io
```

### 创建 docker 开发环境
1. 将本仓库的 docker 模板压缩为 tar 压缩包。
   ```bash
   cd coder-templates/templates/docker
   tar cvf docker.tar * # 不要选择 zip，zip 会卡住
   ```
2. 点击上传模板：
   ![img.png](/images/posts/coder/create_template.png)

3. 填写模板信息
   ![img.png](/images/posts/coder/template-upload-success.png)

   建议选择和我选择的一样的图标，因为你需要一点运气。
   
   如果你在这一步没有报错，那你有30%的概率是天命人。

4. 点击 `create workspace` 来试试创建一个 workspace 吧：
   {{&lt; notice tip &gt;}}
   在进行下一步创建 workspace 之前，为了提升速度，减少遇到的错误，建议你先参考[这里](#可能出现的问题) 构建出镜像。
   {{&lt; /notice &gt;}}

   ![img.png](/images/posts/coder/create-workspace.png)

5. 使用你的全新 workspace：

   ![img.png](/images/posts/coder/create-workspace-success.png)

6. 一些提示
   在 coder 界面右上角，点击自己的头像-Account-SSH KEYS，将公钥添加到 GITHUB 等网站，即可直接下载代码。

   注意在容器中使用的 git，是特制的，没有使用 .ssh 文件夹。

#### 可能出现的问题

整个过程中你可能遇到最多的问题，是网络问题。

1. 如果你的 docker 镜像无法构建成功
   你可以先手动在 coder 安装的机器 build 好镜像：
   ```bash
   cd templates/docker/build
   docker build --build-arg USER=&#34;改成你的登录用户名&#34; .
   docker build --build-arg USER=&#34;zhaoyu&#34; .
   ```
   你也可以在安装后登录上这个 workspace 手动安装 code-server
   ![img.png](/images/posts/coder/install-code-server-manually.png)
   记得端口改成和 main.tl 里面的一样，你可以搜索 `13337`
   你也可以自己修改这个模板，修改完成后点击 build，然后 publish
   ![img.png](/images/posts/coder/edit-template.png)
   安装 code-server 的代码在 main.tf 的 line 39-55

2. 如果你的 docker 镜像可以构建成功，但是创建 workspace 卡住
   去 coder 所在的机器查看容器日志吧：
   ```bash
   docker ps -a
   docker logs -f coder-zhaoyu-1 # 更换为你的容器名称
   ```
   
   这些日志一看就是用 go 语言写的：
   ![img.png](/images/posts/coder/check-docker-logs.png)

3. 如果你的CODER_ACCESS_URL使用了https，并且使用了自签名或者letsEncrypt证书，workspace将无法启动
   容器将卡在下载 coder-agent 一步，并且vscode远程将无法连接。
   
   自签名就不说了，letsEncrypt也有问题，原理就是虽然浏览器信任了letsEncrypt，但是ubuntu和macos等os还是没有信任它的。
   
   将证书添加进系统信任列表即可（macos双击仓库中 templates/docker/build/letsEncrypt.crt，linux执行以下命令）。

   ```bash
   sudo mkdir -p /usr/local/share/ca-certificates/extra/
   sudo cp /tmp/letsEncrypt.crt /usr/local/share/ca-certificates/extra/
   sudo update-ca-certificates
   ```
   

漂亮！现在你完成了 docker 开发环境快速配置！点一下 workspace 上的几个按钮试试？

![img.png](/images/posts/coder/install-workspace-success.png)

### 创建 pve 开发环境

1. 在 pve 中创建一个虚拟机模板

   你可以在 pve 的 ui 中创建，也可以参考如下命令，创建一个 ubuntu 22.04 的 vm 模板。
   
   在 pve 机器上，使用 root 用户，执行以下命令：

   ```bash
   cd /root
   wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img # 我是在裸金属服务器安装的pve。你可能需要选择 kvm，请在此查看 https://cloud-images.ubuntu.com/jammy/current/
   VM_ID=110 ## 改成一个不存在的ID，本例以 110 为例
   STORAGE_ID=data ## 你的可能是local-lvm，请修改
   qm create $VM_ID --cores 2 --memory 4096 --name ubuntu2204vm --net0 virtio,bridge=vmbr0 # 注意这里的名字 ubuntu2204vm，将会在下一步用到
   qm set $VM_ID --scsi0 ${STORAGE_ID}:0,import-from=/root/ubuntu-22.04-server-cloudimg-amd64.img # 
   qm set $VM_ID --scsi1 ${STORAGE_ID}:cloudinit
   qm set $VM_ID --boot c --bootdisk scsi0
   qm set $VM_ID --ciuser root --cipassword 123 # 密码可改
   # qm set $VM_ID  --ipconfig0 ip=10.0.10.123/24,gw=10.0.10.1,ip6=dhcp # 这种写法是静态 ip
   qm set $VM_ID --ipconfig0 ip=dhcp,ip6=dhcp # 这种写法是 dhcp
   ```
2. 启动虚拟机，注意安装过程中可以选择 ubuntu 的源仓库，建议选择（需要手动输入）。
3. 安装成功后，在 pve 中将该 vm 转换为模板。
4. 修改 main.tf 中的变量
   在 main.tf 中搜索&#34;请修改&#34;字样，请将这些字样全部替换。
   
   | 修改位置                | 说明                                                                                          |
   |---------------------|---------------------------------------------------------------------------------------------|
   | pm_api_url          | pve api地址，如https://192.168.0.3:8006/api2/json                                               |
   | pm_api_token_id     | pve token id，如root@pam!changeme，你可以在pve Server View中点击 Data Center-Permissions-Api Tokens添加 |
   | pm_api_token_secret | pve token secret, 如 11111-xxxxx-yyy                                                         |
   | host                | 搜索&#34;请修改：pve主机地址&#34;，你安装 pve 的机器，用于 ssh                                                          |
   | user                | pve 的机器ssh用户名                                                                               |
   | password            | pve 的机器ssh密码                                                                                |
   | target_node         | 请修改：pve 的节点（node）                                                                           |
   | clone               | 请修改：要克隆的虚拟机名称                                                                               |
   | storage             | 请修改：硬盘存储id，如local-lvm                                                                       |
5. 上传模板
   ```bash
   cd coder-templates/templates/pve
   tar cvf pve.tar * # 不要选择 zip，zip 会卡住
   ```
   ![img.png](/images/posts/coder/upload-pve-template.png)
6. 创建 workspace
   这一次和 docker 不一样，你必须要选择 cpu，内存，硬盘大小。  
   注意硬盘大小必须大于你在 pve 中创建 vm 时选择的硬盘大小，否则将无法启动。

## JetBrains GateWay
我是 JetBrains 的粉丝，JetBrains 同样也支持远程开发，
不过你需要在容器里面下载一个 Idea，我测试几回，很遗憾没有一次下载成功。

## 开发 android，flutter 等需要界面的应用
普通的 web 开发，coder 可以帮助你通过 proxy 访问页面。举例你在写一个 docusaurus 的网站，监听地址是 3000：
```
# 你应该将它启动到全0监听，否则将会显示 connect ECONNREFUSED 0.0.0.0:3000
npm run start -- --host 0.0.0.0
```

但是 flutter 这样的开发，coder 可能就无能为力了。

不过这一切等待我测试后再下结论。

## 不足之处

1. 错误提醒不够明显，比如上传zip模板的时候，会一直卡住，后台docker logs没有任何输出
   ![img.png](/images/posts/coder/stuck-when-uploading.png)
   又比如在创建 docker workspace 的时候，如果没有事先 build 好 docker 镜像，将只会看到 `docker_image.main: Still creating... [3m30s elapsed]` 而看不到任何 docker build 信息。
2. 没有批量删除 workspace 界面操作
3. 没有导出模板的功能
4. 没有明显的命令行提示
5. 没有中文
6. 界面功能很简陋

## 我为什么写这篇文章

凡是需要联网下载的东西，对于中国的程序员来说都很困难，翻墙不能解决所有的问题，因为一些命令行、CI任务、远程机器并不是总适合使用代理软件。
每隔一段时间，中国程序员往往都要耗费一到两天的时间解决下载问题，不过我们庆幸，总有路没有被堵死。

coder 显然还没有中国大陆的用户，因为它的界面没有中文，也没有中文的文档。

最重要的是使用过程中需要下载的各种依赖，在中国是无法下载的。本文旨在为有兴趣的个人开发者提供一个能够跑通的示例，希望能有更多的人使用 coder。

我没有从 coder 接受任何赞助，本文也不是一个严谨的指导文档。

[1](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/infrastructure-as-code）


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/08/coder-how-to/  

