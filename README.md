# Boundless Prover 节点挖矿保姆级教程

## 📋 目录
- [概述](#概述)
- [环境准备](#环境准备)
- [依赖安装](#依赖安装)
- [硬件配置](#硬件配置)
- [网络配置](#网络配置)
- [质押和挖矿](#质押和挖矿)
- [监控和维护](#监控和维护)

## 概述

本教程将指导您如何在 Vast.ai 云服务器上部署 Boundless Prover 节点进行挖矿。

### 🚨 重要提醒
- **推荐环境**：Vast.ai 云服务器或本地 Ubuntu 22.04 虚拟机
- **不推荐**：Windows + WSL2（GPU 兼容性差、易崩溃、难调试）
- **最低要求**：4GB 内存、3 个 CPU 核心、NVIDIA GPU

### 📊 预期收益
- 参与 Boundless 网络验证
- 获得网络奖励
- 支持去中心化计算

## 环境准备

### 1. 购买 Vast.ai 机器
- **参考教程**：https://x.com/YOYOMYOYOA/status/1708092318614708242
- **Vast.ai 链接**：https://cloud.vast.ai/?ref_id=87823
- **TG 交流群**：https://t.me/YOYOZKS

### 2. 系统环境要求
- ✅ Ubuntu 22.04 LTS
- ✅ NVIDIA GPU（推荐 RTX 4000 系列或更高）
- ✅ 至少 4GB 内存
- ✅ 至少 3 个 CPU 核心
- ✅ 稳定的网络连接

## 依赖安装

### 步骤 1：更新系统并安装基础软件包
```bash
# 更新系统
apt update && apt upgrade -y

# 安装必要的软件包
apt install curl iptables build-essential git wget lz4 jq make gcc nano \
automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev tar \
clang bsdmainutils ncdu unzip libleveldb-dev libclang-dev ninja-build -y
```

### 步骤 2：克隆 Boundless 仓库
```bash
# 克隆仓库
git clone https://github.com/boundless-xyz/boundless

# 进入目录
cd boundless

# 切换到稳定版本
git checkout release-0.10

# 验证版本
git log --oneline -1
```

### 步骤 3：快速设置 Boundless 依赖项
```bash
# 运行设置脚本
bash ./scripts/setup.sh
```

**⚠️ 重要提示**：
- 此步骤会重新启动机器安装 NVIDIA 驱动
- 请耐心等待，不要中断进程
- 重启后重新连接到服务器并继续后续步骤

### 步骤 4：安装 Rust
```bash
# 安装 Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 加载环境变量
. "$HOME/.cargo/env"

# 更新 Rust
rustup update

# 验证安装
rustc --version
cargo --version
```

### 步骤 5：安装 rzup
```bash
# 安装 rzup
curl -L https://risczero.com/install | bash

# 重新加载环境
source ~/.bashrc

# 验证安装
rzup --version
```

### 步骤 6：安装 Rust 工具链
```bash
# 安装工具链
rzup install

# 验证安装
rzup show
```

### 步骤 7：安装 RISC Zero CLI 工具
```bash
# 安装 bento-client
cargo install --git https://github.com/risc0/risc0 bento-client --bin bento_cli

# 添加到 PATH
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc

# 重新加载环境
source ~/.bashrc

# 验证安装
bento_cli --version
```

### 步骤 8：安装 Boundless CLI
```bash
# 安装 Boundless CLI
cargo install --locked boundless-cli

# 设置 PATH
export PATH=$PATH:/root/.cargo/bin

# 重新加载环境
source ~/.bashrc

# 验证安装
boundless --version
```

### 步骤 9：安装 just
```bash
# 安装 just
curl -fsSL https://just.systems/install.sh | bash -s -- --to /usr/local/bin

# 验证安装
just --version
```

**✅ 验证所有安装**：
```bash
# 检查所有工具是否正确安装
echo "=== 安装验证 ==="
rustc --version
cargo --version
rzup --version
bento_cli --version
boundless --version
just --version
nvidia-smi
```

## 硬件配置

### 1. 查看 GPU 信息
```bash
# 查看所有 GPU 的 ID
nvidia-smi -L

# 查看当前 GPU 实时使用情况
nvidia-smi

# 查看 GPU 详细信息
nvidia-smi -q
```

### 2. 配置 compose.yml

`compose.yml` 是 Boundless Prover 的核心配置文件，定义了所有挖矿服务的运行参数。

#### 编辑配置文件
```bash
# 编辑配置文件
nano compose.yml
```

#### 服务配置详解

**x-exec-agent-common 服务**
- **作用**：执行订单的预检过程，评估 Prover 是否可以对订单出价
- **优化建议**：增加此服务的资源可以提高并发证明数量

**gpu_prove_agent 服务**
- **作用**：使用 GPU 处理已锁定的订单证明
- **优化建议**：增加 CPU 和内存可以提高单个 GPU 的性能

#### 配置示例（根据您的机器调整）

**基础配置**：
```yaml
x-exec-agent-common: &exec-agent-common
  <<: *agent-common
  mem_limit: 4G  # 根据您的机器内存调整（建议：总内存的 80%）
  cpus: 3        # 根据您的机器 CPU 核心数调整（建议：总核心数的 75%）
  environment:
    <<: *base-environment
    RISC0_KECCAK_PO2: ${RISC0_KECCAK_PO2:-17}
  entrypoint: /app/agent -t exec --segment-po2 ${SEGMENT_SIZE:-21}

gpu_prove_agent0:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G  # 每个 GPU 代理的内存
  cpus: 4        # 每个 GPU 代理的 CPU 核心
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['0']    # GPU 设备 ID
            capabilities: [gpu]
```

#### 不同硬件配置推荐

**4GB 内存，4 核 CPU 配置**：
```yaml
x-exec-agent-common: &exec-agent-common
  mem_limit: 3G    # 4GB * 0.8 = 3.2GB，取整为 3G
  cpus: 3          # 4核 * 0.75 = 3核

gpu_prove_agent0:
  mem_limit: 3G
  cpus: 3
```

**8GB 内存，8 核 CPU 配置**：
```yaml
x-exec-agent-common: &exec-agent-common
  mem_limit: 6G    # 8GB * 0.8 = 6.4GB，取整为 6G
  cpus: 6          # 8核 * 0.75 = 6核

gpu_prove_agent0:
  mem_limit: 6G
  cpus: 6
```

**16GB 内存，16 核 CPU 配置**：
```yaml
x-exec-agent-common: &exec-agent-common
  mem_limit: 12G   # 16GB * 0.8 = 12.8GB，取整为 12G
  cpus: 12         # 16核 * 0.75 = 12核

gpu_prove_agent0:
  mem_limit: 12G
  cpus: 12
```

#### 多 GPU 配置

如果您有多个 GPU，可以为每个 GPU 创建独立的代理：

```yaml
# GPU 0 代理
gpu_prove_agent0:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G
  cpus: 4
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['0']
            capabilities: [gpu]

# GPU 1 代理
gpu_prove_agent1:
  <<: *agent-common
  runtime: nvidia
  mem_limit: 4G
  cpus: 4
  entrypoint: /app/agent -t prove
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['1']
            capabilities: [gpu]
```

#### 配置优化原则

| 资源类型 | 推荐分配比例 | 说明 |
|----------|-------------|------|
| 内存 | 总内存的 80% | 留 20% 给系统和其他进程 |
| CPU | 总核心数的 75% | 留 25% 给系统和其他进程 |
| GPU | 100% | 全部用于挖矿 |

**⚠️ 重要提示**：
- 逐步调整资源，不要一次性大幅增加
- 确保系统有足够资源运行其他进程
- 高负载时注意 GPU 温度
- 修改前备份原始配置文件
- 每次修改后测试服务是否正常运行

## 网络配置

### 1. 选择网络
Boundless 提供三个网络选择：

| 网络 | 用途 | 推荐度 |
|------|------|--------|
| Base Mainnet | 主网 | ⭐⭐⭐⭐⭐ |
| Base Sepolia | 测试网 | ⭐⭐⭐⭐ |
| Ethereum Sepolia | 测试网 | ⭐⭐⭐ |

### 2. 配置环境变量
```bash
# 进入文件内部
nano .env.eth-sepolia

# 在文件中添加
export PRIVATE_KEY="你的私钥"
export RPC_URL="你的RPC节点"

# 让文件生效
source .env.eth-sepolia
```

### 3. 安全提醒
- 🔒 **私钥安全**：永远不要分享您的私钥
- 🔒 **备份重要**：备份您的私钥和配置文件
- 🔒 **权限设置**：确保配置文件权限正确

## 质押和挖矿

### 1. 获取测试币
- **测试币领取地址**：https://faucet.circle.com
- **领取步骤**：
  1. 访问网站
  2. 连接钱包
  3. 选择网络（Sepolia）
  4. 点击领取

### 2. 质押 USDC
```bash
# 检查账户余额
boundless account balance

# 质押 10 USDC
boundless account deposit-stake 10

# 验证质押状态
boundless account stake-info
```

**⚠️ 重要**：只有质押后运行节点时才能接到任务。

### 3. 启动挖矿

#### 运行 Bento
```bash
# 启动bento
just bento

# 检查日志
just bento logs
```

#### 运行 Broker
```bash
# 启动broker
just broker

# 检查日志
just broker logs
```

## 监控和维护

### 1. 日常监控命令
```bash
# 查看系统资源
htop

# 查看 GPU 使用情况
watch -n 1 nvidia-smi

# 查看磁盘使用
df -h

# 查看网络连接
netstat -tulpn
```

### 2. 日志监控
```bash
# 实时查看 Bento 日志
tail -f logs/bento.log

# 实时查看 Broker 日志
tail -f logs/broker.log

# 查看错误日志
grep -i error logs/*.log
```

### 3. 性能优化
```bash
# 优化系统性能
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p

# 设置进程优先级
renice -n -10 -p $(pgrep -f "just bento")
renice -n -10 -p $(pgrep -f "just broker")
```

### 4. 备份和恢复
```bash
# 备份重要文件
tar -czf boundless-backup-$(date +%Y%m%d).tar.gz .env* compose.yml

# 备份私钥（安全位置）
cp .env.eth-sepolia ~/backup/
```

## 完成

🎉 **恭喜！** 您已经成功部署了 Boundless Prover 节点，可以在 7 月 8 日开始挖矿了！

### 下一步
1. 监控节点运行状态
2. 关注官方更新和公告
3. 参与社区讨论
4. 优化配置以获得最佳性能

## 联系方式

- **TG 交流群**：https://t.me/YOYOZKS
- **官方文档**：https://docs.boundless.xyz
- **GitHub**：https://github.com/boundless-xyz/boundless

## 注意事项

1. ✅ 确保所有依赖正确安装
2. ✅ 根据实际硬件配置调整参数
3. ✅ 保持网络连接稳定
4. ✅ 定期检查节点运行状态
5. ✅ 关注官方更新和公告
6. ✅ 妥善保管私钥和配置文件
7. ✅ 定期备份重要数据
8. ✅ 监控系统资源使用情况

---

**最后更新**：2024年12月
**版本**：v1.0 