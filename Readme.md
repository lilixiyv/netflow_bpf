# 项目说明

## 代码结构

分为三个子项目：
1. netflow： 基于ebpf的虚拟化网络流量采集程序，包括ebpf探针源码和用户态管理程序，负责在Linux中插入ebpf探针，按照配置文件进行原始流量过滤与采集，并进行字段解析、关联记录与虚拟机或容器节点，最后将处理好的流记录批量化存储于MySQL数据库中。
2. netflow_backend：基于SpringBoot的Web后端，负责从MySQL中查询对应数据提供给前端
3. netflow_frontend：基于Vue的Web前端，并采用Element Plus进行组件设计，使用Echarts进行图表设计。

## 环境配置

### netflow

注意：ebpf技术支持的特性与内核版本有关，若运行时出现问题，请考虑排查内核版本问题，本项目开发使用内核版本为6.8.0-55-generic

#### clang

项目使用clang将ebpf探针源代码（netflow/src/bpf/netflow.bpf.c）编译为对象文件，clang与llvm版本应足够高，本项目采用14版本

#### bpftool

bpftool 用于将eBPF探针源码转换为骨架头文件，从而在用户态程序中可使用libbpf库对其进行生命周期管理并进行数据交互。

从github上获取源码并安装：(项目版本：7.4.0）
```bash
sudo apt update
sudo apt upgrade
sudo apt install -y git build-essential libelf-dev clang llvm
sudo apt install linux-tools-$(uname -r)
git clone --recurse-submodules https://github.com/libbpf/bpftool.git
cd bpftool/src
make
sudo make install
```

验证安装并创建软连接：
```bash
bpftool --version
sudo ln -s /usr/local/sbin/bpftool /usr/sbin/bpftool
bpftool --help
```

#### MySQL

安装
```bash
sudo apt update
sudo apt install mysql-server
```

设置root密码
```bash
sudo mysqladmin -u root password
```

创建项目数据库与用户（数据库与用户名任意，但需与netflow/netflow.confg.json以及netflow/assets/init_table.sql）中的保持一致：
```bash
mysql -u root -p
CREATE DATABASE netflow_db;
CREATE USER 'netflow'@'%' IDENTIFIED BY ’password';
GRANT ALL PRIVILEGES ON netflow_db.* TO 'netflow'@'%';
```

初始化项目数据表（脚本位于netflow/assets/init_table.sql）
```bash
mysql -u netflow -p
source netflow/assets/init_table.sql
```

#### 开发库安装

1. libmysqlclient： 项目通过该库实现数据库操作（项目版本：21.2.42）
2. libglib：提供了一组跨平台的实用函数和数据结构，项目使用该库进行Hash操作（项目版本：2.80.0）
3. libcjson：轻量级C语言JSON解析与生成库，项目使用该库进行配置文件解析(项目版本：1.7.17）
4. libcurl：多功能URL数据传输库，项目使用该库与Docker守护进程通信（项目版本：8.5.0）
5. libxml2：XML处理库，项目使用该库处理虚拟机信息（项目版本：2.9.14）
6. libvirt：虚拟机管理API，项目使用该库与QEMU-KVM通信获取虚拟机信息（项目版本：10.0.0）

```bash
sudo apt update
sudo apt install libmysqlclient-dev libglib2.0-dev libcjson-dev libcurl4-openssl-dev libxml2-dev libvirt-dev libtinfo5
sudo apt install clang libelf1 libelf-dev zlib1g-dev
```

#### 编译运行

项目使用Makefile，在netflow路径下运行：
```bash
make
```
即可完成编译

运行：
```bash
sudo ./netflow
```

### netflow_backend

#### 工具安装

maven（3.8.7）
```bash
sudo apt install maven
```

java（17）安装：
```bash
sudo apt install openjdk-17-jdk
```
配置默认java版本：
```bash
sudo update-alternatives --config java
```
选择与Java 17对应的选项编号

#### 运行

```bash
mvn spring-boot:run
```

### netflow_frontend

#### 工具安装

安装nodejs与npm，项目版本分别为20.19.3与10.8.2
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

#### 运行

安装依赖
```bash
npm install
```

运行：
```bash
npm run dev
```

# 测试环境配置s

测试环境需分别至少包含一个虚拟机与一个容器节点。

其中，虚拟机需使用KVM-QEMU，容器使用Docker

虚拟机与容器内部的系统与版本则无要求。

参考测试环境配置：

| 节点类型 | 配置项         | 配置值                             |
|----------|----------------|-----------------------------------|
| 宿主机   | 操作系统       | Linux Mint 21.3                  |
|          | Linux内核版本 | 6.8.0                            |
|          | QEMU-KVM版本   | 6.2.0                            |
|          | Docker版本     | 28.0.2                           |
|          | MySQL版本      | 8.0.41                           |
|虚拟机     | 网络模式       | NAT                              |
|          | 网段           | 192.168.100.0/24                 |
|          | 操作系统       | Ubuntu 24.04.2                   |
| 容器     | 节点数         | 2                                |
|          | 网络模式       | Bridge                           |
|          | 网段           | 172.17.0.0/16                    |
|          | 操作系统       | Ubuntu 22.04.1                   |

## QEMU-KVM配置

参考https://www.cnblogs.com/beaclnd/p/18047633

### 安装配置virt-manager

#### 安装
```bash
sudo apt update
sudo apt upgrade
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
sudo adduser $USER libvirt
sudo adduser $USER kvm
sudo chown $USER:$USER /var/run/libvirt/libvirt-sock
```

#### 启动
启动Virt-Manager
```bash
virt-manager
```

### 配置虚拟机

1. 下载Ubuntu 24.04 Server, 下载链接https://ubuntu.com/download/servero
2. 在virt-manager中进行安装配置
3. 使用默认配置即可

## docker

### 安装docker

https://docs.docker.com/engine/install/ubuntu/

**注意** 不能使用Docker Desktop，其基于虚拟机实现，无法在宿主机上观察内部容器。

### 将当前用户加入docker组

避免需要root权限运行docker

```bash
sudo usermod -aG docker
```

之后注销再重新登入

### 配置容器

使用任意容器，本例中使用netshoot，其包含若干网络工具，方便测试。

```bash
docker run -it nicolaka/netshoot
```

### 测试流量构建

构造docker容器、虚拟机、主机三者之间的通信流量即可，例如，使用容器ping虚拟机（可能在宿主机中需要添加防火墙规则）
