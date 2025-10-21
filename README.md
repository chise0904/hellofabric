# hellofabric


## 下載 Fabric 工具和映像檔

```
curl -sSL https://bit.ly/2ysbOFE | bash -s -- 2.5.0 1.5.5
```


## 創建專案目錄

```
# 創建專案根目錄
mkdir -p ~/fabric-financial-network
cd ~/fabric-financial-network

# 創建子目錄結構
mkdir -p organizations/banka
mkdir -p organizations/bankb
mkdir -p organizations/insurance
mkdir -p organizations/ordererOrg
mkdir -p channel-artifacts
mkdir -p chaincode
```


## 設定環境變量

```
# 將 Fabric 工具加入 PATH
export PATH=${HOME}/fabric-samples/bin:$PATH

# 設定配置文件路徑
export FABRIC_CFG_PATH=${PWD}

# 驗證工具可用
cryptogen version
configtxgen version

# 預期輸出：
# cryptogen:
#  Version: 2.5.x
# configtxgen:
#  Version: 2.5.x

# 將這些環境變數加入 ~/.bashrc 以便永久生效
echo "export PATH=\${HOME}/fabric-samples/bin:\$PATH" >> ~/.bashrc
echo "export FABRIC_CFG_PATH=\${PWD}" >> ~/.bashrc

```



## 使用 cryptogen 生成所有證書

```
cryptogen generate --config=./crypto-config.yaml --output="organizations"
```


## 驗證證書生成

```
# 檢查 Bank A 的 CA 證書
ls -la organizations/banka/ca/
# 預期看到：
# ca.banka.financial.com-cert.pem
# priv_sk (私鑰)

# 檢查 Peer 證書
ls -la organizations/banka/peers/peer0.banka.financial.com/msp/signcerts/
# 預期看到：
# peer0.banka.financial.com-cert.pem

# 檢查用戶證書
ls organizations/banka/users/
# 預期看到：
# Admin@banka.financial.com/
# User1@banka.financial.com/
# User2@banka.financial.com/

# 驗證證書內容（可選）
openssl x509 -in organizations/banka/ca/ca.banka.financial.com-cert.pem -text -noout | head -20

```


## 創建 channel-artifacts 目錄


```
# 確保目錄存在
mkdir -p channel-artifacts

# 設定輸出路徑
export CHANNEL_ARTIFACTS=./channel-artifacts

```

## 生成創世區塊


```
# 生成 Orderer 創世區塊
configtxgen -profile ThreeOrgsOrdererGenesis \
    -channelID system-channel \
    -outputBlock ${CHANNEL_ARTIFACTS}/genesis.block

# 預期輸出：
# 2024-10-13 12:00:00.000 UTC 0001 INFO [common.tools.configtxgen] main -> Loading configuration
# 2024-10-13 12:00:00.000 UTC 0002 INFO [common.tools.configtxgen.localconfig] completeInitialization -> orderer type: etcdraft
# 2024-10-13 12:00:00.000 UTC 0003 INFO [common.tools.configtxgen.localconfig] Load -> Loaded configuration: configtx.yaml
# 2024-10-13 12:00:00.000 UTC 0004 INFO [common.tools.configtxgen] doOutputBlock -> Generating genesis block
# 2024-10-13 12:00:00.000 UTC 0005 INFO [common.tools.configtxgen] doOutputBlock -> Writing genesis block

# 驗證創世區塊已生成
ls -lh ${CHANNEL_ARTIFACTS}/genesis.block
# 預期：文件存在，大小約 15-20KB

```


## 生成 banking-channel 配置

```
export CHANNEL_ARTIFACTS=./channel-artifacts

# 生成 banking-channel 的創建交易
configtxgen -profile BankingChannel \
    -outputCreateChannelTx ${CHANNEL_ARTIFACTS}/banking-channel.tx \
    -channelID banking-channel

# 預期輸出：
# 2024-10-13 12:01:00.000 UTC 0001 INFO [common.tools.configtxgen] main -> Loading configuration
# 2024-10-13 12:01:00.000 UTC 0002 INFO [common.tools.configtxgen] doOutputChannelCreateTx -> Generating new channel configtx
# 2024-10-13 12:01:00.000 UTC 0003 INFO [common.tools.configtxgen] doOutputChannelCreateTx -> Writing new channel tx

# 驗證文件
ls -lh ${CHANNEL_ARTIFACTS}/banking-channel.tx

```


## 生成 insurance-channel 配置

```
export CHANNEL_ARTIFACTS=./channel-artifacts

# 生成 insurance-channel 的創建交易
configtxgen -profile InsuranceChannel \
    -outputCreateChannelTx ${CHANNEL_ARTIFACTS}/insurance-channel.tx \
    -channelID insurance-channel

# 驗證
ls -lh ${CHANNEL_ARTIFACTS}/insurance-channel.tx

```



