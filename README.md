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