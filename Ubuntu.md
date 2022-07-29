# Ubuntu

## WSL

### 安装PPA

[wslu (In development) : Patrick Wu (launchpad.net)](https://launchpad.net/~callmepk/+archive/ubuntu/wslu-dev)

```bash
sudo add-apt-repository ppa:callmepk/wslu-dev
sudo apt update
```

### 升级到WSL2



### 重启

```powershell
Get-Service LxssManager | Restart-Service
```

## 安装vimplus

[chxuan/vimplus: An automatic configuration program for vim (github.com)](https://github.com/chxuan/vimplus)

```bash
git clone https://github.com/chxuan/vimplus.git ~/.vimplus
cd ~/.vimplus
./install.sh //不加sudo
```

可能需要手工编译ycm

```bash
cd ~/.vim/plugged/YouCompleteMe 
./install.py
```

## 安装MySQL

[Add or connect a database with WSL | Microsoft Docs](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-database)

```bash
sudo apt install mysql-server mysql-client
sudo apt install libmysqlclient-dev
sudo /etc/init.d/mysql start
sudo mysql_secure_installation
sudo mysql
```

如果出现sock被占用或者权限不足问题，安装PPA

## 换源

[ubuntu | 镜像站使用帮助 | 清华大学开源软件镜像站 | Tsinghua Open Source Mirror](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)

## 安装开发环境

```bash
sudo apt install build-essential
```

