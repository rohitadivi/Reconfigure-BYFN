### Implicit meta policy vs Explicit policy


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

#### continue with the usual process of converting to proto and calculating the difference to obtain delta. Wrap delta in envelope to sign it with Org1 and send update to orderer with Org2

#### Once the update is sent to orderer the Org3 is now added to the channel and the application channel policy is also modified.
