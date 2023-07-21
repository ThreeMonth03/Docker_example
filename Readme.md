---
title: 透過docker建立jupyter與tensorboard環境
date: 2023-07-21 03:18 +0800
cover: /images/background/docker.png
tags:
- 阿克補習班
categories:
- 知識
---

<escape><!-- more --><escape>

## 前言
在實驗室第二次新生訓練中，學長透過一個小作業，讓我們碩零學習docker。這個小作業是透過docker-compose和local forwarding，來將兩個分別安裝了jupyter和tensorboard的images
啟用服務。

## 作業檔案簡介
可以看到，這個repository裡面有jupyter，tensorboard兩個資料夾，分別有兩個Dockerfile。   
還有docker-compose.yml，直接管理兩個Dockerfile產生的image和container。   
然後main.ipynb和logs是用來demo驗證jupyter和tensorboard功能正常。
 
## 如何執行code
1. vpn去實驗室。目的是為了作業要求連去國網。如果要連其他server，就不用vpn去實驗室。
2. 透過ssh，local forwarding去有docker的server，像這次作業要連去國網。  
參考指令為 ```ssh -L localhost:[your_computer_port]:localhost:[server_port] [Account]@[Server IP]```，而我使用的兩個port分別是```localhost:8323:localhost:8323和localhost:8324:localhost:8324```，因為有兩個port，所以要開兩個terminal連線。
3. 在terminal上打bash，因為國網的server要自己bash。
4. Clone我的source code，在有```docker-compose.yml```的資料夾打上```docker-compose up -d```。
5. 在瀏覽器開啟```http://localhost:8323/```和```http://localhost:8324/```，即可使用jupyter和tensorboard服務。
6. 當用完服務後，在terminal打```docker-compose down```。如果意外動到```docker-compose.yml```，請打 ```docker-compose down --remove-orphans```，刪除沒```docker-compose.yml```認領的孤兒container。


## 筆記
### Local Forwarding
如下圖所示，host可能想用server上的其中一個特定的port，但可能防火牆不會開放直接ssh連線。

<img src="https://johnliu55.tw/ssh-tunnel/images/local_scenario1_problem.png" alt="防火牆" title="防火牆"> 

於是可以利用local forwarding，來繞過防火牆的限制，使用伺服器上的其他port，格式如下:  
```ssh -L localhost:[your_computer_port]:localhost:[server_port] [Account]@[Server IP]```

<img src="https://johnliu55.tw/ssh-tunnel/images/local_scenario1_solved.png" alt="Local Forwarding" title="Local Forwarding"> 

看上圖，假如Account是threemonth，server ip是123.456.78.901，照著上圖的話，local forwarding指令為:  
```ssh -L localhost:9090:localhost:8080 threemonth@123.456.78.901```

### Jupyter & Tensorboard
太習慣用vscode的Jupyter，變得不會下指令開Jupyter和tensorboard了，這裡記錄一下。  

```jupyter notebook --no-browser --ip=0.0.0.0 --port=8080 --allow-root --NotebookApp.token='' --NotebookApp.password='' ```   
這句指令的話   
```--no-browser```就能避免server打不開瀏覽器的情況。   
```--ip=0.0.0.0```的用法能讓任何ip地址使用Jupyter服務。   
```--port```的話就是寫container_port。   
```--allow-root```是解決Running as root is not recommended的錯誤，至於發生原因不清楚。   
然後如果將jupyter開在背景執行，可能就要另外驗證token和密碼，透過```--NotebookApp.token='' --NotebookApp.password='' ```可以省掉驗證。

```tensorboard --logdir ./logs --host=0.0.0.0 --port=8081```   
這句指令的話   
```--logdir```後面就是接log檔的path。   
```--host=0.0.0.0```的用法能讓任何host地址使用tensorboard服務。    
```--port```的話就是寫container_port，   

### Image & Container
#### 簡介
在docker中，有兩個東西分別叫做image和container，他們之間的概念有點像是電腦硬體和作業系統。  
image的話主要處理環境的安裝，container則是依據單一個image，提供對應服務和管理每個服務檔案的地方。而每次服務結束，container就可以隨手刪除，image可以留下來，等下次要使用服務時，再開啟container。

#### 常用Image語法
首先要先寫Dockerfile，或是去dockerhub找別人整理好的Dockerfile，Dockerfile內含有各種指令，包含要安裝的包，或是想自動執行的指令。  

有了Dockerfile後，可以透過下列指令來建立image:  
```docker build -t [image_name] [path]```   
-t這個flag用法是幫image取名，image_name就是image要取的名字，path就是Dockerfile放的資料夾。舉例來說，如果在Dockerfile那層的資料夾中，想創建叫做jupyter_image，那指令就是:  
```docker build -t jupyter_image .```   
如果覺得建立的image有問題，可以使用```docker build -t [image_name] [path] --no-cache```。

建立好image後，透過```docker images```可以查看所有建立的images。

如果要刪除掉image的話，打上```docker image rm [image_name]```就行了。

#### Dockerfile簡介
關於Dockerfile的結構，我決定用我這次作業的其中一個image展示。   
```Dockerfile

FROM pytorch/pytorch:1.13.0-cuda11.6-cudnn8-devel

RUN apt-get update &&\
    apt-get -y upgrade &&\
    apt-get install -y git net-tools vim sudo tcsh gcc g++ unzip python3 python3-pip &&\
    apt-get clean &&\
    rm -rf /var/lib/apt/lists/*

RUN pip3 --no-cache-dir install torch \
    torchvision \
    torchaudio \
    tensorboard \
    jupyterlab \
    jupyter

CMD ["tensorboard", "--logdir", "./logs", "--host=0.0.0.0", "--port=8324"]
```
From的話，就是放基本映像檔。   
RUN就是放要安裝的指令，覺得太長的話，可以用```\```換行。    
CMD就是放container開啟後，默認的command，這個檔案的話就是```"tensorboard --logdir ./logs --host=0.0.0.0 --port=8324 ```。

