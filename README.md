# Writing Transactions to a Ledger for an Un-privlaged Author
The following steps descride, by example, how schema and cred-def transactions are generated, signed, and written to the ledger for a non-privilaged author.

The following process uses a fully containerized Indy-CLI environment so there is no need to have the Indy-CLI or any of it's dependancies installed on your machine.

The procedure can be used to write transactions to any ledger by simply initializing the containerized Indy-CLI environemnt with the genisis file from the desired pool.

## Prerequists

The following process uses the Indy-CLI container from [von-network](https://github.com/bcgov/von-network) which provides a containerized `indy-cli` environment and facilities to mount a volume containing `indy-cli` batch script templates and perform variable substitution on those templates, turning them into reusable scripts.

Build and start [von-network](https://github.com/bcgov/von-network); `./manage build`, `./manage start`.

Register DIDs (using seeds) for the following purposes using the `von-network` interface `http://localhost:9000`:
- TOB
  - Seed: the_org_book_0000000000000000000
  - DID: Leave blank
  - Alias: the-org-book
  - Role: None

  - Resulting Registration:
    - DID: 25GuF6zjiywpU1iF4kNJPQ
    - Verkey: awcJaDCU36RQxeqMeRKcrVHvXRZU3Ysbj6sCf6W7Q6A

- Agent
  - Seed: 0000000000000000000000000MyAgent
  - DID: Leave blank
  - Alias: my-agent
  - Role: None

  - Resulting Registration:
    - DID: NFP8kaWvCupbDQHQhErwXb
    - Verkey: Cah1iVzdB6UF5HVCJ2ENUrqzAKQsoWgiWyUopcmN3WHd

- Endorser
  - Seed: ENDORSER123450000000000000000000
  - DID: Leave blank
  - Alias: my-endorser
  - Role: Endorser

  - Resulting Registration:
    - DID: DFuDqCYpeDNXLuc3MKooX3
    - Verkey: 7gTxoFyCpMGfxuwBNXYn1icuyh8DqDpRGi12hj5S2pHk

Reset your Indy-CLI container's environemnt:
```
./manage cli reset
```

Initialize the pool for your Indy-CLI container's environemnt:
```
./manage cli init-pool localpool http://192.168.65.3:9000/genesis
```

## Start up the applications


1. Run OrgBook with Sovrin connection parameters (assuming you have already done the `./manage build` to initialize):
```
seed=the_org_book_0000000000000000000 AUTO_REGISTER_DID=0 LEDGER_URL=not_used GENESIS_URL=http://192.168.65.3:9000/genesis ./manage start
```

2. Run von-agent-template (assuming you have already done the `. init.sh` and `./manage build` to initialize):

```
INDY_LEDGER_URL="" AUTO_REGISTER_DID=0 INDY_GENESIS_URL=http://192.168.65.3:9000/genesis WALLET_SEED_VONX=0000000000000000000000000MyAgent ./manage start
```

**Note:** The DID for the application must already exist on the ledger.

**Note:** The agent will fail to fully syncronize since it's DID doesn't have privileges to write the schema(s) and cred-defs to the ledger.

**Note:** Leave the OrgBook and VON Agent services running, we will be using the agent's wallet database.

## Create and Write the Schema and Cred Def to the Ledger

_In the following examples:_
- Commands are run in your `von-network` directory, as we are using the Indy-CLI container.
- `/c/sovrin-write-txns` is the abosulte path to the scripts contained in this repository.
- The examples are using a local docker environment so some services are being accessed via the DOCKERHOST IP address (192.168.65.3 in these examples).

1. Optional/Recommened - Playing the role of the Author - Export the agent's wallet - it will contain a VON-compatible DID with metadata

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli export-wallet \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  exportPath=/tmp/myorg_issuer_wallet_initialized_with_did.export
```
**Note:** This will make a backup of the wallet in it's initialized state.  There will be nothing else in the wallet other than a DID at this point.

2. Playing the role of the Author - Write the (author signed) schema to a file so it can be later signed and written to the ledger by the (intended) endorser.

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli create-signed-schema \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  poolName=localpool \
  authorDid=NFP8kaWvCupbDQHQhErwXb \
  endorserDid=DFuDqCYpeDNXLuc3MKooX3 \
  schemaName=ian-permit.ian-co \
  schemaVersion=1.0.0 \
  schemaAttributes=corp_num,legal_name,permit_id,permit_type,permit_issued_date,permit_status,effective_date \
  outputFile=/tmp/ian-permit.ian-co_author_signed_schema.txn
  ```


3. Playing the role of the Endorser - Initialize a local wallet:

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli create-wallet \
  walletName=endorser_wallet \
  storageType=default \
  storageConfig='{}' \
  storageCredentials='{}' \
  walletSeed=ENDORSER123450000000000000000000
```


4. Playing the role of the Endorser - Sign the transaction and write to the ledger:

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli write-signed-transaction \
  walletName=endorser_wallet \
  storageType=default \
  storageConfig='{}' \
  storageCredentials='{}' \
  poolName=localpool \
  authorDid=NFP8kaWvCupbDQHQhErwXb \
  endorserDid=DFuDqCYpeDNXLuc3MKooX3 \
  inputFile=/tmp/ian-permit.ian-co_author_signed_schema.txn
```

Make a note of the `seqNo` (`"seqNo":10` in this example), you will need to use it for the `schemaId` later

```
Response:
{"result":{"auditPath":["6hokYVzPdjWCMk1wBLv1EHEEM4r3tD74yAPS7aECTFES","42T88rYvNWBh83QUkvNHUVfHUikWCBWh1zmP5AUUJLZ3"],"reqSignature":{"values":[{"value":"4UTrHUaZqZYwN7jkitB1n6wcyMo6CApE7BouLxS7rdrUJiWT8KM9FkXBUeTmTaoYwegupVmY8TUqcNFYRdjJPaf7","from":"DFuDqCYpeDNXLuc3MKooX3"},{"value":"nrGatoL1VjKvpLV4EBvGkrqPMiSTeJ4A7QeykjTAmb34CtE4VG32ddT8KGAszdoko659E7XRwy38jC42cJPPAPN","from":"NFP8kaWvCupbDQHQhErwXb"}],"type":"ED25519"},"rootHash":"DSsBgS5T6WPMiC3AxnzAsywDaHzVH2ptvezwNweQHF1v","txn":{"protocolVersion":2,"metadata":{"digest":"39c2efb17b75edda9b6da5e7939131bbcc1cc86671bb5d4e9dc6b940c39e61e6","payloadDigest":"397051e80fca9b81a77adae5e08e59fa784bdc16dc11b788d9d421ac8b0205e4","endorser":"DFuDqCYpeDNXLuc3MKooX3","from":"NFP8kaWvCupbDQHQhErwXb","reqId":1569428405919642000},"data":{"data":{"name":"ian-permit.ian-co","attr_names":["legal_name","permit_type","permit_status","effective_date","permit_id","permit_issued_date","corp_num"],"version":"1.0.0"}},"type":"101"},"ver":"1","txnMetadata":{"txnTime":1569438675,"seqNo":10,"txnId":"NFP8kaWvCupbDQHQhErwXb:2:ian-permit.ian-co:1.0.0"}},"op":"REPLY"}
```

7. Playing the role of the Author - Create the cred def in the wallet:

Use the value of the `seqNo` you noted in the previous step as the value for `schemaId` in the following call.

```
./manage \
  -v /c/sovrin-write-txns/ \
  cli \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  walletKey=key \
  poolName=localpool \
  authorDid=NFP8kaWvCupbDQHQhErwXb \
  schemaId=10 \
  schemaName=ian-permit.ian-co \
  schemaVersion=1.0.0 \
  schemaAttributes=corp_num,legal_name,permit_id,permit_type,permit_issued_date,permit_status,effective_date \
  python sovrin-write-txns/cred_def.py


./manage \
  -v /c/sovrin-write-txns/ \
  cli \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  walletKey=key \
  poolName=localpool \
  authorDid=NFP8kaWvCupbDQHQhErwXb \
  schemaId=10 \
  schemaName=ian-permit.ian-co \
  schemaVersion=1.0.0 \
  python sovrin-write-txns/cred_def.py

```

Make a note of the `primary` produced by this script, you will need to use it for the `primaryKey` later

9. Write the (author signed) cred def to a file so it can be later signed by the (intended) endorser and written to the ledger.  Plug in the `schemaId` and `primaryKey` you noted from earlier operations.

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli create-signed-cred-def \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  poolName=localpool \
  authorDid=NFP8kaWvCupbDQHQhErwXb \
  endorserDid=DFuDqCYpeDNXLuc3MKooX3 \
  schemaId=10 \
  signatureType=CL \
  tag=tag \
  primaryKey='{"rctxt":"81311556651828459836377717769612911635448872244458565778512992032029100167832917183807321349962996513681532129082242871622622861749585020943360493233881027315647527014965508925623935175100184037934798123389941781596607434594036206347854898950883492664043332129795518145053363242600288869912439512413530519227852842228465991364095621708497981187877260910778883289917052400368875516430537780188605350695010303491497837525299977541461419619163754537398145748030658205338184935807856541238297889092812513494694116232261363412896288060932737466186085454904144600154771904457525267111262778436569836747056721075042911644991","s":"27900122892698256938157205600596521943807336796848136491304582287807584260852850019272872946017238442234073700349469943105059996484766883373746908571525418342304403753990047229421463330378462700958552126509818137935392534608731755808005064084210613368134824633904511574573478942876277642988542869360298470124142785888847007516046850151368823885430091460923782939893329021422127477347849091386092390788539956730170607102375524708512219098671696600315637290841811596436939648612304342073253025121890295188811297943975093895158015782019281689909124066594043297328507633551631556540847538544659660439551713572795499242313","n":"86241241518880406441502800568718188860049904032905421758308931523933748678097974224289301125143861872614886589502290511311375554013605421720305226832158285614801466530645881037052278997098088484226751665979040563337655848383251081843358398519913847973682403167072703097689914058590232759083064762872410429045760797006284935732330883593513115221493810218471396432151706380554467742745930967790880726622328145813493294362636911919153653250902844951761856346171791331039056686093296430674065574562489526558070509332716791945279205542800531425083338691732881199336159450104404845937039409168915330342448692519813842623573","r":{"permit_type":"48805936384448287510230363407279884794759036195695961343986307581609727330327875490167331723392817972136322788914717332513898706284401821625504342124276000116722646534485445325597021635668433983935423607067516524153496709902692614486927122029962908074797144309666543081108721371029735038023067972038466809097232045135050101981641598018354679405558244213247389500547909407515293918271149084339610227286241601300477032863467068757025369779568445282656745788756944946662971156344571588459604890515839752820996751369651591801255111880570477702671217122049089832434056768652493803859704940544142674136694656302712730531966","corp_num":"33046763067113679821870365078412251050131455720931551012146945883086391895384022085319829302557285535243475973040176557959823370829892211840713103973962847228040734812367844385580631759922487945385188917929377025322151127471458647766306211826657350728276666035943490332861886464757598583523190850447407966171451304629188134978422399966348822404475107223158013015974189608505358611861345845258227330621704617597056304999173717292536704995610523431168808282887916148388876633734505738615446427671134740558859362833577831029119945011309336866419631837151849360847014538241606327637010669327269090382690466966184129795984","effective_date":"10882214941931505159084769547754657607536584068900752578235482352600468051092165428005150892600441268764160411534728330130059909406869319578540229975928319218008799704275275237747594381942727755772154405122233713403821257398295503669119093309716582408341070478877737132771771717272229721729366484894040906835555861251536208196123190759359097330299325550145330577321590950824661290381372065322508633369581883650075773088494889865983606258732900474862987430620260390375896270619001094308203388866706685598054415059353179576244186032473341218109971104715974668057968968634675220246705330186171336996647728208012599374507","permit_status":"27946424695000949442318385043674497284118615619862269280783868453737792557908005955663201819671410544336724642120234232190598186706001970141469583314655027182212262007929489699419750678014412099596221381950637367031036774827266589210994386240754690571474763913027195158863225836887618543012137128728082581494050889613034012344254904497127637290865500766106080276983538970633418872087719645917214147378641309409715627935017249196084840167012756586625011780016335996917636138036584810659316119297019808706362555848968347084459141829118086983302474658303810057620259256426910967532404867255907059003363913316568051948533","master_secret":"29636470989553312953512052343767533897770948045793051298323332795262221831445804485815430649675607786694789512263849881461943762423423492614876964439520736400861145143323187524548393012510717632869240430919696000332420826243013736057106950371894812906122431576373850097698390115546137191476210350131281520142581261207040025455115494171772803294977118565703091304952384383709690639936395720577109137352845468602930544713973840252014117601965409141833728680715336134979622323761706483337922004484718294787358322401785352124371936404397283933385302604701379133968758198031687203705823284498414225645145212443921702792343","legal_name":"5660034783836504193742464989651322060665267906930911697047485673948329403905537661353842603982929180582740117698828325407266598440200293899985250862642813591950852393667033160279550169416057243581277765062727786281705617106266547816277439997300192499960107181405606900191414060174974578184011984570159936495656570785800303759853050159948318655778580447799979572039723624629231079917744555525261024319781627568929980527500796592701691797249581983223260017580196699629077619094291772615117233544748841801373439116211356544658709838667071327487610214561467464365222131623474244709387550751354190515710109340775400077241","permit_id":"58398553703945753654015246043939441010268107629449799590488072366917244356458370726918823308523159878811883591973547559770485114184857306299895780289780849449902061372286891605303288342803392379356445487299270021587574976275676022930896395256001337718712824804254533923077909065021935368399229096541740093295801396546265721860966354364210433173499437892617102685356853656586478588171905283560840444303779787266712854616457962984562311283331389858586919644747360103177126252978134651153287580217653045075841521541477507872357950663040533401333597576148820256044755934016540494546723068096960358524687661627736859313012","permit_issued_date":"55803665190282812500960877544167883278279767782934750815120011400671462607518059638435357347146044393011603269264268342661974298907350551466036279375283489166352939167413295422507794583860234943496646017876069549122920721191709074825547596117385159721555584021753553359842129698550593638113428942116984019133939190487368981593631398708670393101580216838804700552415997800446157975427354704278097389789893760892290560137990021221117651831453668684741014933969767011053865315615608001897990557820830044277238101336524579847036524177794666276705513504392372290128517501311686877831023592313226910261867674139972664354705"},"z":"58615272175361682673406939521926115833589826988468340852840763216876573383003132176033958846502876872266686819388854575596761178100784371338190256085017526335515291925098013022984083678941823530486808832436679115482111599202586122341742321569133089331563953080625376610230872855621699439020196533331264559550580399771081421475930049575577806831500026567030531084827539892546004430362777270479821794670677834054804622397785620644409823823856324921853283860988058533788823299167787008577846153112275451089138175534689468260581170035511215235614907482726229930316833761865384209025570625734008653247814608307426461545519"}' \
  outputFile=/tmp/dev_cred_def_signed.txt
  ```

10. Endorser signs and writes to the ledger

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli write-signed-transaction \
  walletName=endorser_wallet \
  poolName=localpool \
  authorDid=NFP8kaWvCupbDQHQhErwXb \
  endorserDid=DFuDqCYpeDNXLuc3MKooX3 \
  inputFile=/tmp/dev_cred_def_signed.txt
  ```


11. Export our "temporary wallet"

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli export-wallet \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  exportPath=/tmp/myorg_issuer_complete.export
```

## Restart the Agent with the New Wallet

1. Kill the VON Agent processes (`./manage rm`)


2. Startup *only* the Agent Wallet Database (so we can import our temporary wallet):

```
./manage start myorg-wallet-db
```


3. Now run the script to import our database:

```
./manage \
  -v /c/sovrin-write-txns \
  indy-cli import-wallet \
  walletName=myorg_issuer \
  storageType=postgres_storage \
  storageConfig='{"url":"192.168.65.3:5435"}' \
  storageCredentials='{"account":"DB_USER","password":"DB_PASSWORD","admin_account":"postgres","admin_password":"mysecretpassword"}' \
  importPath=/tmp/myorg_issuer_complete.export
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
myorg-agent_1      | 2019-09-12 01:03:24,641 INFO [von_anchor.wallet]: Wallet bctob-Verifier-Wallet set seed hash metadata for DID NFP8kaWvCupbDQHQhErwXb
myorg-agent_1      | 2019-09-12 01:03:25,052 INFO [von_anchor.wallet]: Opened wallet myorg_issuer on handle 7
myorg-agent_1      | 2019-09-12 01:03:25,059 INFO [von_anchor.wallet]: Wallet myorg_issuer got verkey GcWd3dYwmptFdGBf4DFMi5jUxmcgBNVoDWRV2ritLh4r for existing DID NFP8kaWvCupbDQHQhErwXb
myorg-agent_1      | 2019-09-12 01:03:26,168 INFO [von_anchor.anchor.base]: myorg_issuer endpoint already set as http://192.168.65.3:5001
myorg-agent_1      | 2019-09-12 01:03:26,168 INFO [vonx.indy.service]: messages.Endpoint stored: http://192.168.65.3:5001
myorg-agent_1      | 2019-09-12 01:03:26,168 INFO [vonx.indy.service]: Checking for schema: ian-permit.ian-co (1.0.7)
myorg-agent_1      | 2019-09-12 01:03:26,333 INFO [von_anchor.anchor.base]: BaseAnchor.get_schema: got schema SchemaKey(origin_did='NFP8kaWvCupbDQHQhErwXb', name='ian-permit.ian-co', version='1.0.7') from ledger
myorg-agent_1      | 2019-09-12 01:03:26,335 INFO [vonx.indy.service]: Checking for credential def: ian-permit.ian-co (1.0.7)
myorg-agent_1      | 2019-09-12 01:03:26,654 INFO [von_anchor.anchor.base]: BaseAnchor.get_cred_def: got cred def NFP8kaWvCupbDQHQhErwXb:3:CL:70371:tag from ledger
myorg-agent_1      | 2019-09-12 01:03:26,654 INFO [vonx.indy.service]: Indy agent synced: ian-co
myorg-agent_1      | 2019-09-12 01:03:27,049 INFO [von_anchor.wallet]: Opened wallet bctob-Verifier-Wallet on handle 13
myorg-agent_1      | 2019-09-12 01:03:27,051 INFO [von_anchor.wallet]: Wallet bctob-Verifier-Wallet got verkey GcWd3dYwmptFdGBf4DFMi5jUxmcgBNVoDWRV2ritLh4r for existing DID NFP8kaWvCupbDQHQhErwXb
myorg-agent_1      | 2019-09-12 01:03:27,217 INFO [von_anchor.anchor.base]: BaseAnchor.get_endpoint: got endpoint for NFP8kaWvCupbDQHQhErwXb from cache
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