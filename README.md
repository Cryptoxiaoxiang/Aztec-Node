# Aztec-节点搭建


Linux OS
```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
- 测试 Docker
```bash
sudo docker run hello-world
```
```bash
sudo systemctl enable docker
sudo systemctl restart docker
```
```bash
sudo apt update
sudo apt install -y curl git socat docker.io docker-compose
sudo systemctl enable --now docker
```

### 2 : 安装 Aztec Cli

```bash
bash -i <(curl -s https://install.aztec.network)
```

Linux OS

```bash
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```
测试 cli 

```bash
aztec --version
```
```bash
aztec-up alpha-testnet
```

应该会输出类似于 0.8.5 

###  3 : 获取IP并打开端口

```bash
curl ifconfig.me
```

记录下IP地址

打开必要端口
```bash
# Firewall
ufw allow 22
ufw allow ssh
ufw enable
```
```bash
# Sequencer
ufw allow 40400
ufw allow 8080
```

### 4 : 创建 env 环境文件

这里要用到之前获取的 Sepolia RPC 和 Sepolia BEACON RPC 

```bash
nano ~/.aztec-sequencer.env
```
将ETHEREUM_HOSTS=之后替换成你的sepolia RPC
将L1_CONSENSUS_HOST_URLS=之后替换成你的 sepolia Beacon RPC

这里有一个技巧，如果你获得了多个RPC,可以把他们都加上互为备份，这样一个不行会自动切换为另一个，还可以使用多个平台的免费额度
就是把“ETHEREUM_HOSTS=rpc.ankr.com/eth_sepolia” 替换成“export ETHEREUM_HOSTS=rpc_1,rpc_2,rpc_3,rpc_4,rpc_5”


```bash
# Your Infura Execution RPC (sepolia)
ETHEREUM_HOSTS=rpc.ankr.com/eth_sepolia

# Reliable Public Beacon Chain (Consensus) Endpoint
L1_CONSENSUS_HOST_URLS=lodestar-sepolia.chainsafe.io

# Replace with your values
P2P_IP=your.public.ip.address
SEQUENCER_VALIDATOR_PRIVATE_KEY=YourPrivateKey
SEQUENCER_COINBASE=0xYourPublicWalletAddress
```
保存文件


### 5 : 启动节点

打开一个新窗口放在后台
```bash
screen -S Aztec
```
读取环境文件

```bash
source ~/.aztec-sequencer.env
```
```bash
aztec start --node --archiver --sequencer \
  --network alpha-testnet \
  --l1-rpc-urls "$ETHEREUM_HOSTS" \
  --l1-consensus-host-urls "$L1_CONSENSUS_HOST_URLS" \
  --sequencer.validatorPrivateKey "$SEQUENCER_VALIDATOR_PRIVATE_KEY" \
  --sequencer.coinbase "$SEQUENCER_COINBASE" \
  --p2p.p2pIp "$P2P_IP" \
  --p2p.maxTxPoolSize 1000000000 \
  --sequencer.governanceProposerPayload 0x54F7fe24E349993b363A5Fa1bccdAe2589D5E5Ef
```
<img width="907" alt="Screenshot 2025-06-08 at 15 44 37" src="https://github.com/user-attachments/assets/081d0845-0862-46f4-abc0-e68d88f986ef" />
显示区块号就说明成功了

如果要更新节点：
```bash
aztec-up latest
```


