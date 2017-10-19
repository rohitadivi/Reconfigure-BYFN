# Reconfigure-BYFN

### Start configtxlator

configtxlator start &

CONFIGTXLATOR_URL=http://127.0.0.1:7059


### Decode the block to human readable json format

curl -s -X POST --data-binary @mychannel.block "$CONFIGTXLATOR_URL/protolator/decode/common.Block" | jq . > config_block.json


### Isolating current config

jq .data.data[0].payload.data.config config_block.json > config.json


### Generating new config

jq '. * {"channel_group":{"groups":{"Application":{"groups":{"Org3MSP": .channel_group.groups.Application.groups.Org1MSP}}}}}'  config.json  | jq '.channel_group.groups.Application.groups.Org3MSP.values.MSP.value.config.name = "Org3MSP"' > updated_config.json


### TODO: replace all org1MSP instances with Org3MSP

### Translating original config to proto

curl -s -X POST --data-binary @config.json "$CONFIGTXLATOR_URL/protolator/encode/common.Config" > config.pb


### Translating updated config to proto

curl -s -X POST --data-binary @updated_config.json "$CONFIGTXLATOR_URL/protolator/encode/common.Config" > updated_config.pb


### Computing config update

curl -s -X POST -F channel=mychannel -F "original=@config.pb" -F "updated=@updated_config.pb" "${CONFIGTXLATOR_URL}/configtxlator/compute/update-from-configs" > config_update.pb



### Decoding config update

curl -s -X POST --data-binary @config_update.pb "$CONFIGTXLATOR_URL/protolator/decode/common.ConfigUpdate" | jq . > config_update.json


### Generating config update envelope

echo '{"payload":{"header":{"channel_header":{"channel_id":"'mychannel'", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' | jq . > config_update_in_envelope.json


### Encoding config update envelope

curl -s -X POST --data-binary @config_update_in_envelope.json "$CONFIGTXLATOR_URL/protolator/encode/common.Envelope" > config_update_in_envelope.pb


### Sending config update to channel
peer channel update -f config_update_in_envelope.pb -c mychannel -o orderer.example.com:7050 --tls –cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem


### TODO: Before passing it to Orderer we need to sign it with both the existing orgs.
