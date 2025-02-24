# ZedBoard 影像辨識
## 簡述
透過 xilinx 提供的 Deep Learning Processing Unit (DPU)，在Zedboard上進行影像辨識之功能。

## 環境
必須使用以下版本
Ubuntu18.04
vivado 2019.2
[petalinux 2019.2](https://www.xilinx.com/member/forms/download/xef.html?filename=petalinux-v2019.2-final-installer.run)
[DNNDK v3.1](https://drive.google.com/file/d/1cypbcTahJ5WTM4Z7i9wDFMsd9gVPW_8x/view?usp=drive_link)
[DPU檔案](https://drive.google.com/file/d/1fTqYYhExuSNNzC64WFvk1SkOgriDtbft/view?usp=drive_link)

## 輸出硬體
解壓縮下載檔案後開啟 vivado 2019.2，並在 TCL console 輸入
```shell=
cd project_path/Resnet50_ZedBoard/pl
source ./scripts/Resnet50_ZedBoard.tcl 
```
![image](https://hackmd.io/_uploads/rJ5xPTVqke.png)
點選Generate bitstream，需等待一段時間完成

![image](https://hackmd.io/_uploads/H1r3lfPqJe.png)
輸出.xsa檔且必須包含bitstream
## 安裝 Peatalinux 2019.2

### 1. 下載所需依賴項
```shell=
sudo apt install -y tofrodos iproute2 gawk xvfb gcc g++ net-tools \
  libncurses5-dev tftpd zlib1g-dev zlib1g:i386 libssl-dev flex bison \ 
  libselinux1 wget curlsocat xterm python python3 screen pax diffstat \
  chrpath autoconf libtool texinfo unzip zlib1g-dev gcc-multilib
```

petalinux2019.2 中的 yocto SDK 需要 python2，但 ubuntu18.04 並未有 python2 的版本，因此需要另外安裝。




```shell=
sudo apt install -y curl
curl -O https://bootstrap.pypa.io/pip/2.7/get-pip.py
sudo apt install -y gcc make zlib1g-dev
wget https://www.python.org/ftp/python/2.7.18/Python-2.7.18.tgz
tar -xvzf Python-2.7.18.tgz
cd Python-2.7.18
./configure
make
sudo make install
```


### 2. 安裝 petalinux 2019.2 SDK


```shell=
chmod +x ~/YOUR_FILE_PATH/petalinux-v2019.2-final-installer.run
mkdir ~/petalinux/2019.2
cd ~/petalinux/2019.2
~/YOUR_FILE_PATH/petalinux-v2019.2-final-installer.run ~/petalinux/2019.2
```

1. 執行後可能會遇到 petalinux_installation_log 相關問題，但通常是執行權限問題，輸入chmod 777 petalinux_installation_log 可解決。
2. 安裝過程中需要同意一些，根據terminel上的指示操作即可。


安裝完成後將其加入環境變數，之後每次開啟Terminal不需額外輸入指令即可使用 petalinux commands
```shell=
sudo nano ~/.bashrc
#貼上下方指令
source ~/petalinux/2019.2/settings.sh
```

## 建立 Petalinux 專案

### Create the PetaLinux project
```shell=
cd Resnet50_ZedBoard_2019_2/apu/resnet50_zedboard_bsp/
petalinux-create -t project -s resnet50_zedboard.bsp -n YOUR_PROJECT_NAME --force
```

### Select kernel module for USB Cam
```shell=
petalinux-config -c kernel
```
進入Device Drivers

![image](https://hackmd.io/_uploads/SJdy9dDqkg.png)

進入USB support

![image](https://hackmd.io/_uploads/ryds5OD5ye.png)

進入Support for Host-side USB，啟用OTG support
![image](https://hackmd.io/_uploads/r1ik6uv9Jx.png)

回到 Deivce Drivers，進入Multimedia support
![image](https://hackmd.io/_uploads/HJZzCuD5kg.png)

進入 Media USB Adapters
![image](https://hackmd.io/_uploads/HJryRuw5kx.png)

啟用 UVC input event device support
![image](https://hackmd.io/_uploads/r1kqAuwqkx.png)

完成後儲存命名檔案

### Configure the PetaLinux project with the HW design 
```shell=
cd Resnet50_ZedBoard_2019_2/apu/resnet50_zedboard_bsp/YOUR_PROJECT_NAME/
petalinux-config --get-hw-description=~/DIRETORY_PATH/Resnet50_ZedBoard_2019_2/pl/prj/zedboard --silentconfig
```

### Build the PetaLinux project 
```shell=
petalinux-build
```

### Create BOOT.BIN and image.ub for the SD card
```shell=
cd Resnet50_ZedBoard_2019_2/apu/resnet50_zedboard_bsp/YOUR_PROJECT_NAME/images/linux
petalinux-package --boot --fsbl zynq_fsbl.elf --u-boot u-boot.elf --fpga system.bit --force
```

最後將 Resnet50_ZedBoard_2019_2/apu/resnet50_zedboard_bsp/YOUR_PROJECT_NAME/images/linux 中的 BOOT.BIN & image.ub 存入 SDcard


## 編譯 Resnet-50 應用

### Generate a new SDK
```shell=
cd Resnet50_ZedBoard_2019_2/apu/resnet50_zedboard_bsp/YOUR_PROJECT_NAME/
petalinux-build -s 
cp Resnet50_ZedBoard_2019_2/apu/resnet50_zedboard_bsp/YOUR_PROJECT_NAME/images/linux/sdk.sh $TRD_HOME/apu/apps/
```

### Extract the SDK
```shell=
cd Resnet50_ZedBoard_2019_2/apu/apps 
chmod 777 sdk.sh 
./sdk.sh -d ./sdk -y
```


### Download xilinx_dnndk_v3.1 and copy the resnet50 folder
[DNNDK v3.1檔案](https://drive.google.com/file/d/1cypbcTahJ5WTM4Z7i9wDFMsd9gVPW_8x/view?usp=drive_link)
```shell=
cp -r xilinx_dnndk_v3.1/ZedBoard/samples/resnet50 Resnet50_ZedBoard_2019_2/apu/apps
```

### Modify Sample Code 
將範例程式修改成可使用相機辨識的功能
```shell=
#修改此檔案
Resnet50_ZedBoard_2019_2/apu/apps/resnet50/src/main.cc
```


### Compile Resnet50 application
```shell=
unset LD_LIBRARY_PATH 
. Resnet50_ZedBoard_2019_2/apu/apps/sdk/environment-setup-cortexa9t2hf-neon-xilinx-linux-gnueabi 
cd Resnet50_ZedBoard_2019_2/apu/apps/resnet50 
make
```

若遇到如下圖問題，通常是沒有選到正確的編譯器。
![image](https://hackmd.io/_uploads/r1m_zu_cJx.png)

 1、 修改 makefile，並將預設g++編譯器刪除。
```shell=
YOUR_PATH/Resnet50_ZedBoard_2019_2/apu/apps/resnet50/makefile
```
![image](https://hackmd.io/_uploads/rJSpGOO91l.png)


2、在 ~/.bashrc 加入下方指令，確保編譯時使用的是針對 arm 平台的編譯器
```shell=
source Resnet50_Zedoard_2019_2/apu/apps/sdk/environment-setup-cortexa9t2hf-neon-xilinx-linux-gnueabi
```

完成後可以看到編譯器已經不是原本的 x86 使用的 g++。
![image](https://hackmd.io/_uploads/S13j7uu5ke.png)




最後將 Resnet50_ZedBoard_2019_2/apu/apps/resnet50 資料夾存入 SDcard

## DEMO


#### UART Setting 

![image](https://hackmd.io/_uploads/HJdSl2_5Jx.png)

#### LOGIN
user : root
password : root

![image](https://hackmd.io/_uploads/S1pPSndcJx.png)

#### Run application
```shell=
cd /run/media/mmcblk0pl/resnet50
./resnet50
```
![image](https://hackmd.io/_uploads/SyX18nucJe.png)

[DEMO 影片](https://www.youtube.com/watch?v=IpimcaEAdA0)

#### Problem

1. 辨識精確度過低
2. 執行過程有時會死當
