# sovrin-write-txns
Test of preparing transactions to write to sovrin network


## Starting von-agent-template on the Sovrin Staging network with un-privileged DID's

You can see this process in action here:  https://zoom.us/recording/play/D8WVfMhYakL19PYXoxTBQgBfYMH8bQO372fC75zTJplAycBSj7sTE-fmQy--RFU4?continueMode=true

Indy-cli scripts are all in https://github.com/ianco/sovrin-write-txns

**Note:** You have to manually enter wallet passwords at every step (create, open, export and import).

## Start up the applications


1. Run OrgBook with Sovrin connection parameters (note no local von-network requried):
```
GENESIS_URL=https://raw.githubusercontent.com/sovrin-foundation/sovrin/master/sovrin/pool_transactions_sandbox_genesis AUTO_REGISTER_DID=false LEDGER_URL=https://foo.com ./manage start seed=0000000000000o_anon_research_inc
```


2. Run von-agent-template (assume you have already done the `. init.sh` and `./manage build` to initialize):

```
./manage start
```

**Note:** This requires a DID on the Sovrin network

**Note:** This will fail because the DID doesn't have privileges to write to the Sovrin network

**Note:** Leave the OrgBook and VON Agent services running, we will be using the wallet database

## Create and Write the Schema and Cred Def to the Ledger

1. Optional/Recommened - Export the agent's wallet - it will contain a VON-compatible DID with metadata

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/issuer-0-export-wallet.txt
```

**Note:** There will be nothing else in the wallet other than a DID at this point.


2. Optional/Recommened - Import a working copy of the wallet

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/issuer-1-import-wallet-working-copy.txt
```


3. Write the (author signed) schema to a file so it can be later signed by the (intended) endorser and written to the ledger.

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/issuer-2-create-signed-schema.txt
```


4. Initialize an endorser wallet

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/endorser-1-create-wallet.txt
```


5. Sign the transaction with your endorser and write to the ledger:

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/endorser-2-write-schema.txt
```

Make a note of the `Sequence Number`, you will need it later

```
Following Schema has been received.
Metadata:
+------------------------+-----------------+---------------------+---------------------+
| Identifier             | Sequence Number | Request ID          | Transaction time    |
+------------------------+-----------------+---------------------+---------------------+
| DFuDqCYpeDNXLuc3MKooX3 | 70371           | 1568249112025424000 | 2019-09-12 00:45:09 |
+------------------------+-----------------+---------------------+---------------------+
Data:
+-------------------+---------+---------------------------------------------------------------------------------------------------------+
| Name              | Version | Attributes                                                                                              |
+-------------------+---------+---------------------------------------------------------------------------------------------------------+
| ian-permit.ian-co | 1.0.7   | "permit_id","permit_type","permit_issued_date","permit_status","effective_date","legal_name","corp_num" |
+-------------------+---------+---------------------------------------------------------------------------------------------------------+
```


6. Edit scripts to include the above Sequence Number = `cred_def.py` and `issuer-3-create-cred-def.txt`


7. Create the cred def in our local wallet:

```
python3 cred_def.py
```

Make a note of the `primary` produced by this script, copy and paste it into an editor, and edit to remove all spaces, line feeds etc


8. Edit `issuer-3-create-cred-def.txt` to include the primary from the above step


9. Write the (author signed) cred def to a file so it can be later signed by the (intended) endorser and written to the ledger.

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/issuer-3-create-signed-cred-def.txt
```

10. Endorser signs and writes to the ledger

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/endorser-3-write-cred-def.txt
```


11. Export our "temporary wallet"

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/issuer-4-export-wallet.txt
```

## Restart the Agent with the New Wallet

1. Kill the VON Agent processes (`./manage rm`)


2. Startup *only* the Agent Wallet Database (so we can import our temporary wallet):

```
./manage start myorg-wallet-db
```


3. Now run the script to import our database:

```
indy-cli ~/Projects/sovrin-write-txns/sovrin-staging-ledger/issuer-5-import-wallet.txt
```


