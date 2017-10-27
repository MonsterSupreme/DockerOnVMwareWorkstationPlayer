# Docker on VMware Workstation Player

雖然現在 Windows 10 對 Docker 已經有 Native Support，可以直接裝 Docker for Windows，但是這基本上要 Windows 10 Professional 以上的版本，而且還要開啟 Hyper-V，所以我一直都還是使用 Docker Toolbox for Windows，底層透過 Oracle VirtualBox 提供 Hypervisor 這一層。

自從升級了 Windows 10 Creative Update 之後，VirtualBox 一直怪怪的，Docker 起不來，Vagrant 也起不來。網路上看了一堆文章，都說是新的 NDIS6 Driver 有問題，所以安裝 VirtualBox 的時候，要加上 `-msiparams NETWORKTYPE=NDIS5` 參數，使用舊版的 NDIS5 Driver，不過這一招對我無效。使用 Docker Toolbox for Windows 自帶的 Oracle VirtualBox 也不行，我自己去下載舊版的 VirtualBox，太舊的版本在 Windows 10 Creative Update 之後還跑不起來，可以跑起來的版本搭配 Vagrant 或是 Docker 依舊不能運作。

不喜歡 Hyper-V，VirtualBox 又有問題，那就只剩下 VMware Player。Vagrant 支援 VMware 的部分我記得是要另外收費的，Docker 的話在 [Machine drivers](https://docs.docker.com/machine/drivers/) 這一頁上說對 VMware Workstation 有 Gonzalo Peci 寫的 [Unofficial Plugin](https://github.com/pecigonzalo/docker-machine-vmwareworkstation)，爬文看了 Issue 之後，發現這個 Plugin 並沒有要求一定要 VMware Workstation Pro 版本，免費的 VMware Workstation Player 版本也支援，所以就到 [Releases](https://github.com/pecigonzalo/docker-machine-vmwareworkstation/releases) 下載 `docker-machine-driver-vmwareworkstation.exe` 檔案最新的版本，放在 Docker Toolbox 安裝的目錄 `C:\Program Files\Docker Toolbox`，跟 `docker-machine.exe` 檔案放在一起。然後，再把這個目錄下的 `start.sh` 檔案先備份起來，改成 Gonzalo Peci 網頁上提供的寫法：

```bash
#!/bin/bash

export PATH="$PATH:/mnt/c/Program Files (x86)/VMware/VMware Workstation"

trap '[ "$?" -eq 0 ] || read -p "Looks like something went wrong in step ´$STEP´... Press any key to continue..."' EXIT

VM=${DOCKER_MACHINE_NAME-default}
DOCKER_MACHINE=./docker-machine.exe

BLUE='\033[1;34m'
GREEN='\033[0;32m'
NC='\033[0m'


if [ ! -f "${DOCKER_MACHINE}" ]; then
  echo "Docker Machine is not installed. Please re-run the Toolbox Installer and try again."
  exit 1
fi

vmrun.exe list | grep \""${VM}"\" &> /dev/null
VM_EXISTS_CODE=$?

set -e

STEP="Checking if machine $VM exists"
if [ $VM_EXISTS_CODE -eq 1 ]; then
  "${DOCKER_MACHINE}" rm -f "${VM}" &> /dev/null || :
  rm -rf ~/.docker/machine/machines/"${VM}"
  #set proxy variables if they exists
  if [ -n ${HTTP_PROXY+x} ]; then
    PROXY_ENV="$PROXY_ENV --engine-env HTTP_PROXY=$HTTP_PROXY"
  fi
  if [ -n ${HTTPS_PROXY+x} ]; then
    PROXY_ENV="$PROXY_ENV --engine-env HTTPS_PROXY=$HTTPS_PROXY"
  fi
  if [ -n ${NO_PROXY+x} ]; then
    PROXY_ENV="$PROXY_ENV --engine-env NO_PROXY=$NO_PROXY"
  fi  
  "${DOCKER_MACHINE}" create -d vmwareworkstation $PROXY_ENV "${VM}"
fi

STEP="Checking status on $VM"
VM_STATUS="$(${DOCKER_MACHINE} status ${VM} 2>&1)"
if [ "${VM_STATUS}" != "Running" ]; then
  "${DOCKER_MACHINE}" start "${VM}"
  yes | "${DOCKER_MACHINE}" regenerate-certs "${VM}"
fi

STEP="Setting env"
eval "$(${DOCKER_MACHINE} env --shell=bash ${VM})"

STEP="Finalize"
clear
cat << EOF


                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/

EOF
echo -e "${BLUE}docker${NC} is configured to use the ${GREEN}${VM}${NC} machine with IP ${GREEN}$(${DOCKER_MACHINE} ip ${VM})${NC}"
echo "For help getting started, check out the docs at https://docs.docker.com"
echo
cd

docker () {
  MSYS_NO_PATHCONV=1 docker.exe "$@"
}
export -f docker

if [ $# -eq 0 ]; then
  echo "Start interactive shell"
  exec "$BASH" --login -i
else
  echo "Start shell with command"
  exec "$BASH" -c "$*"
fi
```

如果有購買 VMware Workstation Pro 版本的 License 的話，這時候就可以執行 Docker Quickstart Terminal，透過 Boot2Docker 建立 `default` VM，從此過著幸福快樂的生活。

如果使用 VMware Workstation Player 版本的話，問題就來了：

第一，`docker-machine-driver-vmwareworkstation.exe` 檔案會用到 `vmrun.exe` 檔案，這個檔案可以透過 Command-Line 的型式控制 VMware Workstation 建立/執行/停止/刪除 VM，功能就跟 VirtualBox 的 `VBoxManage.exe` 檔案是一樣的。VMware Workstation Pro 版本裝好之後就會有 `vmrun.exe` 這個檔案，但是 VMware Workstation Player 版本沒有。不過沒關係，`vmrun.exe` 檔案其實是免費的，它是 VMware VIX API 的一部份，在 [Download VMware Workstation Player](https://my.vmware.com/en/web/vmware/free#desktop_end_user_computing/vmware_workstation_player/14_0|PLAYER-1400|product_downloads) 頁面的 Product Downloads 標籤頁下載 VMware Workstation Player 的時候，點選旁邊的 Drivers & Tools 標籤頁，就可以下載 VMware VIX。安裝完成之後，就有 `vmrun.exe` 檔案了。

第二，根據上面建議的 `start.sh` 檔案內容來看，`docker-machine-driver-vmwareworkstation.exe` 檔案預設會到 `/mnt/c/Program Files (x86)/VMware/VMware Workstation` 目錄去找 `vmrun.exe` 檔案，如果裝 VMware Workstation Pro 版本沒有問題，但我們是透過 VMware VIX 來提供的。這部分有兩個解法。如果用 Git 提供的 `bash.exe` 檔案的話，其實上面的路徑開頭不是 `/mnt/c/...`，而是 `/c/...`，VMware VIX 安裝的目錄名稱是 `VMware VIX`，所以整個合起來，路徑應該改成 `/c/Program Files (x86)/VMware/VMware VIX` 才對。雖然 VMware Workstation 好像在很久以前就是 x64 版本，但是它的安裝位置一直都放在 `Program Files (x86)`，這部分是沒問題的。如果不想改 `start.sh` 檔案的話，另一個解法其實可以定義 `VMWARE_HOME` 環境變數，內容是 `C:\Program Files (x86)\VMware\VMware VIX`，這樣也可以。

第三，這時候如果就去執行 Docker Quickstart Terminal 的話，會發現 Boot2Docker 新建的 `default` VM 沒有 `default.vmdk` 檔案，也就是沒有 Disk Image，所以 VM 會跑不起來。這個問題一樣是因為 VMware Workstation Pro 版本有提供 `vmware-vdiskmanager.exe` 檔案，這個檔案就是 VMware Virtual Disk Manager，用來管理 VM 的 Disk Image，但是 VMware Workstation Player 很巧又剛好沒有。不過一樣沒關係，`vmware-vdiskmanager.exe` 檔案剛好又是免費的，它是 Virtual Disk Development Kit (VDDK) 的一部份，在 [VDDK for vSphere 6.5](https://code.vmware.com/web/sdk/65/vddk) 可以下載。因為 VDDK 是以 ZIP 或 TAR 檔案型式發佈，所以必須自行把它解壓縮，統一放到 `C:\Program Files (x86)\VMware\VDDK` 好了。

第四，雖然檔案都找齊了，為了擔心 `PATH` 沒設定好找不到，乾脆全部自己設定，`PATH` 環境變數要加上以下的路徑：

```
C:\Program Files (x86)\VMware\VMware Player
C:\Program Files (x86)\VMware\VMware VIX
C:\Program Files (x86)\VMware\VDDK\bin
```

第五，Windows 內建沒有 Bash 與 SSH 的功能，但是安裝 Docker Toolbox for Windows 時順便安裝的 Git，裡面其實都有提供，所以 `PATH` 環境變數要再加上以下的路徑：

```
C:\Program Files\Git\bin
C:\Program Files\Git\usr\bin
```

OK，現在執行 Docker Quickstart Terminal：

```bash
Running pre-create checks...
Creating machine...
(default) Copying C:\Users\Monster\.docker\machine\cache\boot2docker.iso to 
    C:\Users\Monster\.docker\machine\machines\default\boot2docker.iso...
(default) Creating SSH key...
(default) Creating VM...
(default) Creating disk 'C:\Users\Monster\.docker\machine\machines\default\default.vmdk'
(default) Virtual disk creation successful.
(default) Starting default...
(default) Waiting for VM to come online...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Error creating machine: Error detecting OS: OS type not recognized
Looks like something went wrong in step ´Checking if machine default exists´... Press any key to continue...
```

不知道為什麼，在 `(default) Waiting for VM to come online...` 那邊會卡一段時間，`Waiting for SSH to be available...` 那邊也會卡一段時間，事實上如果直接在 VMware Workstation Player 啟動這個 VM，速度是很快的。

如果一切順利，人品好的話，就能夠透過 Boot2Docker 建立 `default` VM，開始進行各項 Docker 操作。不過你知道的，人生不如意之事十常八九，我自己的經驗是，十次大概只有一次會成功，八九次都像上面一樣失敗。

詭異的是，`default` VM 其實跑起來了，只是沒辦法取得 VM 的 IP，SSH 連不上去的感覺，但是可以關掉：

```bash
$ docker-machine status
Running

$ docker-machine stop
Stopping "default"...
Machine "default" was stopped.
``` 

即使運氣好成功了，這時候透過底下的指令把關掉的 `default` VM 重新啟動，結果又會掛在那邊，可是實際上是在跑的：

```bash
$ docker-machine start
Starting "default"...
Machine "default" was started.
Waiting for SSH to be available...
(這邊會卡住，必須要 Ctrl+C 強迫中斷)

$ docker-machine status
Running

$ docker-machine stop
Stopping "default"...
Machine "default" was stopped.
```

記得，如果建立 `default` VM 失敗，一定要先用 `docker-machine` 指令把 `default` VM 停掉，然後再到 `C:\Users\使用者帳號\.docker\machine\machines` 目錄下，把 `default` 子目錄整個刪除。如果有檔案被 Lock 住所以刪不掉，那就重開機再刪除。

不管怎麼樣，總是建立了信心，至少大致上是沒問題的，只是小地方要修正一下。

其實更慘的是，就算基於那十分之一左右的機率成功了，今天下班關機之後，明天想繼續操作的時候，前面的 `start.sh` 檔案又會把好不容易建立成功的 `default` VM 砍掉重練，然後，你知道的...。

就算你一直不關機，用預設值長出來的 `default` VM 的配備是：

- CPU：1 Core
- RAM：1024 MB
- HD：20 GB  

這樣的 `default` VM 很多東西都跑不起來。

辛苦在網路上爬了幾十篇文章之後，底下的執行方式大概是沒問題了。前面的所有設定都還是要做，因為那是基本功。只是那個 `start.sh` 檔案似乎是一切問題的來源，裡面很多地方銜接起來有時候不太順，預設值也不夠用，所以，就乾脆直接下指令執行就好。

第一，建立 `default` VM，2 Core CPU, 4096 MB RAM，20 GB HD：

```bash
$ docker-machine --native-ssh create -d vmwareworkstation 
    --vmwareworkstation-memory-size 4096 --vmwareworkstation-cpu-count 2 default
Running pre-create checks...
Creating machine...
(default) Copying C:\Users\Monster\.docker\machine\cache\boot2docker.iso to 
    C:\Users\Monster\.docker\machine\machines\default\boot2docker.iso...
(default) Creating SSH key...
(default) Creating VM...
(default) Creating disk 'C:\Users\Monster\.docker\machine\machines\default\default.vmdk'
(default) Virtual disk creation successful.
(default) Starting default...
(default) Waiting for VM to come online...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: 
    docker-machine env default

$ docker-machine ls
NAME      ACTIVE   DRIVER              STATE     URL                         SWARM   DOCKER        ERRORS
default   -        vmwareworkstation   Running   tcp://192.168.40.128:2376           v17.10.0-ce
```

這時候不必因為成功就急著下其他 Docker 指令，因為 Docker 相關設定還沒設好，只會有錯：

```bash
$ docker ps
error during connect: 
Get http://%2F%2F.%2Fpipe%2Fdocker_engine/v1.31/containers/json: 
open //./pipe/docker_engine: 
The system cannot find the file specified. 
In the default daemon configuration on Windows, the docker client must be run elevated to connect. 
This error may also indicate that the docker daemon is not running.
```

第二，取得並設定目前 Docker 相關設定：

```bash
$ docker-machine env default
SET DOCKER_TLS_VERIFY=1
SET DOCKER_HOST=tcp://192.168.40.128:2376
SET DOCKER_CERT_PATH=C:\Users\Monster\.docker\machine\machines\default
SET DOCKER_MACHINE_NAME=default
SET COMPOSE_CONVERT_WINDOWS_PATHS=true
REM Run this command to configure your shell:
REM     @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i

$ @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i

$ set | grep DOCKER
DOCKER_CERT_PATH=C:\Users\Monster\.docker\machine\machines\default
DOCKER_HOST=tcp://192.168.40.128:2376
DOCKER_MACHINE_NAME=default
DOCKER_TLS_VERIFY=1
DOCKER_TOOLBOX_INSTALL_PATH=C:\Program Files\Docker Toolbox
```

第三，這時候就可以開始玩 Docker 了，比方說來個 Hello, World!：

```bash
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
5b0f327be733: Pull complete
Digest: sha256:07d5f7800dfe37b8c2196c7b1c524c33808ce2e0f74e7aa00e603295ca9a0972
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```

第四，如果要開其他 Console 執行 Docker 的話，只要做好 Docker 相關設定即可：

```bash
$ @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i

$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```

第五，如果今天工作告一段落，想要關掉 Docker，明天再繼續其他操作，就透過 `docker-machine` 指令管理即可：

```bash
$ docker-machine stop                                                                                             
Stopping "default"...                                                                                             
Machine "default" was stopped.                                                                                    
                                                                                                                  
$ docker-machine start                                                                                            
Starting "default"...                                                                                             
Machine "default" was started.                                                                                    
Waiting for SSH to be available...                                                                                
Detecting the provisioner...                                                                                      
Started machines may have new IP addresses. You may need to re-run the `docker-machine env` command.              
                                                                                                                  
$ @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i                                                  
                                                                                                                  
$ docker run hello-world                                                                                          
                                                                                                                  
Hello from Docker!                                                                                                
This message shows that your installation appears to be working correctly.                                        
                                                                                                                  
To generate this message, Docker took the following steps:                                                        
 1. The Docker client contacted the Docker daemon.                                                                
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.                                         
 3. The Docker daemon created a new container from that image which runs the                                      
    executable that produces the output you are currently reading.                                                
 4. The Docker daemon streamed that output to the Docker client, which sent it                                    
    to your terminal.                                                                                             
                                                                                                                  
To try something more ambitious, you can run an Ubuntu container with:                                            
 $ docker run -it ubuntu bash                                                                                     
                                                                                                                  
Share images, automate workflows, and more with a free Docker ID:                                                 
 https://cloud.docker.com/                                                                                        
                                                                                                                  
For more examples and ideas, visit:                                                                               
 https://docs.docker.com/engine/userguide/                                                                        
```

事情處理到這邊，應該是差不多了。結果到網路上亂逛，居然發覺 Chocolatey 好像有懶人包：

```bash
$ choco install -y docker
$ choco install -y docker-machine
$ choco install -y docker-machine-vmwareworkstation
```

環境設定那五步，似乎都不必做了。