## 生成錨節點配置

```
export CHANNEL_ARTIFACTS=./channel-artifacts

# Bank A 在 banking-channel 的錨節點配置
configtxgen -profile BankingChannel \
    -outputAnchorPeersUpdate ${CHANNEL_ARTIFACTS}/BankAMSPanchors.tx \
    -channelID banking-channel \
    -asOrg BankAMSP

# Bank B 在 banking-channel 的錨節點配置
configtxgen -profile BankingChannel \
    -outputAnchorPeersUpdate ${CHANNEL_ARTIFACTS}/BankBMSPanchors.tx \
    -channelID banking-channel \
    -asOrg BankBMSP

# Bank B 在 insurance-channel 的錨節點配置
configtxgen -profile InsuranceChannel \
    -outputAnchorPeersUpdate ${CHANNEL_ARTIFACTS}/BankBMSPanchors-insurance.tx \
    -channelID insurance-channel \
    -asOrg BankBMSP

# Insurance Co 在 insurance-channel 的錨節點配置
configtxgen -profile InsuranceChannel \
    -outputAnchorPeersUpdate ${CHANNEL_ARTIFACTS}/InsuranceCoMSPanchors.tx \
    -channelID insurance-channel \
    -asOrg InsuranceCoMSP

# 驗證所有配置文件已生成
ls -lh ${CHANNEL_ARTIFACTS}/
# 預期看到：
# genesis.block
# banking-channel.tx
# insurance-channel.tx
# BankAMSPanchors.tx
# BankBMSPanchors.tx
# BankBMSPanchors-insurance.tx
# InsuranceCoMSPanchors.tx

```


## 測試步驟

### 1. 檢查所有容器是否正常運行

```bash
# 查看容器狀態
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 應該看到 10 個容器都在運行：
# - 3 個 orderer
# - 6 個 peer (每個組織 2 個)
# - 1 個 CLI
```

### 2. 檢查 Peer 日誌確認沒有錯誤

```bash
# 檢查各個 peer 是否正常啟動
docker logs peer0.banka.financial.com 2>&1 | tail -20
docker logs peer0.bankb.financial.com 2>&1 | tail -20
docker logs peer0.insurance.financial.com 2>&1 | tail -20

# 應該看到類似：
# "Starting peer with ID=[peer0.banka.financial.com]"
# "Started peer with ID=[peer0.banka.financial.com]"
```

### 3. 檢查 Orderer 日誌

```bash
docker logs orderer1.orderer.financial.com 2>&1 | tail -20

# 應該看到：
# "Beginning to serve requests"
# "Starting without a system channel"
```

### 4. 進入 CLI 容器測試

```bash
# 進入 CLI 容器
docker exec -it cli bash

# 你應該會看到類似的提示符：
# root@xxxxx:/opt/gopath/src/github.com/hyperledger/fabric/peer#
```

### 5. 測試 Peer 連線（在 CLI 容器內）

```bash
# 測試連接 peer0.banka
peer channel list

# 應該輸出（目前還沒加入任何通道，所以是空的）：
# Channels peers has joined: 
```

### 6. 創建通道（Banking Channel）

```bash
# 在 CLI 容器內執行

# 設定 Orderer CA 證書路徑
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/ordererOrganizations/orderer.financial.com/orderers/orderer1.orderer.financial.com/msp/tlscacerts/tlsca.orderer.financial.com-cert.pem

# 創建 banking-channel
peer channel create \
  -o orderer1.orderer.financial.com:7050 \
  -c banking-channel \
  -f ./channel-artifacts/banking-channel.tx \
  --outputBlock ./channel-artifacts/banking-channel.block \
  --tls \
  --cafile $ORDERER_CA

# 如果成功，會看到：
# "Received block: 0"
# 並且會生成 banking-channel.block 文件
```

### 7. 加入通道 - BankA Peer0

```bash
# 仍在 CLI 容器內

# BankA Peer0 加入通道
peer channel join -b ./channel-artifacts/banking-channel.block

# 成功輸出：
# "Successfully joined peer to the channel 'banking-channel'"
```

### 8. 切換到 BankA Peer1 並加入通道

```bash
# 切換環境變數到 peer1.banka
export CORE_PEER_ADDRESS=peer1.banka.financial.com:8051
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/peers/peer1.banka.financial.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/peers/peer1.banka.financial.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/peers/peer1.banka.financial.com/tls/ca.crt

# 加入通道
peer channel join -b ./channel-artifacts/banking-channel.block
```

### 9. 切換到 BankB 並加入通道

