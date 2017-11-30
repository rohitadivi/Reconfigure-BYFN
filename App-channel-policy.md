### This document is for my understanding of implicit meta policy and the explicit policy on application channel

#### The default policy on the application channel is implictly defined which can be 'MAJORITY' 'ANY' or 'ALL'

#### To update the application channel policies from imlicit to explicit we need to follow the below steps

#### The policy is updated and defined explicity after adding a new org (ORG3MSP) to the exisiting channel and this can be done by editing the ``` updated_config.json ```

#### To edit the policies scroll down to the section in json file where the system channel policies are defined and change the mod policy for 'Admins' 'Readers' and 'Writers'

#### In the mod policy for admins, change the type to 1 from 3 to indicate that we are defining the policy explicitly

#### Edit the Identities in value field to specify which orgs will be admins for the channel to replace the existing rule of ``` 'MAJORITY' to "n_out_of":{ "n":2 } ```in this case Org1MSP Admin and Org2MSP Admin are defined and need to be signed by both. This is represented as ``` { "signed_by":0 }, { "signed_by":1 } ```
