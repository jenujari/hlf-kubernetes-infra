# Hyper ledger fabric infrastructure on kubernetes

### Deployment steps  
---
Create persistent volume and persistent volume claim using hostpath driver.

`kubectl apply -f 1_pv`

---
Deploy ca server using hyperledger/fabric-ca:latest image for 3 org and 1 order service using deployments and create services for every deployment to connect them on special ports. 

`kubectl apply -f 2_ca`

---
Generate certificates using ca server by creating the job with hyperledger/fabric-ca-tools:latest image and running scripts `orderer-certs.sh`, `org1-certs.sh` from scripts folder in that pod and move final resulting certs in organization folder with PVC.

`kubectl apply -f 3_certs`

---
Generate genesis block, connection profile and chanel artifacts using fabric tools by creating the job with hyperledger/fabric-tools:2.4 image and running scripts `createGenesis.sh`, `createChannel.sh` from scripts folder in that pod and move final resulting in respective folder with PVC.

 Note : There is issue with hyperledger/fabric-tools:latest tag and docker can't pull fabric-tools image with latest tag so manually 2.4 tag has to be mention for now.

`kubectl apply -f 4_artifacts`

---
Create deployments and services for orderer

`kubectl apply -f 5_orderer`

---
Create config maps for further use in peer pods. 

`kubectl apply -f 6_configmap`

---
Create deployments and services for 3 orgs.

`cd 7_peers`

`kubectl apply -f org1`

`kubectl apply -f org2`

`kubectl apply -f org3`

---
log in to the shell of org1-cli pod and run following commands to generate mychannel block from the root dir.

`./scripts/createAppChannel.sh`

it will generate mychannel.block file in to the channel-artifacts folder.

now run following command from all the cli pod to join all three org to mychannel.

`peer channel join -b ./channel-artifacts/mychannel.block`

verify that peer has join channel by running following command from all cli pods.

`peer channel list`

run following command from respective cli pod to update the channel for anchor peer.

`./scripts/updateAnchorPeer.sh Org1MSP` {in cli-peer0-org1}

`./scripts/updateAnchorPeer.sh Org2MSP` {in cli-peer0-org2}

`./scripts/updateAnchorPeer.sh Org3MSP` {in cli-peer0-org3}

---
Create packages for all three orgs to install package using peer command in all three peer orgs.

using any linux pod go to below path inside mounted persistent folder and run following two command for org1

`tar cfz code.tar.gz connection.json` {in cli-peer0-org1}

`tar cfz basic-org1.tgz code.tar.gz metadata.json` {in cli-peer0-org1}

update connection file for rest of the ord2 and org3 respectively and run above command for rest of two org. 

`tar cfz basic-org2.tgz code.tar.gz metadata.json` {in cli-peer0-org2}

`tar cfz basic-org3.tgz code.tar.gz metadata.json` {in cli-peer0-org3}

---

from the cli org1 shell run following command to install chaincode on org1 cli pod. And after that for respective rest of the pod.

`cd /opt/gopath/src/github.com/chaincode/basic/packaging/` {in every pod first visit this path}

`peer lifecycle chaincode install basic-org1.tgz` {in cli-peer0-org1}

`peer lifecycle chaincode install basic-org2.tgz` {in cli-peer0-org2}

`peer lifecycle chaincode install basic-org3.tgz` {in cli-peer0-org3}

at success of each command please note down the package identifier generated at end of every command to reference them latter with respective to their orgs. 

---
build docker image for your chain code.

go to chaincode/basic directory.

run the following command to generate docker image for your chain code.

`docker build -t <username>/basic-cc-hlf:1.0 .`

`docker push jhon5456/basic-cc-hlf:1.0`

this will generate docker image and push it to your docker repo.

---

now copy paste your respective Chaincode code package identifier in CHAINCODE_ID variable of respective chaincode deployment file inside the folder 8_cc_deploy/basic deployment file.

`kubectl apply -f 8_8_cc_deploy\basic`

