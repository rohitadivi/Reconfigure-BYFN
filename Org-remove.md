### Removing an Org from a channel

#### Use BYFN to spin up a network with 2 orgs consisting of 4 peers and an orderer

#### Generate network artifacts and bring up the network
```
./byfn.sh -m generate
./byfn.sh -m up
```

#### Spin up the cli container
```
docker exec -it cli bash
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

#### open the config.json file and manually remove Org2 MSP material and write it to updated_config.json file

#### since the original config file is modified to updated_config.json get the original config file again
```
jq .data.data[0].payload.data.config config_block.json > config.json
```

#### continue the remaining steps of computing update and re-marshall with configtxlator

#### sign with Org1 and send update to orderer with Org2

#### now the Org2 is removed from the channel hence it will not be able to make any invokes.

#### Org2 can still query the ledger since it has the record of the ledger till the time it is removed from the channel and the value will be constant

#### To completely remove the Org2 we have to remove it from the system channel also

#### Switch to orderer and fetch the latest config block
```
peer channel fetch config sys_config_block.pb -o orderer.example.com:7050 -c testchainid --tls --cafile $ORDERER_CA
```

#### repeat the above steps of unmarshall remove Org2MSP and remarshall

#### switch to orderer admin user and update the channel

#### Org2 is now successfully removed !!..
