### This document is for my understanding of implicit meta policy and the explicit policy on application channel

##### The default policy on the application channel is implictly defined which can be 'MAJORITY', 'ANY', or 'ALL'

##### To update the application channel policies from implicit to explicit we need to follow the below steps

##### The policy is updated and defined explicitly after adding a new org (ORG3MSP) to the existing channel and this can be done by editing the ``` updated_config.json ```

##### To edit the policies scroll down to the section in json file where the application channel policies are defined and change the mod policy for 'Admins' 'Readers' and 'Writers'

##### In the mod policy for Admins, change the type from '3' to '1' to indicate that we are defining the policy explicitly

##### Edit the Identities in value field to specify which orgs will be admins for the channel to replace the existing rule of ``` 'MAJORITY' to "n_out_of":{ "n":2 } ```in this case Org1MSP Admin and Org2MSP Admin are defined and need to be signed by both. This is represented as ``` { "signed_by":0 }, { "signed_by":1 } ```

##### The type should be changed from '3' to '1' in both Readers and Writers as well to indicate that we are defining the policy explicitly

##### Org3MSP should only be added to Readers and Writers and not to Admins. This will allow the peers of Org3 to read and write to the channel but they cannot sign or make any changes to the channel

##### The last line with "type:0" in the updated_config.json should be removed and make sure to check that the json is valid by checking in json lint

##### Once the above edits are made to the json file continue with the usual process of converting to proto and calculating the difference to obtain delta and wrap it in envelope to sign it with Org1 and send update to orderer with Org2

##### Once the update is sent to orderer the Org3 is now added to the channel and the channel policy is also modified.
