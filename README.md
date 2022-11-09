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