---

now run following command in respective cli pods to approve your chain code for org.

`peer lifecycle chaincode approveformyorg --channelID mychannel --name basic --version 1.0 --init-required --package-id basic:<org1_basic_chain_code_id> --sequence 1 -o orderer:7050 --tls --cafile $ORDERER_CA` {in cli-peer0-org1}

`peer lifecycle chaincode approveformyorg --channelID mychannel --name basic --version 1.0 --init-required --package-id basic:<org2_basic_chain_code_id> --sequence 1 -o orderer:7050 --tls --cafile $ORDERER_CA` {in cli-peer0-org2}

`peer lifecycle chaincode approveformyorg --channelID mychannel --name basic --version 1.0 --init-required --package-id basic:<org3_basic_chain_code_id> --sequence 1 -o orderer:7050 --tls --cafile $ORDERER_CA` {in cli-peer0-org3}

---
Check for commit readiness from any of the cli org.

`peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name basic --version 1.0 --init-required --sequence 1 -o -orderer:7050 --tls --cafile $ORDERER_CA` {in cli-peer0-org3}

will bring output like following 

```
Chaincode definition for chaincode 'basic', version '1.0', sequence '1' on channel 'mychannel' approval status by org:
Org1MSP: true
Org2MSP: true
Org3MSP: true
```

---
commit the chain code with following command 

`peer lifecycle chaincode commit -o orderer:7050 --channelID mychannel --name basic --version 1.0 --sequence 1 --init-required --tls true --cafile $ORDERER_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --peerAddresses peer0-org3:7051 --tlsRootCertFiles /organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt` {in cli-peer0-org3}

will bring output like following 

```
2022-11-11 12:02:51.414 UTC 002c INFO [chaincodeCmd] ClientWait -> txid [b204515291cb07ac7b8291e30999cd75f52cd5b648781eac177203ae27b9ba1d] committed with status (VALID) at peer0-org1:7051
2022-11-11 12:02:51.438 UTC 002d INFO [chaincodeCmd] ClientWait -> txid [b204515291cb07ac7b8291e30999cd75f52cd5b648781eac177203ae27b9ba1d] committed with status (VALID) at peer0-org3:7051
2022-11-11 12:02:51.446 UTC 002e INFO [chaincodeCmd] ClientWait -> txid [b204515291cb07ac7b8291e30999cd75f52cd5b648781eac177203ae27b9ba1d] committed with status (VALID) at peer0-org2:7051
```

---

Init ledger 

`peer chaincode invoke -o orderer:7050 --isInit --tls true --cafile $ORDERER_CA -C mychannel -n basic --peerAddresses peer0-org1:7051 --tlsRootCertFiles /organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --peerAddresses peer0-org3:7051 --tlsRootCertFiles /organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt -c '{"Args":["InitLedger"]}' --waitForEvent` {in cli-peer0-org3}

---

invoke command

`peer chaincode invoke -o orderer:7050 --tls true --cafile $ORDERER_CA -C mychannel -n basic --peerAddresses peer0-org1:7051 --tlsRootCertFiles /organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --peerAddresses peer0-org3:7051 --tlsRootCertFiles /organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt -c '{"Args":["CreateAsset","asset100","red","50","tom","300"]}' --waitForEvent` {in cli-peer0-org2}

---

query command

`peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'` {in cli-peer0-org1}

---

test couch db from local 

`kubectl port-forward -n=hlf services/peer0-org1 5984:5984`

visit http://localhost:5984/_utils/ to access couch db in browser.

---

create docker image for api nodejs express app from api folder.

`docker build -t jhon5456/basic-api-hlf:latest .`

`docker push jhon5456/basic-api-hlf:latest`

now deploy kubernetes deployment for api server with configmaps.

`kubectl apply -f api\k8\configmap.yaml`

`kubectl apply -f api\k8\api.yaml`

port forward for testing api on local.

`kubectl port-forward -n=hlf services/api 4000`

---


