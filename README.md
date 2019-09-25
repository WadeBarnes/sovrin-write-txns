# sovrin-write-txns
Test for preparing transactions to write to sovrin network


## Starting von-agent-template on the Sovrin Staging network with un-privileged DID's

You can see the original this process in action here:  https://zoom.us/recording/play/D8WVfMhYakL19PYXoxTBQgBfYMH8bQO372fC75zTJplAycBSj7sTE-fmQy--RFU4?continueMode=true

Indy-cli scripts are all in https://github.com/ianco/sovrin-write-txns

**Note:** You have to manually enter wallet passwords at every step (create, open, export and import).

## Prerequists

The following process uses the Indy-CLI container from [von-network](https://github.com/bcgov/von-network) which provides a containerized `indy-cli` environemnts and facilities to mount a volume containing `indy-cli` batch script templates and perform variable substitution on those templates, turning them into reusable scripts.

TODO: Referance a von-network indy-cli section for details.

Register DIDs:
- TOB
- Agent, with no role.
- Endorser seed is ENDORSER123450000000000000000000 which creates DID DFuDqCYpeDNXLuc3MKooX3

Reset your Indy-CLI container's environemnt:
```
./manage cli reset
```

Initialize the pool for your Indy-CLI container's environemnt:
```
./manage cli init-pool localpool http://192.168.65.3:9000/genesis
```

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

_In the following examples the scripts:_
- Commands are run in your `von-network` directory, as we are using the Indy-CLI container.
- `/c/sovrin-write-txns` is the abosulte path to the scripts contained in this repository.
- The examples are using a local docker environment so some services are being accessed via the DOCKERHOST IP address (192.168.65.3 in these examples).

1. Optional/Recommened - Export the agent's wallet - it will contain a VON-compatible DID with metadata

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli export-wallet \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  exportPath=/tmp/myorg_issuer_only_did.export
```
**Note:** There will be nothing else in the wallet other than a DID at this point.


2. Optional/Recommened - Import a working copy of the wallet

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli import-wallet \
  walletName=local_wallet \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  importPath=/tmp/myorg_issuer_only_did.export
```


3. Write the (author signed) schema to a file so it can be later signed by the (intended) endorser and written to the ledger.

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli create-signed-schema \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  poolName=localpool \
  authorDid=VePGZfzvcgmT3GTdYgpDiT \
  endorserDid=DFuDqCYpeDNXLuc3MKooX3 \
  schemaName=ian-permit.ian-co \
  schemaVersion=1.0.0 \
  schemaAttributes=corp_num,legal_name,permit_id,permit_type,permit_issued_date,permit_status,effective_date \
  outputFile=/tmp/dev_schema_signed.txt
  ```


4. Initialize an endorser wallet

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli create-local-wallet \
  walletName=endorser_wallet \
  walletSeed=ENDORSER123450000000000000000000
```


5. Sign the transaction with your endorser and write to the ledger:

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli write-signed-transaction \
  walletName=endorser_wallet \
  poolName=localpool \
  authorDid=VePGZfzvcgmT3GTdYgpDiT \
  endorserDid=DFuDqCYpeDNXLuc3MKooX3 \
  inputFile=/tmp/dev_schema_signed.txt
```

Make a note of the `seqNo`, you will need it later

```
Response:
{"result":{"auditPath":["6hokYVzPdjWCMk1wBLv1EHEEM4r3tD74yAPS7aECTFES","42T88rYvNWBh83QUkvNHUVfHUikWCBWh1zmP5AUUJLZ3"],"reqSignature":{"values":[{"value":"4UTrHUaZqZYwN7jkitB1n6wcyMo6CApE7BouLxS7rdrUJiWT8KM9FkXBUeTmTaoYwegupVmY8TUqcNFYRdjJPaf7","from":"DFuDqCYpeDNXLuc3MKooX3"},{"value":"nrGatoL1VjKvpLV4EBvGkrqPMiSTeJ4A7QeykjTAmb34CtE4VG32ddT8KGAszdoko659E7XRwy38jC42cJPPAPN","from":"VePGZfzvcgmT3GTdYgpDiT"}],"type":"ED25519"},"rootHash":"DSsBgS5T6WPMiC3AxnzAsywDaHzVH2ptvezwNweQHF1v","txn":{"protocolVersion":2,"metadata":{"digest":"39c2efb17b75edda9b6da5e7939131bbcc1cc86671bb5d4e9dc6b940c39e61e6","payloadDigest":"397051e80fca9b81a77adae5e08e59fa784bdc16dc11b788d9d421ac8b0205e4","endorser":"DFuDqCYpeDNXLuc3MKooX3","from":"VePGZfzvcgmT3GTdYgpDiT","reqId":1569428405919642000},"data":{"data":{"name":"ian-permit.ian-co","attr_names":["legal_name","permit_type","permit_status","effective_date","permit_id","permit_issued_date","corp_num"],"version":"1.0.0"}},"type":"101"},"ver":"1","txnMetadata":{"txnTime":1569438675,"seqNo":10,"txnId":"VePGZfzvcgmT3GTdYgpDiT:2:ian-permit.ian-co:1.0.0"}},"op":"REPLY"}
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