4. Last step!  Stop the wallet database and start all Agent processes:

```
./manage stop
./manage start
```

(make sure you ./manage `stop` and *NOT* `rm`)

God willing, you will see output something like this:

```
myorg-agent_1      | 2019-09-12 01:03:24,236 INFO [von_anchor.wallet]: Created wallet bctob-Verifier-Wallet
myorg-agent_1      | 2019-09-12 01:03:24,633 INFO [von_anchor.wallet]: Opened wallet bctob-Verifier-Wallet on handle 6
myorg-agent_1      | 2019-09-12 01:03:24,641 INFO [von_anchor.wallet]: Wallet bctob-Verifier-Wallet set seed hash metadata for DID VePGZfzvcgmT3GTdYgpDiT
myorg-agent_1      | 2019-09-12 01:03:25,052 INFO [von_anchor.wallet]: Opened wallet myorg_issuer on handle 7
myorg-agent_1      | 2019-09-12 01:03:25,059 INFO [von_anchor.wallet]: Wallet myorg_issuer got verkey GcWd3dYwmptFdGBf4DFMi5jUxmcgBNVoDWRV2ritLh4r for existing DID VePGZfzvcgmT3GTdYgpDiT
myorg-agent_1      | 2019-09-12 01:03:26,168 INFO [von_anchor.anchor.base]: myorg_issuer endpoint already set as http://192.168.65.3:5001
myorg-agent_1      | 2019-09-12 01:03:26,168 INFO [vonx.indy.service]: messages.Endpoint stored: http://192.168.65.3:5001
myorg-agent_1      | 2019-09-12 01:03:26,168 INFO [vonx.indy.service]: Checking for schema: ian-permit.ian-co (1.0.7)
myorg-agent_1      | 2019-09-12 01:03:26,333 INFO [von_anchor.anchor.base]: BaseAnchor.get_schema: got schema SchemaKey(origin_did='VePGZfzvcgmT3GTdYgpDiT', name='ian-permit.ian-co', version='1.0.7') from ledger
myorg-agent_1      | 2019-09-12 01:03:26,335 INFO [vonx.indy.service]: Checking for credential def: ian-permit.ian-co (1.0.7)
myorg-agent_1      | 2019-09-12 01:03:26,654 INFO [von_anchor.anchor.base]: BaseAnchor.get_cred_def: got cred def VePGZfzvcgmT3GTdYgpDiT:3:CL:70371:tag from ledger
myorg-agent_1      | 2019-09-12 01:03:26,654 INFO [vonx.indy.service]: Indy agent synced: ian-co
myorg-agent_1      | 2019-09-12 01:03:27,049 INFO [von_anchor.wallet]: Opened wallet bctob-Verifier-Wallet on handle 13
myorg-agent_1      | 2019-09-12 01:03:27,051 INFO [von_anchor.wallet]: Wallet bctob-Verifier-Wallet got verkey GcWd3dYwmptFdGBf4DFMi5jUxmcgBNVoDWRV2ritLh4r for existing DID VePGZfzvcgmT3GTdYgpDiT
myorg-agent_1      | 2019-09-12 01:03:27,217 INFO [von_anchor.anchor.base]: BaseAnchor.get_endpoint: got endpoint for VePGZfzvcgmT3GTdYgpDiT from cache
myorg-agent_1      | 2019-09-12 01:03:28,367 INFO [vonx.indy.service]: messages.Endpoint stored: None
myorg-agent_1      | 2019-09-12 01:03:28,367 INFO [vonx.indy.service]: Indy agent synced: bctob
myorg-agent_1      | 2019-09-12 01:03:28,368 WARNING [vonx.indy.tob]: No file found at logo path: ../config/../assets/img/ian-co-logo.jpg
myorg-agent_1      | 2019-09-12 01:03:30,154 INFO [vonx.common.service]: Starting sync: indy
```

## Cleanup local environment

In between runs of the above:

```
rm /tmp/dev*
rm -rf ~/.indy*
```

... and also restart local Indy ledger