### Using the BYFN which consists of two orgs and four peers for the task of adding a new org dynamically to an existing channel

#### Generate Network Artifacts
``` 
./byfn.sh -m generate
```

#### Start the network
``` 
./byfn.sh -m up 
```

#### Change to org3-artifacts directory and Generate crypto material for Org 3
```
../../bin/cryptogen generate --config=./org3-crypto.yaml
```

#### Print org3 using configtxgen and save the output file as org3.json in scripts directory so that it can be accessed through the cli container
``` 
../../bin/configtxgen -printOrg Org3MSP > ../scripts/org3.json
```

#### Run from first-network dir - Copy crypto-config dir to the current working directory to get the orderer ca certs which are needed for making an invoke
```
cp -r crypto-config/ordererOrganizations org3-artifacts/crypto-config/
```

#### Spin up the cli container
```
docker exec -it cli bash
```

#### Install vim & jq
```
apt update && apt install -y vim jq
```

#### Start configtxlator and set below variables in the cli container
```
configtxlator start &
CONFIGTXLATOR_URL=http://127.0.0.1:7059
ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
CHANNEL_NAME=mychannel
```

#### Fetch the latest config block and write that to config_block.pb
```
peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

#### Decode the block to human readable json format
```
curl -X POST --data-binary @config_block.pb "$CONFIGTXLATOR_URL/protolator/decode/common.Block" | jq . > config_block.json
```

#### Isolating current config
```
jq .data.data[0].payload.data.config config_block.json > config.json
```

#### Append and add org3.json and then write the output to updated_config.json
```
jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./scripts/org3.json >& updated_config.json
```

#### Translating original config to proto
```
curl -X POST --data-binary @config.json "$CONFIGTXLATOR_URL/protolator/encode/common.Config" > config.pb
```

#### Translating updated config to proto
```
curl -X POST --data-binary @updated_config.json "$CONFIGTXLATOR_URL/protolator/encode/common.Config" > updated_config.pb
```

#### Computing config update
```
curl -X POST -F channel=$CHANNEL_NAME -F "original=@config.pb" -F "updated=@updated_config.pb" "${CONFIGTXLATOR_URL}/configtxlator/compute/update-from-configs" > config_update.pb
```

#### Decoding config update
```
curl -X POST --data-binary @config_update.pb "$CONFIGTXLATOR_URL/protolator/decode/common.ConfigUpdate" | jq . > config_update.json
```

#### Generating config update and wrapping it in an envelope
```
echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' | jq . > config_update_in_envelope.json
```

#### Encoding config update envelope
```
curl -X POST --data-binary @config_update_in_envelope.json "$CONFIGTXLATOR_URL/protolator/encode/common.Envelope" > config_update_in_envelope.pb
```

#### Sign with org1
```
peer channel signconfigtx -f config_update_in_envelope.pb
```

#### To switch to Org1 set the below env variables - No need to do the below step at this point
```
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
```

#### Switch to org2 -- so that org2 will sign the pb before updating the channel
```
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
```

#### Sending config update to orderer
```
peer channel update -f config_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls true --cafile $ORDERER_CA
```

#### Spin up the docker compose file for Org3 in a separate terminal
```
docker-compose -f docker-compose-org3.yaml up
```

### Spin up the cli container for Org3 (Org3cli)
```
docker exec -it Org3cli bash
```

#### Switch to peer0 of Org3
```
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=peer0.org3.example.com:7051
```

### Fetch the genesis block to join the peers of Org3 to the existing channel
```
peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

#### Join peer0 of Org3
```
peer channel join -b mychannel.block
```

#### Install chaincode on peer0 of Org3
```
peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/
```

#### Switch to Peer1 of Org3, join it to the channel and install same version of chaincode on it to keep the gossip logs normal

#### The same version of chaincode should be installed on Org1 and Org2
#### switch to org 1 and install cc
```
peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/
```

#### Switch to org 2 and install cc
```
peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go/
```

#### Chaincode upgrade should be done using the same version as install
```
peer chaincode upgrade -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 2.0 -c '{"Args":["init","a","90","b","210"]}' -P "OR ('Org1MSP.member','Org2MSP.member','Org3MSP.member')"
```

#### An invoke/ query should work to verify if the new org is successfully added to the channel
```
peer chaincode invoke -o orderer.example.com:7050  --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```
### Org3 is now successfully added to the existing channel !!..