```bash
# 切換到 BankB Peer0
export CORE_PEER_LOCALMSPID="BankBMSP"
export CORE_PEER_ADDRESS=peer0.bankb.financial.com:9051
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer0.bankb.financial.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer0.bankb.financial.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer0.bankb.financial.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/users/Admin@bankb.financial.com/msp

# 加入通道
peer channel join -b ./channel-artifacts/banking-channel.block
```

### 10. BankB Peer1 加入通道

```bash
# 切換到 BankB Peer1
export CORE_PEER_ADDRESS=peer1.bankb.financial.com:10051
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer1.bankb.financial.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer1.bankb.financial.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer1.bankb.financial.com/tls/ca.crt

# 加入通道
peer channel join -b ./channel-artifacts/banking-channel.block
```

### 11. 驗證通道加入成功

```bash
# 列出已加入的通道
peer channel list

# 應該輸出：
# Channels peers has joined:
# banking-channel

# 查看通道詳細資訊
peer channel getinfo -c banking-channel

# 應該輸出類似：
# Blockchain info: {"height":1,"currentBlockHash":"...","previousBlockHash":"..."}
```

### 12. 更新錨節點（Anchor Peers）

```bash
# 切換回 BankA Admin
export CORE_PEER_LOCALMSPID="BankAMSP"
export CORE_PEER_ADDRESS=peer0.banka.financial.com:7051
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/users/Admin@banka.financial.com/msp
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/peers/peer0.banka.financial.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/peers/peer0.banka.financial.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/banka.financial.com/peers/peer0.banka.financial.com/tls/ca.crt

# 更新 BankA 錨節點
peer channel update \
  -o orderer1.orderer.financial.com:7050 \
  -c banking-channel \
  -f ./channel-artifacts/BankAMSPanchors.tx \
  --tls \
  --cafile $ORDERER_CA

# 切換到 BankB 並更新錨節點
export CORE_PEER_LOCALMSPID="BankBMSP"
export CORE_PEER_ADDRESS=peer0.bankb.financial.com:9051
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/users/Admin@bankb.financial.com/msp
export CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer0.bankb.financial.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer0.bankb.financial.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/peers/peer0.bankb.financial.com/tls/ca.crt

peer channel update \
  -o orderer1.orderer.financial.com:7050 \
  -c banking-channel \
  -f ./channel-artifacts/BankBMSPanchors-banking.tx \
  --tls \
  --cafile $ORDERER_CA
```

## 快速測試腳本

創建一個測試腳本方便執行：

```bash
# 退出 CLI 容器
exit

# 在主機創建測試腳本
cat > test-network.sh <<'SCRIPT'
#!/bin/bash

echo "=== 進入 CLI 容器 ==="
docker exec -it cli bash -c '

export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/ordererOrganizations/orderer.financial.com/orderers/orderer1.orderer.financial.com/msp/tlscacerts/tlsca.orderer.financial.com-cert.pem

echo "=== 1. 創建 Banking Channel ==="
peer channel create \
  -o orderer1.orderer.financial.com:7050 \
  -c banking-channel \
  -f ./channel-artifacts/banking-channel.tx \
  --outputBlock ./channel-artifacts/banking-channel.block \
  --tls \
  --cafile $ORDERER_CA

echo "=== 2. BankA Peer0 加入通道 ==="
peer channel join -b ./channel-artifacts/banking-channel.block

echo "=== 3. BankA Peer1 加入通道 ==="
export CORE_PEER_ADDRESS=peer1.banka.financial.com:8051
peer channel join -b ./channel-artifacts/banking-channel.block

echo "=== 4. BankB Peer0 加入通道 ==="
export CORE_PEER_LOCALMSPID="BankBMSP"
export CORE_PEER_ADDRESS=peer0.bankb.financial.com:9051
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/organizations/peerOrganizations/bankb.financial.com/users/Admin@bankb.financial.com/msp
peer channel join -b ./channel-artifacts/banking-channel.block

echo "=== 5. BankB Peer1 加入通道 ==="
export CORE_PEER_ADDRESS=peer1.bankb.financial.com:10051
peer channel join -b ./channel-artifacts/banking-channel.block

echo "=== 6. 驗證通道 ==="
peer channel list

echo "=== 測試完成！ ==="
'
SCRIPT

chmod +x test-network.sh

# 執行測試
./test-network.sh
```

## 成功指標

如果看到以下輸出，表示網路運作正常：
1. ✅ 創建通道成功：`Received block: 0`
2. ✅ 所有 peer 加入成功：`Successfully joined peer to the channel`
3. ✅ `peer channel list` 顯示 `banking-channel`
4. ✅ 沒有 TLS 或連線錯誤

恭喜！你的 Hyperledger Fabric 網路已經成功啟動並可以使用了！🎉
