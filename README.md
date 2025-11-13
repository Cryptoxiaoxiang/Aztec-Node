# Aztec-节点搭建


Linux OS
```bash
#!/usr/bin/env bash
set -e

# 1) 基础更新
sudo apt update -y && sudo apt upgrade -y

# 2) 移除可能的旧组件（不存在也不报错）
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y "$pkg" || true
done

# 3) 依赖与 GPG key
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 4) 正确添加 Docker 官方源（注意 https://、arch、codename）
source /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu ${VERSION_CODENAME} stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5) 安装 Docker Engine + Buildx + Compose
sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6) 启动并开机自启（一般默认已启用，这里再确保一下）
sudo systemctl enable --now docker

# 7) 验证
docker --version
sudo docker run --rm hello-world

# 8) （可选）把当前用户加入 docker 组，免 sudo
# 注意：执行后需要重新登录会话才生效
# sudo usermod -aG docker $USER
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

应该会输出类似于 2.1.2 

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

### 4 : 安装Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
```
```bash
source ~/.bashrc
```
```bash
foundryup
```

### 5 : 使用如下代码生成Validator Keys
```bash
#!/bin/bash

clear
echo "pls provide your old validator info."
read -sp "   put your OLD Sequencer Private Key (will not be shown): " OLD_PRIVATE_KEY && echo
read -p "   put your sepolia rpc only: " ETH_RPC
echo "Goodd. Starting.." && echo " "

rm -f ~/.aztec/keystore/key1.json
echo ":BE READY to write down your private key both eth and BLS and your eth address."
read -p "   Press [Enter] to generate your new keys..."
aztec validator-keys new --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000 && echo " " 
KEYSTORE_FILE=~/.aztec/keystore/key1.json
NEW_ETH_PRIVATE_KEY=$(jq -r '.validators[0].attester.eth' $KEYSTORE_FILE)
NEW_BLS_PRIVATE_KEY=$(jq -r '.validators[0].attester.bls' $KEYSTORE_FILE)
NEW_PUBLIC_ADDRESS=$(cast wallet address $NEW_ETH_PRIVATE_KEY)

echo "good! Your new keys are below. SAVE THIS INFO SECURELY!"
echo "   - NEW ETH Private Key: $NEW_ETH_PRIVATE_KEY"
echo "   - NEW BLS Private Key:  $NEW_BLS_PRIVATE_KEY"
echo "   - NEW Public Address:   $NEW_PUBLIC_ADDRESS"
echo " "

echo "You need to send 0.2 to 0.5 Sepolia eth to this new address:"
echo "   $NEW_PUBLIC_ADDRESS"
read -p "   After the funding tnx is confirmed, press [Enter] to continue.." && echo " "


echo "Approving STAKE spending..."
cast send 0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A "approve(address,uint256)" 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 200000ether --private-key "$OLD_PRIVATE_KEY" --rpc-url "$ETH_RPC" && echo " "

echo "joining the testnet yey..."
aztec add-l1-validator \
  --l1-rpc-urls "$ETH_RPC" \
  --network testnet \
  --private-key "$OLD_PRIVATE_KEY" \
  --attester "$NEW_PUBLIC_ADDRESS" \
  --withdrawer "$NEW_PUBLIC_ADDRESS" \
  --bls-secret-key "$NEW_BLS_PRIVATE_KEY" \
  --rollup 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 && echo " "

echo "All done! u have successfully joined the new testnet now rerun your node with your new pvt and address"
```
这里会先输入测试网用的钱包私钥，然后会生成两个私钥： attester和BLS 记录下来，导入到自己的钱包里， 并且往attester 地址里充值 0.2 sepolia eth

### 6 : 在网络上注册节点
授权20万stk代币
```bash
cast send 0x139d2a7a0881e16332d7D1F8DB383A4507E1Ea7A "approve(address,uint256)" 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67 200000ether --private-key "$PRIVATE_KEY_OF_OLD_SEQUENCER" --rpc-url $ETH_RPC
```
$PRIVATE_KEY_OF_OLD_SEQUENCER 替换成最开始使用的私钥， $ETH_RPC替换成RPC节点的IP地址。 接下来注册：

```bash
aztec \
  add-l1-validator \
  --l1-rpc-urls $ETH_RPC \
  --network testnet \
  --private-key $PRIVATE_KEY_OF_OLD_SEQUENCER \
  --attester $ETH_ATTESTER_ADDRESS \
  --withdrawer $ANY_ETH_ADDRESS \
  --bls-secret-key $BLS_ATTESTER_PRIV_KEY \
  --rollup 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67
```
$ETH_RPC 是RPC节点IP
$ETH_ATTESTER_ADDRESS 就是上面的并且往attester 地址， 
$ANY_ETH_ADDRESS替换成任意钱包地址，用来提款用的，
$BLS_ATTESTER_PRIV_KEY 是上面的BLS私钥

### 7 : 创建 env 环境文件

这里要用到之前获取的 Sepolia RPC 和 Sepolia BEACON RPC 

```bash
nano ~/.aztec-sequencer.env
```
将ETHEREUM_HOSTS=之后替换成你的sepolia RPC ip地址，端口号保持8545
SEQUENCER_VALIDATOR_PRIVATE_KEY 是你要用的钱包的私钥
SEQUENCER_COINBASE 是钱包的地址


这里有一个技巧，如果你获得了多个RPC,可以把他们都加上互为备份，这样一个不行会自动切换为另一个，还可以使用多个平台的免费额度
就是把“ETHEREUM_HOSTS=rpc.ankr.com/eth_sepolia” 替换成“export ETHEREUM_HOSTS=rpc_1,rpc_2,rpc_3,rpc_4,rpc_5”


```bash
# Your Infura Execution RPC (sepolia)
ETHEREUM_HOSTS=http://111.111.111.111:8545 

# Reliable Public Beacon Chain (Consensus) Endpoint
L1_CONSENSUS_HOST_URLS=https://lodestar-sepolia.chainsafe.io/

# Replace with your values
P2P_IP=your.public.ip.address
SEQUENCER_VALIDATOR_PRIVATE_KEY=YourPrivateKey
SEQUENCER_COINBASE=0xYourPublicWalletAddress
```
保存文件


### 8 : 启动节点

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
  --sequencer.validatorPrivateKeys "$SEQUENCER_VALIDATOR_PRIVATE_KEY" \
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


