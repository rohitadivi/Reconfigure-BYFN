### Implicit meta policy vs Explicit policy


#### Generate the network artifacts
#### Start the network
#### Change to org3-artifacts directory and Generate crypto material for Org 3
#### Print org3 using configtxgen and save the output file as org3.json in scripts directory so that it can be accessed through the cli container
#### Run from first-network dir - Copy crypto-config dir to the current working directory to get the orderer ca certs which are needed for making an invoke
#### Spin up the cli container
``` docker exec -it cli bash ```
#### Start configtxlator and set below variables in the cli container
```
configtxlator start &
CONFIGTXLATOR_URL=http://127.0.0.1:7059
ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
CHANNEL_NAME=mychannel
```

#### Fetch the latest config block and write that to config_block.pb
``` peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA ```

#### Decode the block to human readable json format
``` curl -X POST --data-binary @config_block.pb "$CONFIGTXLATOR_URL/protolator/decode/common.Block" | jq . > config_block.json ```

#### Isolating current config
``` jq .data.data[0].payload.data.config config_block.json > config.json ```

#### Append and add org3.json and then write the output to updated_config.json
``` jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./scripts/org3.json >& updated_config.json ```

#### Delete the exisiting implicit application channel policy from updated_config.json using jq tool
``` jq 'del(.channel_group.groups.Application.policies)' updated_config.json  > policy_del.json ```

#### Add the explicit policy from the template 
``` jq -s '.[0] * {"channel_group":{"groups":{"Application":{"policies":.[1]}}}}' policy_del.json ./scripts/templates/policy_template.json >& updated_config.json ```
