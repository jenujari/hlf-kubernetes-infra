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

`./scripts/updateAnchorPeer.sh Org1MSP`

`./scripts/updateAnchorPeer.sh Org2MSP`

`./scripts/updateAnchorPeer.sh Org3MSP`

---