#### 常用Container語法
建好image後，代表安裝好包，接下來就是啟動服務，所以可以透過下列指令建立並執行container:   
```docker run [opotional -it] [(optional) --name [container_name]]  [(optional) -p [server_port]:[container_port]] [(optional) -v [absolutely/relative path]:[container path]] [(optional) --gpus all] [image_name] [(optional) command]```   
其中optional的部分就是可打可不打，看起來很複雜，就慢慢解釋:
1. -it: -it是由-i -t組成，-i代表讓container的標準輸入保持打開，-t代表讓docker分配一個偽終端(pseudo-tty)並綁定到標準輸入上。簡言之，-it通常搭配指令bash，用來開啟一個terminal操作container。
2. --name: 這個後面加上container的名字，強烈建議要加，不然很難管理container。    
3. -p: 這個是local forwarding會用到的，前面接local forwarding的server_port，後面接container_port，為了好管理，大家常取同樣的數字。
4. -v: 這個用來mount，前面加上想被mount的實際資料夾，後面加上container內mount的目的地。
5. --gpus: 讓docker環境能用上gpu，後面加all就是全用。
6. command: 這邊跟images內的CMD一樣，就是docker run後想自動執行的指令，不過在docker run用了command後，images內的CMD就不會自動執行。然後前面提到用了-it後，可以在command這邊下bash來讓操作更加方便。  

講了那麼多，直接給範例好了:
如果只想執行```jupyter_image```這個image，並且container名字叫做```jupyter_container```，指令為```docker run --name jupyter_container jupyter_image```。    
如果想要進一步用container開啟terminal，然後因為local forwarding，container的port設定成8080:8080，還想使用全部的gpu，還想透過mount來方便的在terminal中使用當前資料夾的資料，可以使用下述指令:   
```ssh
docker run -it --name jupyter_container -p 8080:8080 -v ./:/workspace --gpus all jupyter_image bash
```

開好container後，如果想退出，就直接ctrl+c，如果是terminal，就是輸入exit。如果退出後，想要重新使用container，就是先```docker start [container_name]```，然後再```docker attach [container_name]```，就能重新使用container。

至於如何觀察所有正在使用的container，請在terminal內輸入```docker ps``` ，如果連未使用的container都想觀察，請在terminal內輸入```docker ps -a```。

如果因為更改images，想要重建立container，那就得先刪除當前的container，也就是使用```docker rm [CONTAINER ID]```，CONTAINER ID可以在```docker ps```中看到。
### Docker Compose
因為container一次就是管理一個image，如果想要一次建好多個container，提供多元服務，就需要Docker Compose，一次創立與管理多個image和container。

要使用 Docker Compose，就要寫docker-compose.yml來管理container。寫好docker-compose.yml後，就能下指令，創立與管理多個image和container。

#### 常用的Docker Compose指令
Docker compose相較於container簡單許多，如果要啟用docker-compose.yml，直接打```docker-compose up```，如果怕ssh斷線導致服務中斷，就打```docker-compose up -d```
讓服務跑在背景。

如果要停用docker-compose.yml，直接打```docker-compose down```，如果在啟用docker-compose.yml不小心改到docker-compose.yml，可能變成底下的container無人能管，就會出現orphan container，docker-compose也會報錯，所以得打```docker-compose down --remove-orphans```才能正常停用。

#### docker-compose.yml簡介
關於docker-compose.yml的結構，我決定直接在作業旁邊註記當作講解。   
```yaml

version: "3" #版本，目前有3
services: #建服務，底下就是各種container
  Jupyter: #服務名稱
    build: ./jupyter #如果找不到對應的image，去指定的path找Dockerfile進行docker build
    image: docker/threemonth #images名稱
    container_name: jupyterthreemonth #container名稱
    ports: 
    - "8323:8323" #就是docker run中的 -p
    volumes: #就是docker run中的 -v
    - ./:/workspace 
    restart: unless-stopped #意思是指說除了正常停止外，就會一直restart
    command: jupyter notebook --no-browser --ip=0.0.0.0 --port=8323 --allow-root --NotebookApp.token='' --NotebookApp.password='' #像是Dockerfile中的cmd


  Tensorboard:
    build: ./tensorboard
    image: docker/threemonth2
    container_name: tensorboardthreemonth
    ports: 
    - "8324:8324"
    depends_on: #這意思是等到Jupyter服務開好後再弄Tensorboard。
    - Jupyter
    volumes:
    - ./:/workspace 
    restart: unless-stopped

```

## 心得
關於這次新生訓練，算是我近期新生訓練中，收穫最多的一次。在這次新生訓練前，我從來沒嘗試建個docker，所以在寫這次小作業時，我還滿煎熬的，我認為docker對初學者來說相對不太友善。不過在寫作業的過程中，語法越看越多，也和實驗室同學多多討論後，頓時就豁然開朗，很高興又把酷酷的技能收進自己的技能包中。

## Reference
1. https://yeasy.gitbook.io/docker_practice/
2. https://johnliu55.tw/ssh-tunnel.html
3. https://azole.medium.com/docker-container-%E5%9F%BA%E7%A4%8E%E5%85%A5%E9%96%80%E7%AF%87-2-c14d8f852ae4
