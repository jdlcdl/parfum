# Bitcoin Core Watch-Only Liana Simple-Inheritance TR

Exploring how to use Bitcon Core as a Watch-Only wallet, accessed via
[python-bitcoinrpc](https://github.com/jgarzik/python-bitcoinrpc),
"glued" w/
[SeedQReader](https://github.com/pythcoiner/SeedQReader)
for air-gapped QR-Code signing w/ stateless devices like
[Krux](https://github.com/selfcustody/krux)

Note: The sample python sessions below have been done on `signet`, a bitcoin testnet where coins have no value.

---

## Table of Contents
* [Secrets](#secrets): these would NEVER go online. They're cold-storage backups to be kept secure, used by a stateless signer.
* [Extended Public Keys](#extended-public-keys): The `privacy-sensitive` public portion of keys.
* [Output Descriptor](#output-descriptor): Used to create and restore a watch-only wallet. THIS MUST BE BACKED UP.
* [Watch-Only Wallet](#watch-only-wallet): Used to get new addresses, watch for UTXOs, propose and coordinate PSBTs.
* Primary:
  * [Spend Funds](#primary-spend-funds): Create, update, sign, combine, finalize, and extract a PSBT, then broadcast it.
  * [Evolution of a PSBT](#primary-evolution-of-a-psbt): How a PSBT evolves - from creation to broadcast.
  * [Understanding Programmable Money](#primary-understanding-programmable-money): stepping thru validation of this transaction in `btcdeb`.
* Recovery:
  * [Spend Funds](#recovery-spend-funds)
  * [Evolution of a PSBT](#recovery-evolution-of-a-psbt)
  * [Understanding Programmable Money](#recovery-understanding-programmable-money)

---

## Secrets

### A (primary)

**BIP-39 mnemonic**: `auction crucial trend safe faith barrel orbit roast source stereo discover cart`

**BIP-39 passphrase**: ""

### B (recovery)

**BIP-39 mnemonic**: `orange enter age rug chef denial legend topic identify sign always mother`

**BIP-39 passphrase**: ""

![secrets-pri-rec](secrets-pri-rec.png)


---

## Extended Public Keys

### A (primary)

**BIP-32 master fingerprint**: `07fd816d`

**Derivation Path**: `m/48h/1h/0h/2h`

**Key-Origin**: `[07fd816d/48h/1h/0h/2h]`

**xpub**:
```
tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5
```

**xpub w/ key-origin**:
```
[07fd816d/48h/1h/0h/2h]tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5
```

### B (recovery)

**BIP-32 master fingerprint**: `da855a1f`

**Derivation Path**: `m/48h/1h/0h/2h`

**Key-Origin**: `[da855a1f/48h/1h/0h/2h]`

**xpub**:
```
tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5
```

**xpub w/ key-origin**:
```
[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5
```

---

## Output Descriptor:

### Use Liana to create a Simple Inheritance wallet

Instead of manually creating our own descriptor, we'll be using Liana v9 to create a "simple inheritance" Taproot wallet.  This will be a wallet that allows spending via a single Primary key which expires after some time (in our case, 6 hours) allowing utxos to also be spent via a single Recovery key (once a utxo has 36 confirmations).

Once we've added the above XPUBs-w/-key-origin-info as the primary and recovery keys in Liana wallet, we'll be asked to "Backup your wallet descriptor".  The descriptor:
```
tr([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,and_v(v:pk([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),older(36)))#506utvsp
```

Let's take a moment to inspect this Liana descriptor. We'll add some whitespace and indentation for readability below:
```
tr(
  [07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,
  and_v(
    v:pk([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),
    older(36)
  )
)#506utvsp
```
* Liana altered our hardened indicator from h to ',
* both extended public keys A and B have had /<0;1>/* receive/change paths appended,
* Primary path Key A stands alone, separated by the recovery path by a `,` comma character (`tr()` implies `or()`),
* Key B has been wrapped in Miniscript and_v(v:pkh(B),older(36)) so that both terms must succeed,
* all of above has been wrapped in taproot script tr(), with a bech32 checksum added as the suffix

```python
>>> descriptor = "tr([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,and_v(v:pk([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),older(36)))#506utvsp"
```

### Split the descriptor into receive and change `descriptors list` for Bitcoin Core

```python
>>> import bip380_checksum

>>> descriptors = [
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/0/*").split("#")[0]
    ),
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/1/*").split("#")[0]
    )]

>>> print(repr(descriptors))
["tr([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,and_v(v:pk([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36)))#22y0dhdd", "tr([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*,and_v(v:pk([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*),older(36)))#gcgkq5nc"]
```

---

## Watch-Only Wallet

Instead of using `bitcoin-cli` at the command line, we'll be connecting to Bitcoin Core with python-bitcoinrpc as is done in this [BlockchainCommons tutorial](https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/master/18_4_Accessing_Bitcoind_with_Python.md).  If you're more familiar with `bitcoin-cli`, then know that all `rpc.method_name()` calls below can be made like `bitcoin-cli command` with similar paramaters.

In the examples below, `rpc.` is an authproxy object that is still connected to `bitcoind` -- thanks to an exaggerated configuration setting `rpcservertimeout=600` so that connections remain open for 10m, instead of closing once idle for 30s.

```python
# create the wallet
>>> rpc.createwallet(
    "Liana-SI-tr",  # wallet_name
    True,  # disable_private_keys
    True,  # blank
    "",  # passphrase
    True,  # avoid_reuse
    True,  # descriptors
    False,  # load_on_startup
    False)  # external_signer
{'name': 'Liana-SI-tr', 'warnings': ['Empty string given as passphrase, wallet will not be encrypted.']}

# import its descriptors
>>> rpc.importdescriptors([dict(desc=x, timestamp="now") for x in descriptors])
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]

# view the wallet
>>> pprint(rpc.getwalletinfo())
{'avoid_reuse': True,
 'balance': Decimal('0.00200000'),
 'birthtime': 1738675200,
 'blank': True,
 'descriptors': True,
 'external_signer': False,
 'format': 'sqlite',
 'immature_balance': Decimal('0E-8'),
 'keypoolsize': 0,
 'keypoolsize_hd_internal': 0,
 'lastprocessedblock': {'hash': '000000c8a0f024e021e51a3af5981146ab9a93688ab4b9447ebe3c6d97a8110f',
                        'height': 233966},
 'paytxfee': Decimal('0E-8'),
 'private_keys_enabled': False,
 'scanning': False,
 'txcount': 1,
 'unconfirmed_balance': Decimal('0E-8'),
 'walletname': 'Liana-SI-tr',
 'walletversion': 169900}

# view first 10 receive addresses
>>> pprint(rpc.deriveaddresses(descriptors[0], 9))
['tb1pvpqeka564yfls5ma8ps56q2xd4sn8mstm7pe6nfxezw3wc963pysnda6x9',
 'tb1pgfatfr9w9fa4efzmdghhkwxy6xgjmlw7qhpv7lnxhsxvvwvw724qwcfukw',
 'tb1pevejlzshsf9mw3zp6afnmucumpu2f5cl3wy4jupl30sclvlde9dq60j8cd',
 'tb1pr7gpsykqs9jyjm2a3vyeg6dpp3j7c79l53lsux760pay56ehrygq9q3wyu',
 'tb1pk76gc7834mucf22sn4f8yxar36m6zv3dpv0328kg6cxd26pgufrsmzth7q',
 'tb1pveftxgw6qv3kj0gte0nms4guz4rt6690em7a436r66hqc82u00fsyml8m5',
 'tb1p5l2mvx28v85dl4dhg8y3gztzttucgva22h8k2v5496hmj2mxqzfsml65wv',
 'tb1pgvvpkuflm6ezudeltqys069klx2t6uz35zzmqsgr8lc2skz28dss94t06y',
 'tb1pca535ljvzwk3h2qmpau97fma05zksy3kma58d798a357999cmdsq69t0df',
 'tb1pkpa5365e2uq7vwljfchcrrryrq4u9xrpk2m82thruxgpcq6006mscqkjfc']

# view first 10 change addresses
>>> pprint(rpc.deriveaddresses(descriptors[1], 9))
['tb1prn8cy7j5q8sc2g7ahgexmjfxauwvluvp7m80gvw9rzzrdnpculpsw6h524',
 'tb1pxvzcflpfkyh56l7ycrshdsw49pfeq4j57v9uwf2vh4d8nn8r9xjq5yuqq5',
 'tb1p7gvvxu2ly7fuqmt84shqr6pf5g74usphpjydyy63kkjpctkn92asma0wx5',
 'tb1p97zh65xd5unzv76uwkjp9rut0a9rgehqx3xds6dr82kn77f9drfs939d5j',
 'tb1phapydm7v8cndgrc7u4mdm8ldtgy4stf8exhzd2z9qqdnycahch0qxp3fc0',
 'tb1p5h535n38pnr5rvx39h0gutpghu2ecwh25yqpf3tvuefknku5ac7qhdj56u',
 'tb1pjr3vlemjz85emv9t87x4p6zfmc974hgqkpysyysjw4h2s86dmveqmr4vsk',
 'tb1pj83uut68gwvm7cfg6g4p57a7q64t0455vjfjeennlqcmzgvw8dhsfdlqxt',
 'tb1p6qd79us6mtmfaqa463u9aa5l66gdy5pql9wjwja36jtdt9x3ej0qse0swt',
 'tb1p4z2f8s4xhgtes6ytn65lrxhyhcs55ezllenst4qjezq8mlg443gqgvstuc']

#
# funded first receive address from alt.signetfaucet.com
# recycle-to: tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn
#

# view UTXOs
>>> pprint(rpc.listunspent())
[{'address': 'tb1pvpqeka564yfls5ma8ps56q2xd4sn8mstm7pe6nfxezw3wc963pysnda6x9',
  'amount': Decimal('0.00200000'),
  'confirmations': 3,
  'desc': 'tr([07fd816d/48h/1h/0h/2h/0/0]3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b,and_v(v:pk(0246023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6),older(36)))#n9l8wht8',
  'parent_descs': ["tr([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,and_v(v:pk([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36)))#22y0dhdd"],
  'reused': False,
  'safe': True,
  'scriptPubKey': '512060419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849',
  'solvable': True,
  'spendable': True,
  'txid': 'c9883c2194d6701b60510b10ed3c18047711a1e1fc4f24011dbb23386b01dc03',
  'vout': 1}]
```

---

## Primary: Spend Funds

We'll go from start to finish, spending a single UTXO to three outputs.  We'll `cut` from our python-bitcoinrpc session and `paste` into SeedQReader to create QR-Codes for importing into our stateless air-gapped Signer.  We'll also use SeedQReader to read the signed PSBT from our Signer, `cutting` and `pasting` it back into our python-bitcoinrpc session.

```python

# assemble the inputs from our only utxo, note how much is spendable
>>> unspent = rpc.listunspent()[0]
>>> spendable = unspent['amount']
>>> txin = [dict(txid=unspent['txid'], vout=unspent['vout'])]

# assemble the outputs
>>> txout = []

# we'll need the internal wallet's receive descriptors
>>> rcv_desc = rpc.listdescriptors()['descriptors'][0]

# we'll send 50,000 sats to this wallet
>>> my_addr = rpc.deriveaddresses(rcv_desc['desc'], [rcv_desc['next'], rcv_desc['next']])[0]
>>> my_amt = Decimal("0.0005")
>>> txout.append({my_addr: my_amt})
>>> spendable -= my_amt

# we'll also send 100,000 sats to an external wallet
>>> send_addr = "tb1qaq4h7t68he26m5ezef5hqufflggh68s0x9hpzd"
>>> send_amt = Decimal("0.001")
>>> txout.append({send_addr: send_amt})
>>> spendable -= send_amt

# we'll send the rest back to the faucet recycle address, assuming no fees yet
>>> faucet_addr = "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn"
>>> faucet_amt = spendable
>>> txout.append({faucet_addr: spendable})
>>> spendable -= faucet_amt

# we'll pay 10 sats per vbyte
>>> fee_rate = Decimal("0.0000001")
>>> vsize = rpc.decodepsbt(rpc.createpsbt(txin, txout))['tx']['vsize']
>>> fees = vsize * fee_rate
>>> faucet_amt -= fees
>>> spendable += fees
>>> txout[-1] = {faucet_addr: faucet_amt}
>>> assert my_amt + send_amt + faucet_amt + fees == unspent['amount']

>>> pprint(txin)
[{'txid': 'c9883c2194d6701b60510b10ed3c18047711a1e1fc4f24011dbb23386b01dc03',
  'vout': 1}]

>>> pprint(txout)
[{'tb1pgfatfr9w9fa4efzmdghhkwxy6xgjmlw7qhpv7lnxhsxvvwvw724qwcfukw': Decimal('0.0005')},
 {'tb1qaq4h7t68he26m5ezef5hqufflggh68s0x9hpzd': Decimal('0.001')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00048440')}]

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAJwCAAAAAQPcAWs4I7sdASRP/OGhEXcEGDztEAtRYBtw1pQhPIjJAQAAAAD9////A1DDAAAAAAAAIlEgQnq0jK4qe1ykW2ovezjE0ZEt/d4Fws9+ZrwMxjmO8qqghgEAAAAAABYAFOgrfy9HvlWt0yLKaXBxKfoRfR4POL0AAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yfCRAwAAAQErQA0DAAAAAAAiUSBgQZt2mqkT+FN9OGFNAUZtYTPuC9+DnU0myJ0XYLqISSIVwTw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobJiBGAjt4cuACdXfYpFE4m8WkMOcp8tOJoj7MoSN9w7f31q0BJLLAIRY8ND4GOBt+eSb9vlTGOrAEDFwshJHi1L1yOMuQZgQKGx0AB/2BbTAAAIABAACAAAAAgAIAAIAAAAAAAAAAACEWRgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399Y9AaiaBBy3H13ns7ochEIDN30X545bl0ONt5faG9RYFQl82oVaHzAAAIABAACAAAAAgAIAAIAAAAAAAAAAAAEXIDw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobARggqJoEHLcfXeezuhyEQgM3fRfnjluXQ423l9ob1FgVCXwAAQUgJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNABBigAwCUgytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq2tASSyIQckp+SAiZoU6gFAZn7uHDlcXSEFG1R6E42vq1TMN/a00B0AB/2BbTAAAIABAACAAAAAgAIAAIAAAAAAAQAAACEHytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq09AVeXvZIEBjWGOZC2ALMh60u+Gy35+kLDc4ZGkQyvv0Ne2oVaHzAAAIABAACAAAAAgAIAAIAAAAAAAQAAAAAAAA=='

# result of signing by external signer (in this case, krux)
>>> signed_psbt="cHNidP8BAJwCAAAAAQPcAWs4I7sdASRP/OGhEXcEGDztEAtRYBtw1pQhPIjJAQAAAAD9////A1DDAAAAAAAAIlEgQnq0jK4qe1ykW2ovezjE0ZEt/d4Fws9+ZrwMxjmO8qqghgEAAAAAABYAFOgrfy9HvlWt0yLKaXBxKfoRfR4POL0AAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yfCRAwAAAQErQA0DAAAAAAAiUSBgQZt2mqkT+FN9OGFNAUZtYTPuC9+DnU0myJ0XYLqISQEIQgFAw4NYHoPX3KAuzCetFtGpmICA8Df1KeJileA+MGM8fZnOn7rKpyVygihANZHsBc9EJ8irkgF9/J2Fyzx6T9WCGgETQMODWB6D19ygLswnrRbRqZiAgPA39SniYpXgPjBjPH2Zzp+6yqclcoIoQDWR7AXPRCfIq5IBffydhcs8ek/VghoAAAAA"

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, signed_psbt])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538)

---

## Primary: Evolution of a PSBT

In the above session, the PSBT evolves with each step. Below these changes are explained.

### `psbt = rpc.createpsbt(...`
Bitcoin Core creates the initial psbt template, knowing only the input's `txid` and `vout`, but including no information about what that input is worth or what the spending conditions might be.  Note: because this is segwit, the final pre-segwit `txid` is already known, but not the `hash`.
```json
{
  "tx": {
    "txid": "19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538",
    "hash": "19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538",
    "version": 2,
    "size": 156,
    "vsize": 156,
    "weight": 624,
    "locktime": 233968,
    "vin": [
      {
        "txid": "c9883c2194d6701b60510b10ed3c18047711a1e1fc4f24011dbb23386b01dc03",
        "vout": 1,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967293
      }
    ],
    "vout": [
      {
        "value": 0.00050000,
        "n": 0,
        "scriptPubKey": {
          "asm": "1 427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
          "desc": "rawtr(427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa)#mcpj3fra",
          "hex": "5120427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
          "address": "tb1pgfatfr9w9fa4efzmdghhkwxy6xgjmlw7qhpv7lnxhsxvvwvw724qwcfukw",
          "type": "witness_v1_taproot"
        }
      },
      {
        "value": 0.00100000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 e82b7f2f47be55add322ca69707129fa117d1e0f",
          "desc": "addr(tb1qaq4h7t68he26m5ezef5hqufflggh68s0x9hpzd)#3ntuatq0",
          "hex": "0014e82b7f2f47be55add322ca69707129fa117d1e0f",
          "address": "tb1qaq4h7t68he26m5ezef5hqufflggh68s0x9hpzd",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00048440,
        "n": 2,
        "scriptPubKey": {
          "asm": "0 447fde1e37d97255b5821d2dee816e8f18f6bac9",
          "desc": "addr(tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn)#zau8pzqg",
          "hex": "0014447fde1e37d97255b5821d2dee816e8f18f6bac9",
          "address": "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn",
          "type": "witness_v0_keyhash"
        }
      }
    ]
  },
  "global_xpubs": [
  ],
  "psbt_version": 0,
  "proprietary": [
  ],
  "unknown": {
  },
  "inputs": [
    {
    }
  ],
  "outputs": [
    {
    },
    {
    },
    {
    }
  ]
}
```

### `unsigned_psbt = descriptorprocesspsbt(...`
Bitcoin Core then uses the descriptors to add information pertaining to how the inputs can be spent.  In this case, it fills an empty dictionary with the following content -- within the `inputs` list.
```json
      "witness_utxo": {
        "amount": 0.00200000,
        "scriptPubKey": {
          "asm": "1 60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849",
          "desc": "rawtr(60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849)#undena74",
          "hex": "512060419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849",
          "address": "tb1pvpqeka564yfls5ma8ps56q2xd4sn8mstm7pe6nfxezw3wc963pysnda6x9",
          "type": "witness_v1_taproot"
        }
      },
      "taproot_scripts": [
        {
          "script": "2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ad0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c13c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "a89a041cb71f5de7b3ba1c844203377d17e78e5b97438db797da1bd45815097c"
          ]
        }
      ],
      "taproot_internal_key": "3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
      "taproot_merkle_root": "a89a041cb71f5de7b3ba1c844203377d17e78e5b97438db797da1bd45815097c"
```
It also fills the first empty dictionary in `outputs`, knowing it belongs to this wallet
```json
      "taproot_internal_key": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
      "taproot_tree": [
        {
          "depth": 0,
          "leaf_ver": 192,
          "script": "20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e"
          ]
        }
      ]
```
and knowing the input value, it calculates and includes the fee.
```json
   ,
  "fee": 0.00001560
```

### `signed_psbt = ...`
It is the job of the Signer to display pertinent information to the user so that they understand what is being signed, and to add signatures to the PSBT.  In this case, krux adds two keys to the existing dictionary in `inputs`
```json
      "final_scriptwitness": [
        "c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf4427c8ab92017dfc9d85cb3c7a4fd5821a"
      ],
      "taproot_key_path_sig": "c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf4427c8ab92017dfc9d85cb3c7a4fd5821a"
    
```
and also strips some fields, when "Sign to QR" is used (for a less-bulky QR-code and optimized user experience).  It strips `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root` from `inputs`.  It also strips `taproot_internal_key`, `taproot_tree` and `taproot_bip32_derivs` from `outputs`.

### `combined_psbt = rpc.combinepsbt(...`
Had the air-gapped signer not stripped any fields in its quest for an optimized transmit-via-qrocde user-experience, the resulting `signed_psbt` would have been the same as the `combined_psbt`.  Still, because other types of transactions may require multiple signatures from multiple Signers, it is important for the Combiner to consider combining all signed PSBTs with the original PSBT distributed to Signers.  In this case, Bitcoin Core simply replaces some dictionary keys that had been stripped out.

Bitcoin Core does not restore anything in the the `inputs`, but it does restore keys stripped from the `outputs`
```json
      "taproot_internal_key": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
      "taproot_tree": [
        {
          "depth": 0,
          "leaf_ver": 192,
          "script": "20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e"
          ]
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
As the Finalizer role, Bitcoin Core is tasked to determine whether or not the PSBT is complete and final, by verifying that spending conditions for all inputs have been met.  In this case, it does not alter the combined PSBT.

By default, when the PSBT is complete and final, Bitcoin Core's `finalizepsbt` will also extract the final rawtransaction hex so that it's ready to be broadcasted into the mempool.

```json
{
  "txid": "19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538",
  "hash": "828c3bc4c52b54c205ce00893d0a6d7f31cdc42d8664a507e24281af435e148b",
  "version": 2,
  "size": 224,
  "vsize": 173,
  "weight": 692,
  "locktime": 233968,
  "vin": [
    {
      "txid": "c9883c2194d6701b60510b10ed3c18047711a1e1fc4f24011dbb23386b01dc03",
      "vout": 1,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf4427c8ab92017dfc9d85cb3c7a4fd5821a"
      ],
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
        "desc": "rawtr(427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa)#mcpj3fra",
        "hex": "5120427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
        "address": "tb1pgfatfr9w9fa4efzmdghhkwxy6xgjmlw7qhpv7lnxhsxvvwvw724qwcfukw",
        "type": "witness_v1_taproot"
      }
    },
    {
      "value": 0.00100000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 e82b7f2f47be55add322ca69707129fa117d1e0f",
        "desc": "addr(tb1qaq4h7t68he26m5ezef5hqufflggh68s0x9hpzd)#3ntuatq0",
        "hex": "0014e82b7f2f47be55add322ca69707129fa117d1e0f",
        "address": "tb1qaq4h7t68he26m5ezef5hqufflggh68s0x9hpzd",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 0.00048440,
      "n": 2,
      "scriptPubKey": {
        "asm": "0 447fde1e37d97255b5821d2dee816e8f18f6bac9",
        "desc": "addr(tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn)#zau8pzqg",
        "hex": "0014447fde1e37d97255b5821d2dee816e8f18f6bac9",
        "address": "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn",
        "type": "witness_v0_keyhash"
      }
    }
  ]
}
```

---

## Primary: Understanding Programmable Money

We'll use [btcdeb](https://github.com/bitcoin-core/btcdeb), a bitcoin script debugger, to understand how validation of the above transaction works.

In a nutshell, spending programmable money is about the owner providing `bitcoin script` "inputs" that will be completed by the utxo's `bitcoin scriptPubKey`, completing without error and leaving a `True` value on top of the stack.  It's necessary to know about the `scriptPubKey` that is part of the address from the input utxo -- whose source was the input transaction and which is stored within each node's `utxoset`.  Below is our input (the second output and utxo of our funding transaction):
```json
    {
      "value": 0.00200000,
      "n": 1,
      "scriptPubKey": {
        "asm": "1 60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849",
        "desc": "rawtr(60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849)#undena74",
        "hex": "512060419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849",
        "address": "tb1pvpqeka564yfls5ma8ps56q2xd4sn8mstm7pe6nfxezw3wc963pysnda6x9",
        "type": "witness_v1_taproot"
      }
    }

```

Below, we will start a `btcdeb` session at the bash commandline using the rawtransaction hex of the above transaction, as well as the same for this transaction's only input.
```
btcdeb --quiet --tx=0200000000010103dc016b3823bb1d01244ffce1a1117704183ced100b51601b70d694213c88
c90100000000fdffffff0350c3000000000000225120427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aaa0
86010000000000160014e82b7f2f47be55add322ca69707129fa117d1e0f38bd000000000000160014447fde1e37d97255b5821d2dee81
6e8f18f6bac90140c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf
4427c8ab92017dfc9d85cb3c7a4fd5821af0910300 --txin=02000000000101e1236b3ad0012d8d060288620a89b1657627e1c1ee5cce
7152779966ab9d8f620100000000fdffffff0227655000000000001600143747f0c295771b869a8c90b3a6f4dc97a20ec0f9400d030000
00000022512060419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba884902473044022045287813dbbc2e750aa0e9
29513ebf65906f2439bcfdf5c19e4de62f2164286a0220622136509e0753afa4e41c95a8b82037e9ee50d37069969443bd33df19b73086
012103f3228f2213afa2925d053ab0e034ad67a55aa446b42ff792995ecd1c312e6f7feb910300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 1; value = 200000
got witness stack of size 1
34 bytes (v0=P2WSH, v1=taproot/tapscript)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(c9883c2194, 1)
script                                                           |                                                             stack 
-----------------------------------------------------------------+-------------------------------------------------------------------
60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849 | c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d9...
OP_CHECKSIG                                                      | 
#0000 60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849
```

Even though we used `--quiet`, we have plenty of output.
* we're signing segwit or taproot
* btcdeb will work one input at a time, and it knows the vout and the value in sats
* It knows that signing this transaction initialized the `script` with two values which it pushed onto the stack (last-in-first-out)
* It knows it's dealing with a WSH or Taproot address and that the script seems valid
* It calcs the input's txid and acknowledges it w/ the vout index
* and then it shows us the script (one step per line) on the left with the initial stack from our txinwitness values.

Let's now walk thru, one step at a time (we'll just type: `step` and hit `<enter>` within the btcdeb session)

input's scriptPubKey is pushed onto the stack
```
		<> PUSH stack 60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849
script                                                           |                                                             stack 
-----------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                      |   60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849
                                                                 | c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d9...
#0001 OP_CHECKSIG
```
Signature is checked consuming both items at top of stack, pushing `1` if valid
```
EvalChecksig() sigversion=2
GenericTransactionSignatureChecker::CheckSchnorrSignature(64 len sig, 32 len pubkey, sigversion=2)
  sig         = c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf4427c8ab92017dfc9d85cb3c7a4fd5821a
  pub key     = 60419b769aa913f8537d38614d01466d6133ee0bdf839d4d26c89d1760ba8849
SignatureHashSchnorr(in_pos=0, hash_type=00)
- taproot sighash
- schnorr sighash = ebe9f1fbd91da5f79a8349b6d99416f7fac1d1fe89ccb11956ffcc2aa744d929
  pubkey.VerifySchnorrSignature(sig=c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf4427c8ab92017dfc9d85cb3c7a4fd5821a, sighash=ebe9f1fbd91da5f79a8349b6d99416f7fac1d1fe89ccb11956ffcc2aa744d929):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                           |                                                             stack 
-----------------------------------------------------------------+-------------------------------------------------------------------
                                                                 |                                                                 01
```

```
script                                                           |                                                             stack 
-----------------------------------------------------------------+-------------------------------------------------------------------
                                                                 |                                                                 01
step
at end of script
```

Since the script completed without failure and left a non-zero value on top of the stack... this transactioin is valid.

---

## Recovery: Spend Funds

This time, we'll spend our only utxo to 2 outputs and we'll sign with the Recovery Key B. Because this path has an OP_CSV, (See [BIP-68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki) and [BIP-112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) for how this works), we must be careful setting the input's nSequence value to be greater-than-or-equal-to 36 (recall older(36) in our Miniscript descriptor). When it's time to spend -- at least 36 blocks after our utxo's first confirmation, we'll also set our nLockTime to the current blockheight (or if pre-signing in advance, we'll extend the nLockTime appropriately).

```python

# assemble the inputs from our only utxo, note how much is spendable
# explicitely set nSequence >= older(36)
>>> unspent = rpc.listunspent()[0]
>>> spendable = unspent['amount']
>>> txin = [dict(txid=unspent['txid'], vout=unspent['vout'], sequence=36)]

# assemble the outputs
>>> txout = []

# we'll need the internal wallet's receive descriptors
>>> rcv_desc = rpc.listdescriptors()['descriptors'][0]

# we'll send 35,000 sats to this wallet
>>> my_addr = rpc.deriveaddresses(rcv_desc['desc'], [rcv_desc['next'], rcv_desc['next']])[0]
>>> my_amt = Decimal("0.00035")
>>> txout.append({my_addr: my_amt})
>>> spendable -= my_amt

# we'll send the rest back to the faucet recycle address, assuming no fees yet
>>> faucet_addr = "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn"
>>> faucet_amt = spendable
>>> txout.append({faucet_addr: spendable})
>>> spendable -= faucet_amt

# we'll pay 10 sats per vbyte
>>> fee_rate = Decimal("0.0000001")
>>> vsize = rpc.decodepsbt(rpc.createpsbt(txin, txout))['tx']['vsize']
>>> fees = vsize * fee_rate
>>> faucet_amt -= fees
>>> spendable += fees
>>> txout[-1] = {faucet_addr: faucet_amt}
>>> assert my_amt + faucet_amt + fees == unspent['amount']

>>> pprint(txin)
[{'sequence': 36,
  'txid': '19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538',
  'vout': 0}]

>>> pprint(txout)
[{'tb1pevejlzshsf9mw3zp6afnmucumpu2f5cl3wy4jupl30sclvlde9dq60j8cd': Decimal('0.00035')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00013750')}]

# since preparing in advance, see how much to adjust nLockTime
>>> unspent['confirmations']
22

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount() + 14)

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAH0CAAAAAThlAHH4t4KNSJgcLdVOZDoVzbD/5ayrAmey1lDD5coZAAAAAAAkAAAAAriIAAAAAAAAIlEgyzMviheCS7dEQddTPfMc2Hik0x+LiVlwP4vhj7PtyVq2NQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJGJIDAAABAStQwwAAAAAAACJRIEJ6tIyuKntcpFtqL3s4xNGRLf3eBcLPfma8DMY5jvKqIhXAJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNAmIMrWE3zEgozquhSCJ7EtOq6bRAp+Epy3maezJ4jz2S6trQEkssAhFiSn5ICJmhTqAUBmfu4cOVxdIQUbVHoTja+rVMw39rTQHQAH/YFtMAAAgAEAAIAAAACAAgAAgAAAAAABAAAAIRbK1hN8xIKM6roUgiexLTqum0QKfhKct5mnsyeI89kurT0BV5e9kgQGNYY5kLYAsyHrS74bLfn6QsNzhkaRDK+/Q17ahVofMAAAgAEAAIAAAACAAgAAgAAAAAABAAAAARcgJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNABGCBXl72SBAY1hjmQtgCzIetLvhst+fpCw3OGRpEMr79DXgABBSCyONiZDcTrcyldfweQCRrTcVs+mg5u3w2hmAoCIjQ7uAEGKADAJSDMd6JmM12Yaox4Je1kbx8mrhtmq0mtFeont2scmLYoMq0BJLIhB7I42JkNxOtzKV1/B5AJGtNxWz6aDm7fDaGYCgIiNDu4HQAH/YFtMAAAgAEAAIAAAACAAgAAgAAAAAACAAAAIQfMd6JmM12Yaox4Je1kbx8mrhtmq0mtFeont2scmLYoMj0BYmC6CtbPk8HB0yOaJy42LFzYd3GNKTAgl1pkO+cLNWbahVofMAAAgAEAAIAAAACAAgAAgAAAAAACAAAAAAA='

# result of signing by external signer (in this case, krux)
>>> signed_psbt = "cHNidP8BAH0CAAAAAThlAHH4t4KNSJgcLdVOZDoVzbD/5ayrAmey1lDD5coZAAAAAAAkAAAAAriIAAAAAAAAIlEgyzMviheCS7dEQddTPfMc2Hik0x+LiVlwP4vhj7PtyVq2NQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJGJIDAAABAStQwwAAAAAAACJRIEJ6tIyuKntcpFtqL3s4xNGRLf3eBcLPfma8DMY5jvKqQRTK1hN8xIKM6roUgiexLTqum0QKfhKct5mnsyeI89kurVeXvZIEBjWGOZC2ALMh60u+Gy35+kLDc4ZGkQyvv0NeQA1Nr+WhlWIpjB0AhzhkVAyzCRto/IIGHcAQKeGXPTboOY6qXcvuTJzWPGbFLw24oRQdIGF1NNuYl84CDG5F974AAAA="

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, signed_psbt])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# try to broadcast early
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
  File "/home/smh/bclipy/jgarzik_bitcoinrpc_authproxy.py", line 143, in __call__
    raise JSONRPCException(response['error'])
jgarzik_bitcoinrpc_authproxy.JSONRPCException: -26: non-final

# It's too early, psbt is complete but not yet final.  we'll wait...
>>> rpc.getblockcount()
234010

>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'd1723fe1f5a3c654622dea2ed80bd8fb8aeed7325907dc61e94c00a45cdff4cf'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/d1723fe1f5a3c654622dea2ed80bd8fb8aeed7325907dc61e94c00a45cdff4cf)

---

## Recovery: Evolution of a PSBT

### `psbt = rpc.createpsbt(...`
```json
{
  "tx": {
    "txid": "d1723fe1f5a3c654622dea2ed80bd8fb8aeed7325907dc61e94c00a45cdff4cf",
    "hash": "d1723fe1f5a3c654622dea2ed80bd8fb8aeed7325907dc61e94c00a45cdff4cf",
    "version": 2,
    "size": 125,
    "vsize": 125,
    "weight": 500,
    "locktime": 234008,
    "vin": [
      {
        "txid": "19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 36
      }
    ],
    "vout": [
      {
        "value": 0.00035000,
        "n": 0,
        "scriptPubKey": {
          "asm": "1 cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95a",
          "desc": "rawtr(cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95a)#r8262gu6",
          "hex": "5120cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95a",
          "address": "tb1pevejlzshsf9mw3zp6afnmucumpu2f5cl3wy4jupl30sclvlde9dq60j8cd",
          "type": "witness_v1_taproot"
        }
      },
      {
        "value": 0.00013750,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 447fde1e37d97255b5821d2dee816e8f18f6bac9",
          "desc": "addr(tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn)#zau8pzqg",
          "hex": "0014447fde1e37d97255b5821d2dee816e8f18f6bac9",
          "address": "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn",
          "type": "witness_v0_keyhash"
        }
      }
    ]
  },
  "global_xpubs": [
  ],
  "psbt_version": 0,
  "proprietary": [
  ],
  "unknown": {
  },
  "inputs": [
    {
    }
  ],
  "outputs": [
    {
    },
    {
    }
  ]
}
```

### `unsigned_psbt = descriptorprocesspsbt(...`
Fills `inputs`
```json
      "witness_utxo": {
        "amount": 0.00050000,
        "scriptPubKey": {
          "asm": "1 427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
          "desc": "rawtr(427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa)#mcpj3fra",
          "hex": "5120427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
          "address": "tb1pgfatfr9w9fa4efzmdghhkwxy6xgjmlw7qhpv7lnxhsxvvwvw724qwcfukw",
          "type": "witness_v1_taproot"
        }
      },
      "taproot_scripts": [
        {
          "script": "20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e"
          ]
        }
      ],
      "taproot_internal_key": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
      "taproot_merkle_root": "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e"
```
and `outputs`
```json
      "taproot_internal_key": "b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
      "taproot_tree": [
        {
          "depth": 0,
          "leaf_ver": 192,
          "script": "20cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832ad0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "6260ba0ad6cf93c1c1d3239a272e362c5cd877718d293020975a643be70b3566"
          ]
        }
      ]
```
and `fee`
```json
   ,
  "fee": 0.00001250
```

### `signed_psbt = ...`
Strips from `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root` from `inputs`, adding:
```json
      "taproot_script_path_sigs": [
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "leaf_hash": "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e",
          "sig": "0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e8398eaa5dcbee4c9cd63c66c52f0db8a1141d20617534db9897ce020c6e45f7be"
        }
      ]
```
Also strips `taproot_internal_key`, `taproot_tree` and `taproot_bip32_derivs` from `outputs`.

### `combined_psbt = rpc.combinepsbt(...`
replaces stripped fields from `inputs`
```json
       ,
      "taproot_scripts": [
        {
          "script": "20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e"
          ]
        }
      ],
      "taproot_internal_key": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
      "taproot_merkle_root": "5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf435e"
```
...and `outputs`
```json
      "taproot_internal_key": "b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
      "taproot_tree": [
        {
          "depth": 0,
          "leaf_ver": 192,
          "script": "20cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832ad0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "6260ba0ad6cf93c1c1d3239a272e362c5cd877718d293020975a643be70b3566"
          ]
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
Strips `taproot_script_path_sigs`, `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root` from `inputs` and adds:
```json
      "final_scriptwitness": [
        "0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e8398eaa5dcbee4c9cd63c66c52f0db8a1141d20617534db9897ce020c6e45f7be",
        "20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2",
        "c024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0"
      ]
```

Final transaction
```json
{
  "txid": "d1723fe1f5a3c654622dea2ed80bd8fb8aeed7325907dc61e94c00a45cdff4cf",
  "hash": "ccc521a30e814d6119ccb668253eecafbcd894c756d42971a362e3ec36425b43",
  "version": 2,
  "size": 265,
  "vsize": 160,
  "weight": 640,
  "locktime": 234008,
  "vin": [
    {
      "txid": "19cae5c350d6b26702abace5ffb0cd153a644ed52d1c98488d82b7f871006538",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e8398eaa5dcbee4c9cd63c66c52f0db8a1141d20617534db9897ce020c6e45f7be",
        "20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2",
        "c024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0"
      ],
      "sequence": 36
    }
  ],
  "vout": [
    {
      "value": 0.00035000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95a",
        "desc": "rawtr(cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95a)#r8262gu6",
        "hex": "5120cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95a",
        "address": "tb1pevejlzshsf9mw3zp6afnmucumpu2f5cl3wy4jupl30sclvlde9dq60j8cd",
        "type": "witness_v1_taproot"
      }
    },
    {
      "value": 0.00013750,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 447fde1e37d97255b5821d2dee816e8f18f6bac9",
        "desc": "addr(tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn)#zau8pzqg",
        "hex": "0014447fde1e37d97255b5821d2dee816e8f18f6bac9",
        "address": "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn",
        "type": "witness_v0_keyhash"
      }
    }
  ]
}
```

---

## Recovery: Understanding Programmable Money

Our input utxo's scriptPubKey
```json
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
        "desc": "rawtr(427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa)#mcpj3fra",
        "hex": "5120427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa",
        "address": "tb1pgfatfr9w9fa4efzmdghhkwxy6xgjmlw7qhpv7lnxhsxvvwvw724qwcfukw",
        "type": "witness_v1_taproot"
      }
    },
```

Starting `btcdeb`
```
btcdeb --quiet --tx=0200000000010138650071f8b7828d48981c2dd54e643a15cdb0ffe5acab0267b2d650c3e5ca1900000000002400000002b888000000000000225120cb332f8a17824bb74441d7533df31cd878a4d31f8b8959703f8be18fb3edc95ab635000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac903400d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e8398eaa5dcbee4c9cd63c66c52f0db8a1141d20617534db9897ce020c6e45f7be2520cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b221c024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d018920300 --txin=0200000000010103dc016b3823bb1d01244ffce1a1117704183ced100b51601b70d694213c88c90100000000fdffffff0350c3000000000000225120427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aaa086010000000000160014e82b7f2f47be55add322ca69707129fa117d1e0f38bd000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90140c383581e83d7dca02ecc27ad16d1a9988080f037f529e26295e03e30633c7d99ce9fbacaa725728228403591ec05cf4427c8ab92017dfc9d85cb3c7a4fd5821af0910300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 0; value = 50000
got witness stack of size 3
34 bytes (v0=P2WSH, v1=taproot/tapscript)
Taproot commitment:
- control  = c024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
- program  = 427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa
- script   = 20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2
- path len = 0
- p        = 24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
- q        = 427ab48cae2a7b5ca45b6a2f7b38c4d1912dfdde05c2cf7e66bc0cc6398ef2aa
- k        = 5e43bfaf0c91468673c342faf92d1bbe4beb21b300b690398635060492bd9757          (tap leaf hash)
  (TapLeaf(0xc0 || 20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadad0124b2))
valid script
- generating prevout hash from 1 ins
[+] COutPoint(19cae5c350, 0)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
<<< taproot commitment >>>                                         |                                                               i: 0
Tweak: 24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc... | k: 5797bd92040635863990b600b321eb4bbe1b2df9fa42c3738646910cafbf...
CheckTapTweak                                                      | 
<<< committed script >>>                                           | 
cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead   | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0000 Tweak: 24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
```
TODO:explain
```
- looping over path (0..-1)
- q.CheckTapTweak(p, k, 0) == success
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead   | 0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e...
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0001 CheckTapTweak
```
TODO:explain
```
		<> PUSH stack cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIGVERIFY                                                  |   cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
24                                                                 | 0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e...
OP_CHECKSEQUENCEVERIFY                                             | 
#0002 cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
```
TODO:explain
```
EvalChecksig() sigversion=3
Eval Checksig Tapscript
- sig must not be empty: ok
- validation weight - 50 -> 138
- 32 byte pubkey (new type); schnorr sig check
GenericTransactionSignatureChecker::CheckSchnorrSignature(64 len sig, 32 len pubkey, sigversion=3)
  sig         = 0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e8398eaa5dcbee4c9cd63c66c52f0db8a1141d20617534db9897ce020c6e45f7be
  pub key     = cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
SignatureHashSchnorr(in_pos=0, hash_type=00)
- tapscript sighash
- schnorr sighash = 8381b74abdc49131af006210991799d8f9727b1cf595165641599e84d183a120
  pubkey.VerifySchnorrSignature(sig=0d4dafe5a19562298c1d00873864540cb3091b68fc82061dc01029e1973d36e8398eaa5dcbee4c9cd63c66c52f0db8a1141d20617534db9897ce020c6e45f7be, sighash=8381b74abdc49131af006210991799d8f9727b1cf595165641599e84d183a120):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0003 OP_CHECKSIGVERIFY
```
TODO:explain
```
		<> PUSH stack 24
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSEQUENCEVERIFY                                             |                                                                 24
#0004 24
```
TODO:explain
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 24
#0005 OP_CHECKSEQUENCEVERIFY
```

Since the script completed without failure and left a non-zero value on top of the stack... this transactioin is valid.
