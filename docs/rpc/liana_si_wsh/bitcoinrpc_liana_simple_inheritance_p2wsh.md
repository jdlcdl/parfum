# Bitcoin Core Watch-Only: Liana Simple-Inheritance WSH

Exploring how to use Bitcon Core as a Watch-Only wallet, accessed via
[python-bitcoinrpc](https://github.com/jgarzik/python-bitcoinrpc),
"glued" w/
[SeedQReader](https://github.com/pythcoiner/SeedQReader)
for air-gapped QR-Code signing w/ stateless devices like
[Krux](https://github.com/selfcustody/krux).

Note: The sample python sessions below have been done on `signet`, a bitcoin testnet where coins have no value.

---

## Table of Contents
* [Secrets](#secrets): these would NEVER go online. They're cold-storage backups to be kept secure, used by a stateless signer.
* [Extended Public Keys](#extended-public-keys): The `privacy-sensitive` public portion of keys.
* [Output Descriptor](#output-descriptor): Used to create and restore a watch-only wallet. THIS MUST BE BACKED UP.
* [Watch-Only Wallet](#watch-only-wallet): Used to get new addresses, watch for UTXOs, propose and coordinate PSBTs.
* Primary path:  
  * [Spend Funds](#primary-spend-funds): Create, update, sign, combine, finalize, and extract a PSBT, then broadcast it.
  * [Evolution of a PSBT](#primary-evolution-of-a-psbt): How a PSBT evolves - from creation to broadcast.
  * [Understanding Programmable Money](#primary-understanding-programmable-money): stepping thru validation of this transaction in `btcdeb`.
* Recovery path:  
  * [Spend Funds](#recovery-spend-funds)
  * [Evolution of a PSBT](#recovery-evolution-of-a-psbt)
  * [Understanding Programmable Money](#recovery-understanding-programmable-money)

---

## Secrets

### A (primary key)

**BIP-39 mnemonic**: `auction crucial trend safe faith barrel orbit roast source stereo discover cart`

**BIP-39 passphrase**: ""

### B (recovery key)

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

Instead of manually creating our own descriptor, we'll be using Liana v9 to create a "simple inheritance" P2WSH wallet.  This will be a wallet that allows spending via a single Primary key which expires after some time (in our case, 6 hours) allowing utxos to also be spent via a single Recovery key (once a utxo has 36 confirmations).

Once we've added the above XPUBs-w/-key-origin-info as the primary and recovery keys in Liana wallet, we'll be asked to "Backup your wallet descriptor".  The descriptor:
```
wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),older(36))))#lz4jfr7g
```

Let's take a moment to inspect this Liana descriptor. We'll add some whitespace and indentation for readability below:
```
wsh(
  or_d(
    pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*),
    and_v(
      v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),
      older(36)
    )
  )
)#lz4jfr7g
```
* Liana altered our hardened indicator from `h` to `'`,
* both extended public keys A and B have had `/<0;1>/*` receive/change paths appended,
* Key A has been wrapped in Miniscript like: `pk(A)`,
* Key B has been wrapped in Miniscript `and_v(v:pkh(B),older(36))` so that both terms must succeed,
* Both of above have been wrapped in Miniscript `or_d()` so that either term succeeding will be sufficient to spend, 
* all of above has been wrapped in witness-script `wsh()`, with a bech32 checksum added as the suffix


### Split the descriptor into receive and change `descriptors list` for Bitcoin Core

```python
# cut-n-pasted from Liana
>>> descriptor = "wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),older(36))))#lz4jfr7g"

>>> import bip380_checksum

>>> descriptors = [
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/0/*").split("#")[0]
    ),
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/1/*").split("#")[0]
    )]

>>> print(repr(descriptors))
["wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36))))#5u9s223k", "wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*),older(36))))#7296gexx"]

# because we might expect Bitcoin Core to have a preference for hardened indicators of `h` over `'`,
# we'll also create another set of effectively the same wallet descriptors, this time using `h` as the
# hardened indicator, so that we can have some expectations of how Bitcoin Core will deal internally.
>>> altdescriptor = descriptor.replace("'", "h").split("#")[0]

>>> altdescriptor = bip380_checksum.descsum_create(altdescriptor)

>>> altdescriptors = [
    bip380_checksum.descsum_create(
        altdescriptor.replace("/<0;1>/*", "/0/*").split("#")[0]
    ),
    bip380_checksum.descsum_create(
        altdescriptor.replace("/<0;1>/*", "/1/*").split("#")[0]
    )]

>>> pprint(repr(altdescriptors))
("['wsh(or_d(pk([07fd816d/48h/1h/0h/2h]tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*),and_v(v:pkh([da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36))))#jsstjx8s', "
 "'wsh(or_d(pk([07fd816d/48h/1h/0h/2h]tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*),and_v(v:pkh([da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*),older(36))))#cxsps4sq']")
```

---

## Watch-Only Wallet

Instead of using `bitcoin-cli` at the command line, we'll be connecting to Bitcoin Core with python-bitcoinrpc as is done in this [BlockchainCommons tutorial](https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/master/18_4_Accessing_Bitcoind_with_Python.md).  If you're more familiar with `bitcoin-cli`, then know that all `rpc.method_name()` calls below can be made like `bitcoin-cli command` with similar paramaters.

In the examples below, `rpc.` is an authproxy object that is still connected to `bitcoind` -- thanks to an exaggerated configuration setting `rpcservertimeout=600` so that connections remain open for 10m, instead of closing once idle for 30s.

```python
# create the wallet
>>> rpc.createwallet(
    "Liana-SI-wsh",  # wallet_name
    True,  # disable_private_keys
    True,  # blank
    "",  # passphrase
    True,  # avoid_reuse
    True,  # descriptors
    False,  # load_on_startup
    False)  # external_signer
{'name': 'Liana-SI-wsh', 'warnings': ['Empty string given as passphrase, wallet will not be encrypted.']}

# import its descriptors
>>> rpc.importdescriptors([dict(desc=x, timestamp="now") for x in descriptors])
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]

# view the wallet
>>> pprint(rpc.getwalletinfo())
{'avoid_reuse': True,
 'balance': Decimal('0E-8'),
 'birthtime': 1738590607,
 'blank': True,
 'descriptors': True,
 'external_signer': False,
 'format': 'sqlite',
 'immature_balance': Decimal('0E-8'),
 'keypoolsize': 0,
 'keypoolsize_hd_internal': 0,
 'lastprocessedblock': {'hash': '00000008d1f58b3f7486142ed4002921746e3361c1bb1ce6b21c7dff98c8b388',
                        'height': 233818},
 'paytxfee': Decimal('0E-8'),
 'private_keys_enabled': False,
 'scanning': False,
 'txcount': 0,
 'unconfirmed_balance': Decimal('0E-8'),
 'walletname': 'Liana-SI-wsh',
 'walletversion': 169900}

# view first 10 receive addresses
>>> pprint(rpc.deriveaddresses(descriptors[0], 9))
['tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw',
 'tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u',
 'tb1q02lgs98r36vpzpkpukndx45l00fyngjmt86dky9czygefcsfdunqhqr8sj',
 'tb1q7pgdah8vx8v2c5v4gm4jv09zkt7ggkhh539szl0yj2296n54d3jq8jkaxk',
 'tb1qetupua7gg8mvvtxzkwzrlwtfaprgu6fwy0fqfmspyp9v7n8q4rgs0nxez6',
 'tb1q0ns9e7a8msd4hamvcnxk6j0jkev55ssarun5p5vm0x6u9t9n4rhq5nd2a5',
 'tb1qpp49qru32ewnlejr0t0zqgl4szatduc8gax0vshh29rky9gtxl9qc7zr86',
 'tb1qh00wwamn2mxsltgls35lddnra6tn28ujxz8dsc67fuv62c5ldk2s5dc9ss',
 'tb1qm2a68rs033l3qhx08t6tfm9cuqav2gd7heqpeez0dd9cl50elwnsd0jrnc',
 'tb1qnandt4ysc2tmw58ky2t270nwlztqgd5g28442fh5s2ywut2jelvsznmug6']

# view first 10 change addresses
>>> pprint(rpc.deriveaddresses(descriptors[1], 9))
['tb1que236a7xkeme93dajej9nleyf8kpus2yjqy907h8u9awh32zv4fqqmhnt2',
 'tb1qfnhvq2m0fz5glqf0fjp3egrzcdc06yswq6ylt5xp8u0qy4l8yu0s5y83r2',
 'tb1qn9yzzxhfujcg3y24fhf6eg9vj890789gljjgrqnpdlt07609h4rseepa5m',
 'tb1qddh7q3k9ma0s2m73pxut4fa4m6h0fkeandc6ayeezgmpyjhym7hqa5dhgt',
 'tb1qql0vdx5zeyujvhgmgjdu023vgyta77gdyep0qtedxqma2tj04h9qmx9zej',
 'tb1qaq4zhfqs874mncndldwg9etcvf0mdxzhgs9wm0xhnqyumn28wanq4n9ff2',
 'tb1q5vdqgqhdmuae45uvy5zg4nuw7ppfknee3ze2f4l4w64lrcwmaw7q6x3g8u',
 'tb1qk5d9paamc7fcfxnddmgl59tsp6rvgv8w9w5gu7s5h5jw2r4nljqqmyc58z',
 'tb1qny8xj8uxvjfa9x96e3taw96vql4llur4052aa3jyyg50ehvazdhs5pgxzz',
 'tb1qjvhgauyyujzke7jvhk40nnumfny4ktlkrxsphs32nnglvvulhy4sa95lsx']

# From previous exercises, we'd expect that Bitcoin Core alters descriptors to use
# `h` hardened indicators in the key-origin, but as we verify below, we can see that
# nothing at all was altered?!?!  Even the descriptor checksums are the same as above.

>>> rpc.listdescriptors()
{'wallet_name': 'Liana-SI-wsh', 'descriptors': [{'desc': "wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36))))#5u9s223k", 'timestamp': 1738590607, 'active': False, 'range': [0, 999], 'next': 0, 'next_index': 0}, {'desc': "wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*),older(36))))#7296gexx", 'timestamp': 1738590607, 'active': False, 'range': [0, 999], 'next': 0, 'next_index': 0}]}

#
# funded first receive address from alt.signetfaucet.com
# recycle-to: tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn
#

# view UTXOs
>>> pprint(rpc.listunspent())
[{'txid': 'a371beff15d028353500b31448cc2923fc9e2560a37c46cddd48c71743da899a', 'vout': 128, 'address': 'tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw', 'witnessScript': '21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b268', 'scriptPubKey': '0020cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717', 'amount': Decimal('0.25540109'), 'confirmations': 1, 'spendable': True, 'solvable': True, 'desc': 'wsh(or_d(pk([07fd816d/48h/1h/0h/2h/0/0]033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b),and_v(v:pkh([da855a1f/48h/1h/0h/2h/0/0]0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6),older(36))))#0w8svtjk', 'parent_descs': ["wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36))))#5u9s223k"], 'reused': False, 'safe': True}]

# We'll note above, that Bitcoin Core HAS INDEED updated the hardened indicators 
# for the descriptor strings attached to this utxo. These descriptors are much different
# Their key-origins have expanded derivation to express the first receive and change addresses,
# and their key-expressions are non-extendable compact public-keys.

# Still, this wallet's descriptors remain unchanged.
>>> rpc.listdescriptors()
{'wallet_name': 'Liana-SI-wsh', 'descriptors': [{'desc': "wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),older(36))))#5u9s223k", 'timestamp': 1738590607, 'active': False, 'range': [0, 1000], 'next': 1, 'next_index': 1}, {'desc': "wsh(or_d(pk([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*),and_v(v:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*),older(36))))#7296gexx", 'timestamp': 1738590607, 'active': False, 'range': [0, 999], 'next': 0, 'next_index': 0}]}

# Needless to say, wallet descriptors are malleable (for the same wallet), and perhaps 
# using a descriptor checksum as a particular wallet's identifier is not so useful,
# at this point in time.
```

---

## Primary: Spend Funds

We'll go from start to finish, spending a single UTXO to three outputs.  We'll `cut` from our python-bitcoinrpc session and `paste` into SeedQReader to create QR-Codes for importing into our stateless air-gapped Signer.  We'll also use SeedQReader to read the signed PSBT from our Signer, `cutting` and `pasting` it back into our python-bitcoinrpc session.

<details><summary>via Liana</summary>
<p>
Liana would want us to sign the following PSBT

```
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
```

decoded:
```json
{
  "tx": {
    "txid": "b5e5dac50a91a1dcc267958c0f05f48bb5f4137d7680bfca1ae522512c817a64",
    "hash": "b5e5dac50a91a1dcc267958c0f05f48bb5f4137d7680bfca1ae522512c817a64",
    "version": 2,
    "size": 156,
    "vsize": 156,
    "weight": 624,
    "locktime": 0,
    "vin": [
      {
        "txid": "a371beff15d028353500b31448cc2923fc9e2560a37c46cddd48c71743da899a",
        "vout": 128,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967293
      }
    ],
    "vout": [
      {
        "value": 0.25400000,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 a47c08b9e9332577a4f1f4c9abae59c4aacd2e86",
          "desc": "addr(tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve)#040gd8sk",
          "hex": "0014a47c08b9e9332577a4f1f4c9abae59c4aacd2e86",
          "address": "tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00080000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
          "desc": "addr(tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u)#5tef56at",
          "hex": "0020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
          "address": "tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u",
          "type": "witness_v0_scripthash"
        }
      },
      {
        "value": 0.00058189,
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
      "witness_utxo": {
        "amount": 0.25540109,
        "scriptPubKey": {
          "asm": "0 cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
          "desc": "addr(tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw)#7x8kgusx",
          "hex": "0020cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
          "address": "tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw",
          "type": "witness_v0_scripthash"
        }
      },
      "non_witness_utxo": {
        "txid": "a371beff15d028353500b31448cc2923fc9e2560a37c46cddd48c71743da899a",
        "hash": "c26517890d35502cb0f4587f3d9bf79b5ccf65b68de7791a60b2d77011fce1ad",
        "version": 3,
        "size": 7741,
        "vsize": 7690,
        "weight": 30760,
        "locktime": 233819,
        "vin": [
          {
            "txid": "64f4c4055ed88ac8e253a1153a68924b47983954bef898f4c268dc39bc8e69df",
            "vout": 0,
            "scriptSig": {
              "asm": "",
              "hex": ""
            },
            "txinwitness": [
              "7c0f1f2fa0f562a21a16901349d506ec52f3b7cb88137ea613b64066b2b74148428e9408dcda94031104d5e997030e9874365f1536bbccb8dcfc43a2c82d54ca"
            ],
            "sequence": 4294967293
          }
        ],
        "vout": [
          {
            "value": 2552.98632830,
            "n": 0,
            "scriptPubKey": {
              "asm": "1 aac35fe91f20d48816b3c83011d117efa35acd2414d36c1e02b0f29fc3106d90",
              "desc": "rawtr(aac35fe91f20d48816b3c83011d117efa35acd2414d36c1e02b0f29fc3106d90)#au3flvfu",
              "hex": "5120aac35fe91f20d48816b3c83011d117efa35acd2414d36c1e02b0f29fc3106d90",
              "address": "tb1p4tp4l6glyr2gs94neqcpr5gha7344nfyznfkc8szkreflscsdkgqsdent4",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.00000000,
            "n": 1,
            "scriptPubKey": {
              "asm": "OP_RETURN 616c742e7369676e65746661756365742e636f6d",
              "desc": "raw(6a14616c742e7369676e65746661756365742e636f6d)#n2h8u9ne",
              "hex": "6a14616c742e7369676e65746661756365742e636f6d",
              "type": "nulldata"
            }
          },
          {
            "value": 0.25540109,
            "n": 2,
            "scriptPubKey": {
              "asm": "0 d546529ed49fbb6ef9ae96f93719b00cb206f3f2",
              "desc": "addr(tb1q64r998k5n7aka7dwjmunwxdspjeqduljhv6xlr)#t0u7uc6r",
              "hex": "0014d546529ed49fbb6ef9ae96f93719b00cb206f3f2",
              "address": "tb1q64r998k5n7aka7dwjmunwxdspjeqduljhv6xlr",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 3,
            "scriptPubKey": {
              "asm": "1 1a1f4da28a52826763fb10c99a8d622f4878fc5fc9cf859038b5ea6a1a032e62",
              "desc": "rawtr(1a1f4da28a52826763fb10c99a8d622f4878fc5fc9cf859038b5ea6a1a032e62)#3z4ecjjq",
              "hex": "51201a1f4da28a52826763fb10c99a8d622f4878fc5fc9cf859038b5ea6a1a032e62",
              "address": "tb1prg05mg5222pxwclmzrye4rtz9ay83lzle88ctypckh4x5xsr9e3qvux4ts",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 4,
            "scriptPubKey": {
              "asm": "1 f947fbf9a49fe4f1148a14a49122e327bfd4a26c8e1919eba9dc12c1d95f50b6",
              "desc": "rawtr(f947fbf9a49fe4f1148a14a49122e327bfd4a26c8e1919eba9dc12c1d95f50b6)#2kglck9m",
              "hex": "5120f947fbf9a49fe4f1148a14a49122e327bfd4a26c8e1919eba9dc12c1d95f50b6",
              "address": "tb1pl9rlh7dynlj0z9y2zjjfzghry7lafgnv3cv3n6afmsfvrk2l2zmqthynxm",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 5,
            "scriptPubKey": {
              "asm": "0 780c8d3869423c5d970a052acec7b4b6eaf166db",
              "desc": "addr(tb1q0qxg6wrfgg79m9c2q54va3a5km40zekmrm9xwr)#5dmn0w30",
              "hex": "0014780c8d3869423c5d970a052acec7b4b6eaf166db",
              "address": "tb1q0qxg6wrfgg79m9c2q54va3a5km40zekmrm9xwr",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 6,
            "scriptPubKey": {
              "asm": "0 abc0c275abcae88a571dd921e2b0a00bc7a71633",
              "desc": "addr(tb1q40qvyadtet5g54camys79v9qp0r6w93na440gq)#stu5r8cd",
              "hex": "0014abc0c275abcae88a571dd921e2b0a00bc7a71633",
              "address": "tb1q40qvyadtet5g54camys79v9qp0r6w93na440gq",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 7,
            "scriptPubKey": {
              "asm": "1 b34c88ee56ae027b5f6070f83f0eaba28f03108f23fc033ea007ca655956a56a",
              "desc": "rawtr(b34c88ee56ae027b5f6070f83f0eaba28f03108f23fc033ea007ca655956a56a)#9e5xv7me",
              "hex": "5120b34c88ee56ae027b5f6070f83f0eaba28f03108f23fc033ea007ca655956a56a",
              "address": "tb1pkdxg3mjk4cp8khmqwrur7r4t528sxyy0y07qx04qql9x2k2k544q4wvx63",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 8,
            "scriptPubKey": {
              "asm": "1 26a312cc93ca503c1ab630fb7b6f6437f26176f86ac5f41a47238417de7cac6d",
              "desc": "rawtr(26a312cc93ca503c1ab630fb7b6f6437f26176f86ac5f41a47238417de7cac6d)#52vxegav",
              "hex": "512026a312cc93ca503c1ab630fb7b6f6437f26176f86ac5f41a47238417de7cac6d",
              "address": "tb1py6339nynefgrcx4kxrahkmmyxlexzahcdtzlgxj8ywzp0hnu43kst6khhl",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 9,
            "scriptPubKey": {
              "asm": "0 aed16c0a3815034b6932625f5e3e84cba4ea2e3f",
              "desc": "addr(tb1q4mgkcz3cz5p5k6fjvf04u05yewjw5t3llpd0ha)#wadc0qf0",
              "hex": "0014aed16c0a3815034b6932625f5e3e84cba4ea2e3f",
              "address": "tb1q4mgkcz3cz5p5k6fjvf04u05yewjw5t3llpd0ha",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 10,
            "scriptPubKey": {
              "asm": "1 6ab2154424ce007a343523d9b22e03ed1b8f69db1d9f783446d4590acdf7ccef",
              "desc": "rawtr(6ab2154424ce007a343523d9b22e03ed1b8f69db1d9f783446d4590acdf7ccef)#6l3nv9v2",
              "hex": "51206ab2154424ce007a343523d9b22e03ed1b8f69db1d9f783446d4590acdf7ccef",
              "address": "tb1pd2ep23pyecq85dp4y0vmytsra5dc76wmrk0hsdzx63vs4n0henhsv77mha",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 11,
            "scriptPubKey": {
              "asm": "0 e4b8586879e3d82ceb1074cb1e675e07bd655205",
              "desc": "addr(tb1quju9s6reu0vze6cswn93ue67q77k25s9c942wh)#v6292yg8",
              "hex": "0014e4b8586879e3d82ceb1074cb1e675e07bd655205",
              "address": "tb1quju9s6reu0vze6cswn93ue67q77k25s9c942wh",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 12,
            "scriptPubKey": {
              "asm": "0 c11eac2f67bd0e5a4df6f15d554e992cc3d970e1",
              "desc": "addr(tb1qcy02ctm8h5895n0k79w42n5e9npaju8puncwtk)#hhgehg7s",
              "hex": "0014c11eac2f67bd0e5a4df6f15d554e992cc3d970e1",
              "address": "tb1qcy02ctm8h5895n0k79w42n5e9npaju8puncwtk",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 13,
            "scriptPubKey": {
              "asm": "1 09d55a1144284dbb2595362365a48c66a620f54330cd1db2ff6345daac3dbfdd",
              "desc": "rawtr(09d55a1144284dbb2595362365a48c66a620f54330cd1db2ff6345daac3dbfdd)#3d7cytq0",
              "hex": "512009d55a1144284dbb2595362365a48c66a620f54330cd1db2ff6345daac3dbfdd",
              "address": "tb1pp8245y2y9pxmkfv4xc3ktfyvv6nzpa2rxrx3mvhlvdza4tpahlws77hrz0",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 14,
            "scriptPubKey": {
              "asm": "1 caa84a0715d7346856e9bd58f4a556ea16244eafc98e3332a8514e4a29be243f",
              "desc": "rawtr(caa84a0715d7346856e9bd58f4a556ea16244eafc98e3332a8514e4a29be243f)#m0u5e2kh",
              "hex": "5120caa84a0715d7346856e9bd58f4a556ea16244eafc98e3332a8514e4a29be243f",
              "address": "tb1pe25y5pc46u6xs4hfh4v0ff2kagtzgn40ex8rxv4g298y52d7yslsdw8cad",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 15,
            "scriptPubKey": {
              "asm": "1 31dbb15650bc94de1f9757bf282d460e7dbf0adb2e0cf71a92a5902898ebd7a5",
              "desc": "rawtr(31dbb15650bc94de1f9757bf282d460e7dbf0adb2e0cf71a92a5902898ebd7a5)#l7mktrrv",
              "hex": "512031dbb15650bc94de1f9757bf282d460e7dbf0adb2e0cf71a92a5902898ebd7a5",
              "address": "tb1px8dmz4jshj2du8uh27ljst2xpe7m7zkm9cx0wx5j5kgz3x8t67jssrdj4m",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 16,
            "scriptPubKey": {
              "asm": "0 690a4cb2d4f04ce6f4a01eecb49a2236101a8857",
              "desc": "addr(tb1qdy9yevk57pxwda9qrmktfx3zxcgp4zzhypvgp2)#p73cz3ng",
              "hex": "0014690a4cb2d4f04ce6f4a01eecb49a2236101a8857",
              "address": "tb1qdy9yevk57pxwda9qrmktfx3zxcgp4zzhypvgp2",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 17,
            "scriptPubKey": {
              "asm": "1 a70bb56c9fb853e8a3b715c716a075bb400330f8a2d794d7f482f521e66611d1",
              "desc": "rawtr(a70bb56c9fb853e8a3b715c716a075bb400330f8a2d794d7f482f521e66611d1)#apw4wtfp",
              "hex": "5120a70bb56c9fb853e8a3b715c716a075bb400330f8a2d794d7f482f521e66611d1",
              "address": "tb1p5u9m2mylhpf73gahzhr3dgr4hdqqxv8c5ttef4l5st6jrenxz8gs8nwrg7",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 18,
            "scriptPubKey": {
              "asm": "0 8f6389075f85daf64485ddaa9a9e3ee8b467959c",
              "desc": "addr(tb1q3a3cjp6lshd0v3y9mk4f4837az6x09vuexlu0s)#tp5l5h2w",
              "hex": "00148f6389075f85daf64485ddaa9a9e3ee8b467959c",
              "address": "tb1q3a3cjp6lshd0v3y9mk4f4837az6x09vuexlu0s",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 19,
            "scriptPubKey": {
              "asm": "0 e2b41c3476fc131307e6a015866e5f1d6a6479a4",
              "desc": "addr(tb1qu26pcdrklsf3xplx5q2cvmjlr44xg7dy84pqu7)#3msh8gac",
              "hex": "0014e2b41c3476fc131307e6a015866e5f1d6a6479a4",
              "address": "tb1qu26pcdrklsf3xplx5q2cvmjlr44xg7dy84pqu7",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 20,
            "scriptPubKey": {
              "asm": "1 1ddc0f2a73a1478a1317810f1d2a5a280a0c7186ef38dfb6924de0b6e56e7233",
              "desc": "rawtr(1ddc0f2a73a1478a1317810f1d2a5a280a0c7186ef38dfb6924de0b6e56e7233)#5ukkm2xk",
              "hex": "51201ddc0f2a73a1478a1317810f1d2a5a280a0c7186ef38dfb6924de0b6e56e7233",
              "address": "tb1prhwq72nn59rc5ychsy8362j69q9qcuvxauudld5jfhstdetwwges2dffk7",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 21,
            "scriptPubKey": {
              "asm": "0 a43a5969f5867af70da345c552cb6a49116d2b91",
              "desc": "addr(tb1q5sa9j604sea0wrdrghz49jm2fygk62u3mv474r)#ffykhk07",
              "hex": "0014a43a5969f5867af70da345c552cb6a49116d2b91",
              "address": "tb1q5sa9j604sea0wrdrghz49jm2fygk62u3mv474r",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 22,
            "scriptPubKey": {
              "asm": "0 da97cb4d71d3748cfaad133291894a3de285d2ed",
              "desc": "addr(tb1qm2tuknt36d6ge74dzvefrz228h3gt5hdac3rd8)#ea4ez4mx",
              "hex": "0014da97cb4d71d3748cfaad133291894a3de285d2ed",
              "address": "tb1qm2tuknt36d6ge74dzvefrz228h3gt5hdac3rd8",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 23,
            "scriptPubKey": {
              "asm": "1 e07783777e903d3b0c5ceb0ada0cb1a4f22ce824d7849a7fba222ffe97d73a87",
              "desc": "rawtr(e07783777e903d3b0c5ceb0ada0cb1a4f22ce824d7849a7fba222ffe97d73a87)#95rhnx8j",
              "hex": "5120e07783777e903d3b0c5ceb0ada0cb1a4f22ce824d7849a7fba222ffe97d73a87",
              "address": "tb1pupmcxam7jq7nkrzuav9d5r935neze6py67zf5la6yghla97h82rsj4tn0m",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 24,
            "scriptPubKey": {
              "asm": "0 2053023dfa67c90b1c0fab8a492180292e40d037",
              "desc": "addr(tb1qypfsy006vlysk8q04w9yjgvq9yhyp5php4uu7l)#f2uezhku",
              "hex": "00142053023dfa67c90b1c0fab8a492180292e40d037",
              "address": "tb1qypfsy006vlysk8q04w9yjgvq9yhyp5php4uu7l",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 25,
            "scriptPubKey": {
              "asm": "0 ad0490ede4e87c09c8196f2f101710c1454e9e32",
              "desc": "addr(tb1q45zfpm0yap7qnjqeduh3q9csc9z5a83j3drs7k)#fcqcwy5f",
              "hex": "0014ad0490ede4e87c09c8196f2f101710c1454e9e32",
              "address": "tb1q45zfpm0yap7qnjqeduh3q9csc9z5a83j3drs7k",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 26,
            "scriptPubKey": {
              "asm": "1 3817d64c803c3a364d96d69b93a9e321d6a62fb49f3f38550afe84d03e4aca39",
              "desc": "rawtr(3817d64c803c3a364d96d69b93a9e321d6a62fb49f3f38550afe84d03e4aca39)#psgl60j0",
              "hex": "51203817d64c803c3a364d96d69b93a9e321d6a62fb49f3f38550afe84d03e4aca39",
              "address": "tb1p8qtavnyq8sarvnvk66de820ry8t2vta5nulns4g2l6zdq0j2eguszs5w5k",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 27,
            "scriptPubKey": {
              "asm": "1 a8caa3782ac5c46679ee75b6f2741c5c3eda37d16c9d60d9a6b69e4ecd55458c",
              "desc": "rawtr(a8caa3782ac5c46679ee75b6f2741c5c3eda37d16c9d60d9a6b69e4ecd55458c)#r9lvuf43",
              "hex": "5120a8caa3782ac5c46679ee75b6f2741c5c3eda37d16c9d60d9a6b69e4ecd55458c",
              "address": "tb1p4r92x7p2chzxv70wwkm0yaqutsld5d73djwkpkdxk60yan24gkxqmclxpd",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 28,
            "scriptPubKey": {
              "asm": "0 32524d813b5ef0640b76ffe0fe217ce48288046f",
              "desc": "addr(tb1qxffymqfmtmcxgzmklls0ugtuujpgspr0kwuwww)#76as58ns",
              "hex": "001432524d813b5ef0640b76ffe0fe217ce48288046f",
              "address": "tb1qxffymqfmtmcxgzmklls0ugtuujpgspr0kwuwww",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 29,
            "scriptPubKey": {
              "asm": "1 9693776751e9feac0b8c3a45cc8f809bd6cb99ea451bffb07b89d33a9cbaea3e",
              "desc": "rawtr(9693776751e9feac0b8c3a45cc8f809bd6cb99ea451bffb07b89d33a9cbaea3e)#qqy6zhfa",
              "hex": "51209693776751e9feac0b8c3a45cc8f809bd6cb99ea451bffb07b89d33a9cbaea3e",
              "address": "tb1pj6fhwe63a8l2czuv8fzueruqn0tvhx02g5dllvrm38fn4896aglqa4z25v",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 30,
            "scriptPubKey": {
              "asm": "1 bad1e73367beb579f5e485d033437850925df568264723651569dab874d61606",
              "desc": "rawtr(bad1e73367beb579f5e485d033437850925df568264723651569dab874d61606)#c0leygxg",
              "hex": "5120bad1e73367beb579f5e485d033437850925df568264723651569dab874d61606",
              "address": "tb1phtg7wvm8h66hna0yshgrxsmc2zf9matgyerjxeg4d8dtsaxkzcrqretg0x",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 31,
            "scriptPubKey": {
              "asm": "1 9a5ad97b12fb10032a778e371c8828a2343e18cf477b38c9e6bf8f0125d58e76",
              "desc": "rawtr(9a5ad97b12fb10032a778e371c8828a2343e18cf477b38c9e6bf8f0125d58e76)#ugnxt4fn",
              "hex": "51209a5ad97b12fb10032a778e371c8828a2343e18cf477b38c9e6bf8f0125d58e76",
              "address": "tb1pnfddj7cjlvgqx2nh3cm3ezpg5g6ruxx0gaan3j0xh78szfw43emqmjk9z2",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 32,
            "scriptPubKey": {
              "asm": "0 5dab736568f12e173a36171852446bd27b5bf087",
              "desc": "addr(tb1qtk4hxetg7yhpww3kzuv9y3rt6fa4huy8h299v7)#h2c5gndm",
              "hex": "00145dab736568f12e173a36171852446bd27b5bf087",
              "address": "tb1qtk4hxetg7yhpww3kzuv9y3rt6fa4huy8h299v7",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 33,
            "scriptPubKey": {
              "asm": "1 bf65d2a47cd348fe2fbe92f5037235dda2ea2559f5643035a22a896cafe3f4f3",
              "desc": "rawtr(bf65d2a47cd348fe2fbe92f5037235dda2ea2559f5643035a22a896cafe3f4f3)#jmk3v2p6",
              "hex": "5120bf65d2a47cd348fe2fbe92f5037235dda2ea2559f5643035a22a896cafe3f4f3",
              "address": "tb1phaja9fru6dy0uta7jt6sxu34mk3w5f2e74jrqddz92yketlr7nesjulhky",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 34,
            "scriptPubKey": {
              "asm": "0 b5f7c00b3019ef021a2c537130ab033041893350",
              "desc": "addr(tb1qkhmuqzesr8hsyx3v2dcnp2crxpqcjv6sdh2sns)#s2ctwdjm",
              "hex": "0014b5f7c00b3019ef021a2c537130ab033041893350",
              "address": "tb1qkhmuqzesr8hsyx3v2dcnp2crxpqcjv6sdh2sns",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 35,
            "scriptPubKey": {
              "asm": "0 b689bdc65dc4879285b487a71c04bb93bb85ac14",
              "desc": "addr(tb1qk6ymm3jacjre9pd5s7n3cp9mjwacttq57tpa3d)#497ydg6u",
              "hex": "0014b689bdc65dc4879285b487a71c04bb93bb85ac14",
              "address": "tb1qk6ymm3jacjre9pd5s7n3cp9mjwacttq57tpa3d",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 36,
            "scriptPubKey": {
              "asm": "0 231e4507eda75d10abac31fd3af93a5e3fe1bcc7",
              "desc": "addr(tb1qyv0y2pld5aw3p2avx87n47f6tcl7r0x8vu67rn)#pdf2cxuc",
              "hex": "0014231e4507eda75d10abac31fd3af93a5e3fe1bcc7",
              "address": "tb1qyv0y2pld5aw3p2avx87n47f6tcl7r0x8vu67rn",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 37,
            "scriptPubKey": {
              "asm": "1 065a634eec5b64f01126ce6682eb4dace50d5d35ee8085460396f1b12ddf5a34",
              "desc": "rawtr(065a634eec5b64f01126ce6682eb4dace50d5d35ee8085460396f1b12ddf5a34)#nn3kmtju",
              "hex": "5120065a634eec5b64f01126ce6682eb4dace50d5d35ee8085460396f1b12ddf5a34",
              "address": "tb1pqedxxnhvtdj0qyfxeeng966d4njs6hf4a6qg23srjmcmztwltg6qquvfke",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 38,
            "scriptPubKey": {
              "asm": "0 5d52bddaca6436ee7f6b29d1fabae7f59577624c",
              "desc": "addr(tb1qt4ftmkk2vsmwulmt98gl4wh87k2hwcjv4w8sue)#ptd3a5py",
              "hex": "00145d52bddaca6436ee7f6b29d1fabae7f59577624c",
              "address": "tb1qt4ftmkk2vsmwulmt98gl4wh87k2hwcjv4w8sue",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 39,
            "scriptPubKey": {
              "asm": "1 09a70d381f5f68bbf520447f5316c001df4511ca5243fcef7f7b987266aae4ac",
              "desc": "rawtr(09a70d381f5f68bbf520447f5316c001df4511ca5243fcef7f7b987266aae4ac)#macarg9c",
              "hex": "512009a70d381f5f68bbf520447f5316c001df4511ca5243fcef7f7b987266aae4ac",
              "address": "tb1ppxns6wqlta5thafqg3l4x9kqq8052yw22fplemml0wv8ye42ujkq7j2fxc",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 40,
            "scriptPubKey": {
              "asm": "1 8c43ccb26016cf0da7bd1f92dbaf6b353fe4d2077dae63020f329b9042357248",
              "desc": "rawtr(8c43ccb26016cf0da7bd1f92dbaf6b353fe4d2077dae63020f329b9042357248)#e94f8433",
              "hex": "51208c43ccb26016cf0da7bd1f92dbaf6b353fe4d2077dae63020f329b9042357248",
              "address": "tb1p33puevnqzm8smfaar7fdhtmtx5l7f5s80khxxqs0x2deqs34wfyqz69x6j",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 41,
            "scriptPubKey": {
              "asm": "1 0c5d9251b4078f2b027a371e1f083030afe87b0d9e1e0cb9a7695bcc9cbb1c5b",
              "desc": "rawtr(0c5d9251b4078f2b027a371e1f083030afe87b0d9e1e0cb9a7695bcc9cbb1c5b)#jczqts6j",
              "hex": "51200c5d9251b4078f2b027a371e1f083030afe87b0d9e1e0cb9a7695bcc9cbb1c5b",
              "address": "tb1pp3wey5d5q78jkqn6xu0p7zpsxzh7s7cdnc0qewd8d9due89mr3dstuy5km",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 42,
            "scriptPubKey": {
              "asm": "1 7ed5f1ddf5aa10808c7d4574f803a768df5fa7ed62dd10afa87264de0c9866a2",
              "desc": "rawtr(7ed5f1ddf5aa10808c7d4574f803a768df5fa7ed62dd10afa87264de0c9866a2)#qxjszp4j",
              "hex": "51207ed5f1ddf5aa10808c7d4574f803a768df5fa7ed62dd10afa87264de0c9866a2",
              "address": "tb1p0m2lrh044gggprrag460sqa8dr04lfldvtw3ptagwfjdurycv63q4n794w",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 43,
            "scriptPubKey": {
              "asm": "0 917d1b7f4b9fdaf38c32690eb21e81258636613d",
              "desc": "addr(tb1qj973kl6tnld08rpjdy8ty85pykrrvcfa0lx4un)#fm7x6463",
              "hex": "0014917d1b7f4b9fdaf38c32690eb21e81258636613d",
              "address": "tb1qj973kl6tnld08rpjdy8ty85pykrrvcfa0lx4un",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 44,
            "scriptPubKey": {
              "asm": "1 fce3007a0e51bb1078e2b36f51d985c789ac7455208f9ee306e09d8d5a40167d",
              "desc": "rawtr(fce3007a0e51bb1078e2b36f51d985c789ac7455208f9ee306e09d8d5a40167d)#fdluw5aj",
              "hex": "5120fce3007a0e51bb1078e2b36f51d985c789ac7455208f9ee306e09d8d5a40167d",
              "address": "tb1pln3sq7sw2xa3q78zkdh4rkv9c7y6caz4yz8eaccxuzwc6kjqze7sljtvv8",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 45,
            "scriptPubKey": {
              "asm": "0 094167dda649cd469f8f552f4116304896a5a17c",
              "desc": "addr(tb1qp9qk0hdxf8x5d8u025h5z93sfzt2tgtudtk0y9)#7xnm0cy3",
              "hex": "0014094167dda649cd469f8f552f4116304896a5a17c",
              "address": "tb1qp9qk0hdxf8x5d8u025h5z93sfzt2tgtudtk0y9",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 46,
            "scriptPubKey": {
              "asm": "0 86544590c80ec1aec9840f427f13efdc52a9eef2",
              "desc": "addr(tb1qse2ytyxgpmq6ajvypap87yl0m3f2nmhj4myegc)#9y3nqxln",
              "hex": "001486544590c80ec1aec9840f427f13efdc52a9eef2",
              "address": "tb1qse2ytyxgpmq6ajvypap87yl0m3f2nmhj4myegc",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 47,
            "scriptPubKey": {
              "asm": "1 40a193e16713dcd8765699c42b33891818aa4b7d9483f7f0b4be9116fb126f40",
              "desc": "rawtr(40a193e16713dcd8765699c42b33891818aa4b7d9483f7f0b4be9116fb126f40)#66t7jrvw",
              "hex": "512040a193e16713dcd8765699c42b33891818aa4b7d9483f7f0b4be9116fb126f40",
              "address": "tb1pgzse8ct8z0wdsajkn8zzkvufrqv25jmajjpl0u95h6g3d7cjdaqq8w4vvq",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 48,
            "scriptPubKey": {
              "asm": "0 03c07ddf9d8d296cf5015865dae1a086fe4fa98c",
              "desc": "addr(tb1qq0q8mhua355keagptpja4cdqsmlyl2vvqlkp33)#csj3v4w6",
              "hex": "001403c07ddf9d8d296cf5015865dae1a086fe4fa98c",
              "address": "tb1qq0q8mhua355keagptpja4cdqsmlyl2vvqlkp33",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 49,
            "scriptPubKey": {
              "asm": "1 c716194abfbd48aa18786db839863c823c2aaba0a6e46a6c11f32de4cc102c3c",
              "desc": "rawtr(c716194abfbd48aa18786db839863c823c2aaba0a6e46a6c11f32de4cc102c3c)#v00dkun5",
              "hex": "5120c716194abfbd48aa18786db839863c823c2aaba0a6e46a6c11f32de4cc102c3c",
              "address": "tb1pcutpjj4lh4y25xrcdkurnp3usg7z42aq5mjx5mq37vk7fnqs9s7qrdd2jk",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 50,
            "scriptPubKey": {
              "asm": "0 34857cf0a8d0903524ac378923b699b38c69428c",
              "desc": "addr(tb1qxjzheu9g6zgr2f9vx7yj8d5ekwxxjs5vcnx8w0)#hkj346m3",
              "hex": "001434857cf0a8d0903524ac378923b699b38c69428c",
              "address": "tb1qxjzheu9g6zgr2f9vx7yj8d5ekwxxjs5vcnx8w0",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 51,
            "scriptPubKey": {
              "asm": "0 69ab66c6a1374e1bdfbf6bff28587b6f62534ea2",
              "desc": "addr(tb1qdx4kd34pxa8phhald0ljskrmda39xn4zkl09z5)#dnn3nt6e",
              "hex": "001469ab66c6a1374e1bdfbf6bff28587b6f62534ea2",
              "address": "tb1qdx4kd34pxa8phhald0ljskrmda39xn4zkl09z5",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 52,
            "scriptPubKey": {
              "asm": "0 fa5d37901786bbecdb424cb7f8b901c0cce22cdf",
              "desc": "addr(tb1qlfwn0yqhs6a7ek6zfjml3wgpcrxwytxl7hd500)#vsp8jjvf",
              "hex": "0014fa5d37901786bbecdb424cb7f8b901c0cce22cdf",
              "address": "tb1qlfwn0yqhs6a7ek6zfjml3wgpcrxwytxl7hd500",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 53,
            "scriptPubKey": {
              "asm": "1 a1ffe8d7e0f369cbf956fd516c5cc7241be728fe6fb669a2b9a630de12b15442",
              "desc": "rawtr(a1ffe8d7e0f369cbf956fd516c5cc7241be728fe6fb669a2b9a630de12b15442)#gxzdgvc5",
              "hex": "5120a1ffe8d7e0f369cbf956fd516c5cc7241be728fe6fb669a2b9a630de12b15442",
              "address": "tb1p58l734lq7d5uh72kl4gkchx8ysd7w287d7mxng4e5ccduy4323pqg0xe38",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 54,
            "scriptPubKey": {
              "asm": "1 487b9a4b6ccdb9cbc1e87a5bfb90aab6548bd6c5ef959d9d0f57b9dd97388908",
              "desc": "rawtr(487b9a4b6ccdb9cbc1e87a5bfb90aab6548bd6c5ef959d9d0f57b9dd97388908)#gdk342rc",
              "hex": "5120487b9a4b6ccdb9cbc1e87a5bfb90aab6548bd6c5ef959d9d0f57b9dd97388908",
              "address": "tb1pfpae5jmvekuuhs0g0fdlhy92ke2gh4k9a72em8g027uam9ec3yyq25x83r",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 55,
            "scriptPubKey": {
              "asm": "0 caf9e9464b29d0518960ef91edf0c7ea0913f452",
              "desc": "addr(tb1qetu7j3jt98g9rztqa7g7mux8agy38azjkvtqef)#4alh58p8",
              "hex": "0014caf9e9464b29d0518960ef91edf0c7ea0913f452",
              "address": "tb1qetu7j3jt98g9rztqa7g7mux8agy38azjkvtqef",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 56,
            "scriptPubKey": {
              "asm": "1 98d52226debc775b60c37b762178560cdb2f2885c0cf4da9b1dca0f9ecebb8f1",
              "desc": "rawtr(98d52226debc775b60c37b762178560cdb2f2885c0cf4da9b1dca0f9ecebb8f1)#n5uq74kc",
              "hex": "512098d52226debc775b60c37b762178560cdb2f2885c0cf4da9b1dca0f9ecebb8f1",
              "address": "tb1pnr2jyfk7h3m4kcxr0dmzz7zkpndj72y9cr85m2d3mjs0nm8thrcsavmgz9",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 57,
            "scriptPubKey": {
              "asm": "1 0156eeb8bf2fc72c9d0a51d61dbc9ea89b9d29a045bf4e7ddd585ea2f4713ce5",
              "desc": "rawtr(0156eeb8bf2fc72c9d0a51d61dbc9ea89b9d29a045bf4e7ddd585ea2f4713ce5)#t735d8px",
              "hex": "51200156eeb8bf2fc72c9d0a51d61dbc9ea89b9d29a045bf4e7ddd585ea2f4713ce5",
              "address": "tb1pq9twaw9l9lrje8g228tpm0y74zde62dqgkl5ulwatp029ar38njs0lgnrc",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 58,
            "scriptPubKey": {
              "asm": "0 87047589aa035febcc8dd30736486bbbf69f6f05",
              "desc": "addr(tb1qsuz8tzd2qd07hnyd6vrnvjrth0mf7mc9np3qxv)#9jykwcj0",
              "hex": "001487047589aa035febcc8dd30736486bbbf69f6f05",
              "address": "tb1qsuz8tzd2qd07hnyd6vrnvjrth0mf7mc9np3qxv",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 59,
            "scriptPubKey": {
              "asm": "1 11c9cc957ea6e46b9ddffec877c1e83e7b7b5f86bb929744bde602b78b4139c7",
              "desc": "rawtr(11c9cc957ea6e46b9ddffec877c1e83e7b7b5f86bb929744bde602b78b4139c7)#f6f4x3g6",
              "hex": "512011c9cc957ea6e46b9ddffec877c1e83e7b7b5f86bb929744bde602b78b4139c7",
              "address": "tb1pz8yue9t75mjxh8wllmy80s0g8eahkhuxhwffw39aucpt0z6p88rs06uqfm",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 60,
            "scriptPubKey": {
              "asm": "0 4becdfda6fe04d4c8a06e356046672fdc1e2ba6c",
              "desc": "addr(tb1qf0kdlkn0upx5ezsxudtqgenjlhq79wnv4rwdfe)#w3awczvd",
              "hex": "00144becdfda6fe04d4c8a06e356046672fdc1e2ba6c",
              "address": "tb1qf0kdlkn0upx5ezsxudtqgenjlhq79wnv4rwdfe",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 61,
            "scriptPubKey": {
              "asm": "0 5aedf7ea48a8200e1f50d90cc7cdfe44f940c7e4",
              "desc": "addr(tb1qttkl06jg4qsqu86smyxv0n07gnu5p3lyyfsfzg)#uj5e3klc",
              "hex": "00145aedf7ea48a8200e1f50d90cc7cdfe44f940c7e4",
              "address": "tb1qttkl06jg4qsqu86smyxv0n07gnu5p3lyyfsfzg",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 62,
            "scriptPubKey": {
              "asm": "1 06fc8a4f72bea101214dbe3426b1542801fdbb067fc1f6b57da4ada88dcbbeb1",
              "desc": "rawtr(06fc8a4f72bea101214dbe3426b1542801fdbb067fc1f6b57da4ada88dcbbeb1)#a0n9uqme",
              "hex": "512006fc8a4f72bea101214dbe3426b1542801fdbb067fc1f6b57da4ada88dcbbeb1",
              "address": "tb1pqm7g5nmjh6sszg2dhc6zdv259qqlmwcx0lqlddta5jk63rwth6cs62nzg6",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 63,
            "scriptPubKey": {
              "asm": "0 24ed05ec42868a97100bb57d5ad452cd2d1399df",
              "desc": "addr(tb1qynkstmzzs69fwyqtk47444zje5k38xwls2y2qp)#kn7xng3m",
              "hex": "001424ed05ec42868a97100bb57d5ad452cd2d1399df",
              "address": "tb1qynkstmzzs69fwyqtk47444zje5k38xwls2y2qp",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 64,
            "scriptPubKey": {
              "asm": "0 6767c9d7f5562486d59c082d055570ff1ffe4110",
              "desc": "addr(tb1qvanun4l42cjgd4vupqks24tslu0lusgscq2hnv)#6c49ufrl",
              "hex": "00146767c9d7f5562486d59c082d055570ff1ffe4110",
              "address": "tb1qvanun4l42cjgd4vupqks24tslu0lusgscq2hnv",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 65,
            "scriptPubKey": {
              "asm": "1 c13e909c21798494f6167f7a348cdca35205ef9bb17805325fd6673e908e2f4b",
              "desc": "rawtr(c13e909c21798494f6167f7a348cdca35205ef9bb17805325fd6673e908e2f4b)#8prqh44g",
              "hex": "5120c13e909c21798494f6167f7a348cdca35205ef9bb17805325fd6673e908e2f4b",
              "address": "tb1pcylfp8pp0xzffask0aarfrxu5dfqtmumk9uq2vjl6ennayyw9a9ss92hy6",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 66,
            "scriptPubKey": {
              "asm": "1 b8653d2f22b1d30970e0b281f0b740647e9afd20ad7fa6855b195aa102594be4",
              "desc": "rawtr(b8653d2f22b1d30970e0b281f0b740647e9afd20ad7fa6855b195aa102594be4)#l7px7s5f",
              "hex": "5120b8653d2f22b1d30970e0b281f0b740647e9afd20ad7fa6855b195aa102594be4",
              "address": "tb1phpjn6tezk8fsju8qk2qlpd6qv3lf4lfq44l6dp2mr9d2zqjef0jqehugxm",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 67,
            "scriptPubKey": {
              "asm": "0 812796319cbb8c73fabd038ace40dbb4a560e132",
              "desc": "addr(tb1qsynevvvuhwx8874aqw9vusxmkjjkpcfj88mz3w)#whmejkvl",
              "hex": "0014812796319cbb8c73fabd038ace40dbb4a560e132",
              "address": "tb1qsynevvvuhwx8874aqw9vusxmkjjkpcfj88mz3w",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 68,
            "scriptPubKey": {
              "asm": "1 afe7ee1932ef5ed18bf299c19d3324b0ce6c4ce53bf52724b234a5b40d3cd378",
              "desc": "rawtr(afe7ee1932ef5ed18bf299c19d3324b0ce6c4ce53bf52724b234a5b40d3cd378)#75vns67d",
              "hex": "5120afe7ee1932ef5ed18bf299c19d3324b0ce6c4ce53bf52724b234a5b40d3cd378",
              "address": "tb1p4ln7uxfjaa0drzljn8qe6veykr8xcn89806jwf9jxjjmgrfu6duq2hdw4z",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 69,
            "scriptPubKey": {
              "asm": "0 f6b91767aa08662fa6e375f9da2401863b586047",
              "desc": "addr(tb1q76u3wea2ppnzlfhrwhua5fqpsca4scz8k4lv2r)#wekuwrau",
              "hex": "0014f6b91767aa08662fa6e375f9da2401863b586047",
              "address": "tb1q76u3wea2ppnzlfhrwhua5fqpsca4scz8k4lv2r",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 70,
            "scriptPubKey": {
              "asm": "0 0f77d4e4988515215b72cd85d09c9a710df85964",
              "desc": "addr(tb1qpamafeycs52jzkmjekzap8y6wyxlsktys7qa7p)#acwc9ce2",
              "hex": "00140f77d4e4988515215b72cd85d09c9a710df85964",
              "address": "tb1qpamafeycs52jzkmjekzap8y6wyxlsktys7qa7p",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 71,
            "scriptPubKey": {
              "asm": "0 4664d9d2b9d192bd90282d8767ef8d60669fda74",
              "desc": "addr(tb1qgejdn54e6xftmypg9krk0mudvpnflkn5pkhwyd)#hwld67kf",
              "hex": "00144664d9d2b9d192bd90282d8767ef8d60669fda74",
              "address": "tb1qgejdn54e6xftmypg9krk0mudvpnflkn5pkhwyd",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 72,
            "scriptPubKey": {
              "asm": "0 68fe1794c70b5b48366660062ef73d25b6f90ed2",
              "desc": "addr(tb1qdrlp09x8pdd5sdnxvqrzaaeaykm0jrkjz75cn2)#r7dctt3r",
              "hex": "001468fe1794c70b5b48366660062ef73d25b6f90ed2",
              "address": "tb1qdrlp09x8pdd5sdnxvqrzaaeaykm0jrkjz75cn2",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 73,
            "scriptPubKey": {
              "asm": "0 276b37212ffbeea8cc920dfeddb3e521824a14a8",
              "desc": "addr(tb1qya4nwgf0l0h23nyjphldmvl9yxpy599gyxcyuh)#fw57dc33",
              "hex": "0014276b37212ffbeea8cc920dfeddb3e521824a14a8",
              "address": "tb1qya4nwgf0l0h23nyjphldmvl9yxpy599gyxcyuh",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 74,
            "scriptPubKey": {
              "asm": "0 57f77327b3bca7e64952e767cfdb971c1700303b",
              "desc": "addr(tb1q2lmhxfanhjn7vj2juanulkuhrstsqvpmegcf5c)#cspd3y50",
              "hex": "001457f77327b3bca7e64952e767cfdb971c1700303b",
              "address": "tb1q2lmhxfanhjn7vj2juanulkuhrstsqvpmegcf5c",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 75,
            "scriptPubKey": {
              "asm": "0 e614787d373d3480ace5f85b3b6f38f1d5ffdecb",
              "desc": "addr(tb1quc28slfh856gpt89lpdnkmec782llhktr2fx83)#fxxv57n0",
              "hex": "0014e614787d373d3480ace5f85b3b6f38f1d5ffdecb",
              "address": "tb1quc28slfh856gpt89lpdnkmec782llhktr2fx83",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 76,
            "scriptPubKey": {
              "asm": "1 35e00ba751b0ca2d9ebdfc4569c3f0d561c0a145273a24a3b284f104d0d6db42",
              "desc": "rawtr(35e00ba751b0ca2d9ebdfc4569c3f0d561c0a145273a24a3b284f104d0d6db42)#z2dcqkvz",
              "hex": "512035e00ba751b0ca2d9ebdfc4569c3f0d561c0a145273a24a3b284f104d0d6db42",
              "address": "tb1pxhsqhf63kr9zm84al3zknsls64supg29yuazfgajsncsf5xkmdpqhp7u0q",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 77,
            "scriptPubKey": {
              "asm": "1 1d7b6e6bc1e73c75c9aba8a9935f7487672695c93f671fee8f3fd4c6c46f10b9",
              "desc": "rawtr(1d7b6e6bc1e73c75c9aba8a9935f7487672695c93f671fee8f3fd4c6c46f10b9)#45plhsvk",
              "hex": "51201d7b6e6bc1e73c75c9aba8a9935f7487672695c93f671fee8f3fd4c6c46f10b9",
              "address": "tb1pr4aku67puu78tjdt4z5exhm5sanjd9wf8an3lm508l2vd3r0zzuscnt4rz",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 78,
            "scriptPubKey": {
              "asm": "0 f405045ef59204b7c828e187972edb9e1ea64cf5",
              "desc": "addr(tb1q7szsghh4jgzt0jpguxrewtkmnc02vn845a92jx)#rc4hdn47",
              "hex": "0014f405045ef59204b7c828e187972edb9e1ea64cf5",
              "address": "tb1q7szsghh4jgzt0jpguxrewtkmnc02vn845a92jx",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 79,
            "scriptPubKey": {
              "asm": "1 bdfa80719516d0b0bffd08f804b10e7958b917655741e626015ef398f57959a2",
              "desc": "rawtr(bdfa80719516d0b0bffd08f804b10e7958b917655741e626015ef398f57959a2)#fpa60lht",
              "hex": "5120bdfa80719516d0b0bffd08f804b10e7958b917655741e626015ef398f57959a2",
              "address": "tb1phhagquv4zmgtp0lapruqfvgw09vtj9m92aq7vfsptmee3atetx3quf5880",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 80,
            "scriptPubKey": {
              "asm": "0 65518b67b59000767cc04a262ae10b6daa9caf09",
              "desc": "addr(tb1qv4gckea4jqq8vlxqfgnz4cgtdk4fetcfhw3r8s)#lfqhj9yl",
              "hex": "001465518b67b59000767cc04a262ae10b6daa9caf09",
              "address": "tb1qv4gckea4jqq8vlxqfgnz4cgtdk4fetcfhw3r8s",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 81,
            "scriptPubKey": {
              "asm": "1 e181cc774a6de16ca2bf1f816817581ff61a4812b3b02dfd70a992a3f0e397ea",
              "desc": "rawtr(e181cc774a6de16ca2bf1f816817581ff61a4812b3b02dfd70a992a3f0e397ea)#t2w03ehk",
              "hex": "5120e181cc774a6de16ca2bf1f816817581ff61a4812b3b02dfd70a992a3f0e397ea",
              "address": "tb1puxquca62dhskeg4lr7qks96crlmp5jqjkwczmlts4xf28u8rjl4qpqtlem",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 82,
            "scriptPubKey": {
              "asm": "1 110ceaa9a80b3f8c55308ceef473481c05a7151eb57c0205835ae836de79d9f2",
              "desc": "rawtr(110ceaa9a80b3f8c55308ceef473481c05a7151eb57c0205835ae836de79d9f2)#n5sgqfs9",
              "hex": "5120110ceaa9a80b3f8c55308ceef473481c05a7151eb57c0205835ae836de79d9f2",
              "address": "tb1pzyxw42dgpvlcc4fs3nh0gu6grsz6w9g7k47qypvrtt5rdhnem8eqxj0q3x",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 83,
            "scriptPubKey": {
              "asm": "1 7d232f84d3282c1bf70ecb3d2e2009e58aa3627f88a7f4c3961e6d01a18139fa",
              "desc": "rawtr(7d232f84d3282c1bf70ecb3d2e2009e58aa3627f88a7f4c3961e6d01a18139fa)#vz076aer",
              "hex": "51207d232f84d3282c1bf70ecb3d2e2009e58aa3627f88a7f4c3961e6d01a18139fa",
              "address": "tb1p053jlpxn9qkphacwev7jugqfuk92xcnl3znlfsukreksrgvp88aqsupc6r",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 84,
            "scriptPubKey": {
              "asm": "1 8d758a6e83f4c71de1642e0f231d4c341a76c63dbb91e6ed485421a0d6bc9ab6",
              "desc": "rawtr(8d758a6e83f4c71de1642e0f231d4c341a76c63dbb91e6ed485421a0d6bc9ab6)#rgvd29q2",
              "hex": "51208d758a6e83f4c71de1642e0f231d4c341a76c63dbb91e6ed485421a0d6bc9ab6",
              "address": "tb1p346c5m5r7nr3mcty9c8jx82vxsd8d33ahwg7dm2g2ss6p44un2mql2cv59",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 85,
            "scriptPubKey": {
              "asm": "0 f4c6fd3890b39f31a1b8d290f1e3d7f17f6d9224",
              "desc": "addr(tb1q7nr06wyskw0nrgdc62g0rc7h79lkmy3ylgtyen)#h3vyt3lc",
              "hex": "0014f4c6fd3890b39f31a1b8d290f1e3d7f17f6d9224",
              "address": "tb1q7nr06wyskw0nrgdc62g0rc7h79lkmy3ylgtyen",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 86,
            "scriptPubKey": {
              "asm": "1 524848cfde02027012221fda20dd1dbfc7b025a0506490ac461d5ea0899fbd13",
              "desc": "rawtr(524848cfde02027012221fda20dd1dbfc7b025a0506490ac461d5ea0899fbd13)#chs6a3n3",
              "hex": "5120524848cfde02027012221fda20dd1dbfc7b025a0506490ac461d5ea0899fbd13",
              "address": "tb1p2fyy3n77qgp8qy3zrldzphgahlrmqfdq2pjfptzxr402pzvlh5fsulap62",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 87,
            "scriptPubKey": {
              "asm": "0 5a8527c0a08f07a3b6d3fed75570c7e8ece73338",
              "desc": "addr(tb1qt2zj0s9q3ur68dknlmt42ux8arkwwvechfd89c)#ckwly3pe",
              "hex": "00145a8527c0a08f07a3b6d3fed75570c7e8ece73338",
              "address": "tb1qt2zj0s9q3ur68dknlmt42ux8arkwwvechfd89c",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 88,
            "scriptPubKey": {
              "asm": "1 255ed1e617ccf4c1c4420e018eb3c9e306922c0240f671860ec6c02d8ec034f8",
              "desc": "rawtr(255ed1e617ccf4c1c4420e018eb3c9e306922c0240f671860ec6c02d8ec034f8)#ynsxrp3w",
              "hex": "5120255ed1e617ccf4c1c4420e018eb3c9e306922c0240f671860ec6c02d8ec034f8",
              "address": "tb1py40dreshen6vr3zzpcqcav7fuvrfytqzgrm8rpswcmqzmrkqxnuq7rw600",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 89,
            "scriptPubKey": {
              "asm": "0 91353994e4777fa9867805023959c5132b72bc1a",
              "desc": "addr(tb1qjy6nn98ywal6npncq5prjkw9zv4h90q6lsewkw)#q0ts6wr7",
              "hex": "001491353994e4777fa9867805023959c5132b72bc1a",
              "address": "tb1qjy6nn98ywal6npncq5prjkw9zv4h90q6lsewkw",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 90,
            "scriptPubKey": {
              "asm": "0 f8991111950543c01ecbb106238f5ef29c32ccfc",
              "desc": "addr(tb1qlzv3zyv4q4puq8ktkyrz8r6772wr9n8uccldw7)#zhz2fnzn",
              "hex": "0014f8991111950543c01ecbb106238f5ef29c32ccfc",
              "address": "tb1qlzv3zyv4q4puq8ktkyrz8r6772wr9n8uccldw7",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 91,
            "scriptPubKey": {
              "asm": "0 a3b2e051c0c5405276fe91833e0f06563bfc0e49",
              "desc": "addr(tb1q5wewq5wqc4q9yah7jxpnurcx2calcrjfx8lmmt)#pk6qt5ek",
              "hex": "0014a3b2e051c0c5405276fe91833e0f06563bfc0e49",
              "address": "tb1q5wewq5wqc4q9yah7jxpnurcx2calcrjfx8lmmt",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 92,
            "scriptPubKey": {
              "asm": "0 3e554ff09664b4e5b29864c25e28b4e7a38e28f9",
              "desc": "addr(tb1q8e25luykvj6wtv5cvnp9u295u73cu28e59jgpu)#wpc3d9l9",
              "hex": "00143e554ff09664b4e5b29864c25e28b4e7a38e28f9",
              "address": "tb1q8e25luykvj6wtv5cvnp9u295u73cu28e59jgpu",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 93,
            "scriptPubKey": {
              "asm": "1 e60085afd0289f5664d294e4df68a930b07ffb84a5d579c32414d174c4b2f37c",
              "desc": "rawtr(e60085afd0289f5664d294e4df68a930b07ffb84a5d579c32414d174c4b2f37c)#sdqaglp0",
              "hex": "5120e60085afd0289f5664d294e4df68a930b07ffb84a5d579c32414d174c4b2f37c",
              "address": "tb1pucqgtt7s9z04vexjjnjd769fxzc8l7uy5h2hnseyznghf39j7d7qtwg9c5",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 94,
            "scriptPubKey": {
              "asm": "1 1f2e7dbb6780a7bba4c8cc235843d6396715f141cbfc8e42c597aa14bd4a383a",
              "desc": "rawtr(1f2e7dbb6780a7bba4c8cc235843d6396715f141cbfc8e42c597aa14bd4a383a)#25mhtx2p",
              "hex": "51201f2e7dbb6780a7bba4c8cc235843d6396715f141cbfc8e42c597aa14bd4a383a",
              "address": "tb1pruh8mwm8sznmhfxges34ss7k89n3tu2pe07gusk9j74pf0228qaq0m2nc4",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 95,
            "scriptPubKey": {
              "asm": "0 d4f6bebdc529d590f14b8061a86d9eae5e92d0fd",
              "desc": "addr(tb1q6nmta0w9982epu2tsps6smv74e0f958agnlm5t)#y5mdzpp0",
              "hex": "0014d4f6bebdc529d590f14b8061a86d9eae5e92d0fd",
              "address": "tb1q6nmta0w9982epu2tsps6smv74e0f958agnlm5t",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 96,
            "scriptPubKey": {
              "asm": "0 f539a661c27ed95fa55c6b7e76ceb9fe42900f47",
              "desc": "addr(tb1q75u6vcwz0mv4lf2uddl8dn4elepfqr68z34jfw)#kqj8x4mu",
              "hex": "0014f539a661c27ed95fa55c6b7e76ceb9fe42900f47",
              "address": "tb1q75u6vcwz0mv4lf2uddl8dn4elepfqr68z34jfw",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 97,
            "scriptPubKey": {
              "asm": "0 8a5f63d2bba2f8b2e6b80b4c80e41cfd210a02d7",
              "desc": "addr(tb1q3f0k854m5tut9e4cpdxgpequl5ss5qkhrk4cfc)#ft6sgtgf",
              "hex": "00148a5f63d2bba2f8b2e6b80b4c80e41cfd210a02d7",
              "address": "tb1q3f0k854m5tut9e4cpdxgpequl5ss5qkhrk4cfc",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 98,
            "scriptPubKey": {
              "asm": "0 c54bfe33a04606998496ac258fa9d8181b084e5b",
              "desc": "addr(tb1qc49luvaqgcrfnpyk4sjcl2wcrqdssnjmx952re)#c4ncqzyc",
              "hex": "0014c54bfe33a04606998496ac258fa9d8181b084e5b",
              "address": "tb1qc49luvaqgcrfnpyk4sjcl2wcrqdssnjmx952re",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 99,
            "scriptPubKey": {
              "asm": "1 9fe0e22abdbe25a86f47758ea1cd00850979f0f3a8609f337cb01b8627658eab",
              "desc": "rawtr(9fe0e22abdbe25a86f47758ea1cd00850979f0f3a8609f337cb01b8627658eab)#wk54pxnh",
              "hex": "51209fe0e22abdbe25a86f47758ea1cd00850979f0f3a8609f337cb01b8627658eab",
              "address": "tb1pnlswy24ahcj6sm68wk82rngqs5yhnu8n4psf7vmukqdcvfm9364s523wg5",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 100,
            "scriptPubKey": {
              "asm": "0 a97e952e90f9d62f1428b5f871b46b89d5f1faff",
              "desc": "addr(tb1q49lf2t5sl8tz79pgkhu8rdrt382lr7hlh2d680)#cpyjx49z",
              "hex": "0014a97e952e90f9d62f1428b5f871b46b89d5f1faff",
              "address": "tb1q49lf2t5sl8tz79pgkhu8rdrt382lr7hlh2d680",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 101,
            "scriptPubKey": {
              "asm": "1 a6f2853cef429f5d47da0fac527078ad8befdd2d3108d6e04556e1d2de385840",
              "desc": "rawtr(a6f2853cef429f5d47da0fac527078ad8befdd2d3108d6e04556e1d2de385840)#rmaka0ag",
              "hex": "5120a6f2853cef429f5d47da0fac527078ad8befdd2d3108d6e04556e1d2de385840",
              "address": "tb1p5meg2080g2046376p7k9yurc4k97lhfdxyyddcz92msa9h3ctpqq9uh240",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 102,
            "scriptPubKey": {
              "asm": "0 8ead8c73ae5e7db441d3a1bfe9f64b44dbe3337e",
              "desc": "addr(tb1q36kccuawte7mgswn5xl7najtgnd7xvm7xpkrv7)#kunmv6p5",
              "hex": "00148ead8c73ae5e7db441d3a1bfe9f64b44dbe3337e",
              "address": "tb1q36kccuawte7mgswn5xl7najtgnd7xvm7xpkrv7",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 103,
            "scriptPubKey": {
              "asm": "0 3d9997ed9d5041945accafbfa03810e852c93c28",
              "desc": "addr(tb1q8kve0mva2pqegkkv47l6qwqsapfvj0pgl8dmeq)#zhv9z9rv",
              "hex": "00143d9997ed9d5041945accafbfa03810e852c93c28",
              "address": "tb1q8kve0mva2pqegkkv47l6qwqsapfvj0pgl8dmeq",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 104,
            "scriptPubKey": {
              "asm": "0 8cb6a7b97f301eba3521bd11debd8746b9f77266",
              "desc": "addr(tb1q3jm20wtlxq0t5dfph5gaa0v8g6ulwunxl9x4pm)#9px0mdlp",
              "hex": "00148cb6a7b97f301eba3521bd11debd8746b9f77266",
              "address": "tb1q3jm20wtlxq0t5dfph5gaa0v8g6ulwunxl9x4pm",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 105,
            "scriptPubKey": {
              "asm": "0 8a2ab926c3bba96867e20d021e0ce9ef402ca732",
              "desc": "addr(tb1q3g4tjfkrhw5kselzp5ppur8faaqzefej6eeyek)#pwz89edx",
              "hex": "00148a2ab926c3bba96867e20d021e0ce9ef402ca732",
              "address": "tb1q3g4tjfkrhw5kselzp5ppur8faaqzefej6eeyek",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 106,
            "scriptPubKey": {
              "asm": "1 7112a5301dc8504446af011181a724a46419dfd2594600824c33c1d79d1e312d",
              "desc": "rawtr(7112a5301dc8504446af011181a724a46419dfd2594600824c33c1d79d1e312d)#wvmn5fdy",
              "hex": "51207112a5301dc8504446af011181a724a46419dfd2594600824c33c1d79d1e312d",
              "address": "tb1pwyf22vqaepgyg340qygcrfey53jpnh7jt9rqpqjvx0qa08g7xyks9c7rdk",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 107,
            "scriptPubKey": {
              "asm": "1 ad01b571c7f7de2ce9d14f4f23eed33124a4f035fdfc1f09e755f0f732e72143",
              "desc": "rawtr(ad01b571c7f7de2ce9d14f4f23eed33124a4f035fdfc1f09e755f0f732e72143)#rwsusxck",
              "hex": "5120ad01b571c7f7de2ce9d14f4f23eed33124a4f035fdfc1f09e755f0f732e72143",
              "address": "tb1p45qm2uw87l0ze6w3fa8j8mknxyj2fup4lh7p7z082hc0wvh8y9psjdw3rp",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 108,
            "scriptPubKey": {
              "asm": "1 87a1452c7430b0bd510f168971e484d11ee8d356267adf15adb06833f9c39ff1",
              "desc": "rawtr(87a1452c7430b0bd510f168971e484d11ee8d356267adf15adb06833f9c39ff1)#ygvhxh36",
              "hex": "512087a1452c7430b0bd510f168971e484d11ee8d356267adf15adb06833f9c39ff1",
              "address": "tb1ps7s52tr5xzct65g0z6yhreyy6y0w356kyead79ddkp5r87wrnlcsccerh2",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 109,
            "scriptPubKey": {
              "asm": "1 608a982c549dc8fd7305ad09359dd6d68fc99dfc7c845397b49b53b5e2731c93",
              "desc": "rawtr(608a982c549dc8fd7305ad09359dd6d68fc99dfc7c845397b49b53b5e2731c93)#tvvtzuec",
              "hex": "5120608a982c549dc8fd7305ad09359dd6d68fc99dfc7c845397b49b53b5e2731c93",
              "address": "tb1pvz9fstz5nhy06uc945ynt8wk668un80u0jz989a5ndfmtcnnrjfs85x36w",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 110,
            "scriptPubKey": {
              "asm": "0 7eaccf8f20192ba51b7505f432f77676ef12ca33",
              "desc": "addr(tb1q06kvlreqry462xm4qh6r9amkwmh39j3nskqwqt)#43sd4d5p",
              "hex": "00147eaccf8f20192ba51b7505f432f77676ef12ca33",
              "address": "tb1q06kvlreqry462xm4qh6r9amkwmh39j3nskqwqt",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 111,
            "scriptPubKey": {
              "asm": "0 72edf05b63110aee33a85863762dd209398b1577",
              "desc": "addr(tb1qwtklqkmrzy9wuvagtp3hvtwjpyuck9thzefqpx)#d36cylvh",
              "hex": "001472edf05b63110aee33a85863762dd209398b1577",
              "address": "tb1qwtklqkmrzy9wuvagtp3hvtwjpyuck9thzefqpx",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 112,
            "scriptPubKey": {
              "asm": "0 c7cb01cd648083f9df30d28ca786b479347dc51a",
              "desc": "addr(tb1qcl9srntyszplnhes62x20p450y68m3g6d4veep)#vuxj2y9p",
              "hex": "0014c7cb01cd648083f9df30d28ca786b479347dc51a",
              "address": "tb1qcl9srntyszplnhes62x20p450y68m3g6d4veep",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 113,
            "scriptPubKey": {
              "asm": "1 ad310751da5175cc71f8b2443d359898ffc6741f5314d592165c83b52427306e",
              "desc": "rawtr(ad310751da5175cc71f8b2443d359898ffc6741f5314d592165c83b52427306e)#90ttvr2w",
              "hex": "5120ad310751da5175cc71f8b2443d359898ffc6741f5314d592165c83b52427306e",
              "address": "tb1p45csw5w6296ucu0ckfzr6dvcnrluvaql2v2dtysktjpm2fp8xphqhnxd70",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 114,
            "scriptPubKey": {
              "asm": "1 253db23f14f8473562a9b6c3b98b61962b62fbdb0c17032755a4c0fc4cd3b11a",
              "desc": "rawtr(253db23f14f8473562a9b6c3b98b61962b62fbdb0c17032755a4c0fc4cd3b11a)#m57ru9v8",
              "hex": "5120253db23f14f8473562a9b6c3b98b61962b62fbdb0c17032755a4c0fc4cd3b11a",
              "address": "tb1py57my0c5lprn2c4fkmpmnzmpjc4k977mpstsxf645nq0cnxnkydqjs9wsz",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 115,
            "scriptPubKey": {
              "asm": "1 4822a2bf9b9373d6feb925378b1d5d8cb410a0a1da6d7a2b04d0dc7f3c6aba4b",
              "desc": "rawtr(4822a2bf9b9373d6feb925378b1d5d8cb410a0a1da6d7a2b04d0dc7f3c6aba4b)#lvk8d7mx",
              "hex": "51204822a2bf9b9373d6feb925378b1d5d8cb410a0a1da6d7a2b04d0dc7f3c6aba4b",
              "address": "tb1pfq3290umjdeadl4ey5mck82a3j6ppg9pmfkh52cy6rw870r2hf9sxsh68a",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 116,
            "scriptPubKey": {
              "asm": "0 6b43c2932a3ee165dcef44355a43d5100c866ee8",
              "desc": "addr(tb1qddpu9ye28mskth80gs645s74zqxgvmhgzdrjzz)#rhekrt2y",
              "hex": "00146b43c2932a3ee165dcef44355a43d5100c866ee8",
              "address": "tb1qddpu9ye28mskth80gs645s74zqxgvmhgzdrjzz",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 117,
            "scriptPubKey": {
              "asm": "0 01ad816304a9775b5af02af49b243497104f67fd",
              "desc": "addr(tb1qqxkczccy49m4kkhs9t6fkfp5jugy7elavlm7uj)#sy3g6msy",
              "hex": "001401ad816304a9775b5af02af49b243497104f67fd",
              "address": "tb1qqxkczccy49m4kkhs9t6fkfp5jugy7elavlm7uj",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 118,
            "scriptPubKey": {
              "asm": "0 cca5a427f2b2f5c104c053877f1654edfc03f2cd",
              "desc": "addr(tb1qejj6gfljkt6uzpxq2wrh79j5ah7q8ukd3fhycu)#xzvsddqs",
              "hex": "0014cca5a427f2b2f5c104c053877f1654edfc03f2cd",
              "address": "tb1qejj6gfljkt6uzpxq2wrh79j5ah7q8ukd3fhycu",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 119,
            "scriptPubKey": {
              "asm": "0 e7b1f8aaf18624267f452ba89f62befb6a836823",
              "desc": "addr(tb1qu7cl32h3scjzvl699w5f7c47ld4gx6prhvedm9)#0mzee5lx",
              "hex": "0014e7b1f8aaf18624267f452ba89f62befb6a836823",
              "address": "tb1qu7cl32h3scjzvl699w5f7c47ld4gx6prhvedm9",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 120,
            "scriptPubKey": {
              "asm": "1 cf81df54ca395a0e2a2485da1af4085ca71bca1f7dde31e117b11687d63d1cfa",
              "desc": "rawtr(cf81df54ca395a0e2a2485da1af4085ca71bca1f7dde31e117b11687d63d1cfa)#z6rtu64z",
              "hex": "5120cf81df54ca395a0e2a2485da1af4085ca71bca1f7dde31e117b11687d63d1cfa",
              "address": "tb1pe7qa74x289dqu23yshdp4aqgtjn3hjsl0h0rrcghkytg043arnaq6eenxv",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 121,
            "scriptPubKey": {
              "asm": "0 2c958620b836492676b67087a773e8e8d63dc4f2",
              "desc": "addr(tb1q9j2cvg9cxeyjva4kwzr6wulgartrm38jyglmz4)#44vcqyem",
              "hex": "00142c958620b836492676b67087a773e8e8d63dc4f2",
              "address": "tb1q9j2cvg9cxeyjva4kwzr6wulgartrm38jyglmz4",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 122,
            "scriptPubKey": {
              "asm": "0 b1a561376b8b3fb347da7017b38c4bd439f6f0eb",
              "desc": "addr(tb1qkxjkzdmt3vlmx376wqtm8rzt6suldu8twyl7a7)#9pyq8t44",
              "hex": "0014b1a561376b8b3fb347da7017b38c4bd439f6f0eb",
              "address": "tb1qkxjkzdmt3vlmx376wqtm8rzt6suldu8twyl7a7",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 123,
            "scriptPubKey": {
              "asm": "0 2f037191dbbe002f668d571b10197216d4416743",
              "desc": "addr(tb1q9uphrywmhcqz7e5d2ud3qxtjzm2yze6rt2n3q9)#ux9tk9nk",
              "hex": "00142f037191dbbe002f668d571b10197216d4416743",
              "address": "tb1q9uphrywmhcqz7e5d2ud3qxtjzm2yze6rt2n3q9",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 124,
            "scriptPubKey": {
              "asm": "0 919df7a43f93eb322fd8ebcefae89e350a9deca9",
              "desc": "addr(tb1qjxwl0fplj04nyt7ca08046y7x59fmm9fumu75a)#4sauhd2w",
              "hex": "0014919df7a43f93eb322fd8ebcefae89e350a9deca9",
              "address": "tb1qjxwl0fplj04nyt7ca08046y7x59fmm9fumu75a",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 125,
            "scriptPubKey": {
              "asm": "0 610d1d6a5b0a7009297ac47f6d329f229e8a2f86",
              "desc": "addr(tb1qvyx366jmpfcqj2t6c3lk6v5ly20g5tuxhekac0)#z76m6fzy",
              "hex": "0014610d1d6a5b0a7009297ac47f6d329f229e8a2f86",
              "address": "tb1qvyx366jmpfcqj2t6c3lk6v5ly20g5tuxhekac0",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 126,
            "scriptPubKey": {
              "asm": "0 6dbb51d9e9fc6a292985eb2105d1916cb59224c7",
              "desc": "addr(tb1qdka4rk0fl34zj2v9avsst5v3dj6eyfx8zt3dyu)#ef3rrhdc",
              "hex": "00146dbb51d9e9fc6a292985eb2105d1916cb59224c7",
              "address": "tb1qdka4rk0fl34zj2v9avsst5v3dj6eyfx8zt3dyu",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 127,
            "scriptPubKey": {
              "asm": "1 2f28d117159fd4a6b2a21e5c3dfb5dfaeabf24aa20a3c6da2025bac6ef78edbf",
              "desc": "rawtr(2f28d117159fd4a6b2a21e5c3dfb5dfaeabf24aa20a3c6da2025bac6ef78edbf)#asknatjm",
              "hex": "51202f28d117159fd4a6b2a21e5c3dfb5dfaeabf24aa20a3c6da2025bac6ef78edbf",
              "address": "tb1p9u5dz9c4nl22dv4zrewrm76alt4t7f92yz3udk3qykavdmmcaklsu79jz0",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 128,
            "scriptPubKey": {
              "asm": "0 cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
              "desc": "addr(tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw)#7x8kgusx",
              "hex": "0020cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
              "address": "tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw",
              "type": "witness_v0_scripthash"
            }
          },
          {
            "value": 0.25540109,
            "n": 129,
            "scriptPubKey": {
              "asm": "1 708599406e122d2b5ffa7c351edbee0ff542df0c99b700202568b3daddd1130c",
              "desc": "rawtr(708599406e122d2b5ffa7c351edbee0ff542df0c99b700202568b3daddd1130c)#p4u6k5f5",
              "hex": "5120708599406e122d2b5ffa7c351edbee0ff542df0c99b700202568b3daddd1130c",
              "address": "tb1pwzzejsrwzgkjkhl60s63aklwpl659hcvnxmsqgp9dzea4hw3zvxqaj9au2",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 130,
            "scriptPubKey": {
              "asm": "1 554dfd648df03db69927e0f8ebba280e6ad073cbeca9bca9024b5b89857da737",
              "desc": "rawtr(554dfd648df03db69927e0f8ebba280e6ad073cbeca9bca9024b5b89857da737)#e2gpr2kc",
              "hex": "5120554dfd648df03db69927e0f8ebba280e6ad073cbeca9bca9024b5b89857da737",
              "address": "tb1p24xl6eyd7q7mdxf8uruwhw3gpe4dqu7taj5me2gzfddcnpta5umssdgnqe",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 131,
            "scriptPubKey": {
              "asm": "1 2d8f56c7c7d9256482fd0ca398ae29ab45e1e88db43d9d713a44f3ac0d5fc097",
              "desc": "rawtr(2d8f56c7c7d9256482fd0ca398ae29ab45e1e88db43d9d713a44f3ac0d5fc097)#6d2ulvtr",
              "hex": "51202d8f56c7c7d9256482fd0ca398ae29ab45e1e88db43d9d713a44f3ac0d5fc097",
              "address": "tb1p9k84d378myjkfqhapj3e3t3f4dz7r6ydks7e6uf6gne6cr2lczts0fyp7z",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 132,
            "scriptPubKey": {
              "asm": "1 208c3f528fc15edb0c9e13528400035db26999a9b97a12e30a2af0fa80392a31",
              "desc": "rawtr(208c3f528fc15edb0c9e13528400035db26999a9b97a12e30a2af0fa80392a31)#47lq2eme",
              "hex": "5120208c3f528fc15edb0c9e13528400035db26999a9b97a12e30a2af0fa80392a31",
              "address": "tb1pyzxr7550c90dkry7zdfggqqrtkexnxdfh9ap9cc29tc04qpe9gcsn2taet",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 133,
            "scriptPubKey": {
              "asm": "1 0de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad911",
              "desc": "rawtr(0de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad911)#n8rlk3jc",
              "hex": "51200de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad911",
              "address": "tb1pphs9eqnqw45lh3ajdslmuh9wv3uvs0x3eel99xhlhkjq46u2mygsk2r362",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 134,
            "scriptPubKey": {
              "asm": "0 96799725503e9844bb4f4ae9603f4e4036c3c064",
              "desc": "addr(tb1qjeuewf2s86vyfw60ft5kq06wgqmv8sryjkuwaa)#ch84qjgu",
              "hex": "001496799725503e9844bb4f4ae9603f4e4036c3c064",
              "address": "tb1qjeuewf2s86vyfw60ft5kq06wgqmv8sryjkuwaa",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 135,
            "scriptPubKey": {
              "asm": "0 87c219e169d8f6ec3884bb7ab0b56aa5fc1dfc8a",
              "desc": "addr(tb1qslppnctfmrmwcwyyhdatpdt25h7pmly2mwky5r)#xc8rhw03",
              "hex": "001487c219e169d8f6ec3884bb7ab0b56aa5fc1dfc8a",
              "address": "tb1qslppnctfmrmwcwyyhdatpdt25h7pmly2mwky5r",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 136,
            "scriptPubKey": {
              "asm": "0 d46c8e0c8a6ec9b93ca4d867b4bdaf6d373b85da",
              "desc": "addr(tb1q63kgury2dmymj09ympnmf0d0d5mnhpw634ffe2)#0uepdzrd",
              "hex": "0014d46c8e0c8a6ec9b93ca4d867b4bdaf6d373b85da",
              "address": "tb1q63kgury2dmymj09ympnmf0d0d5mnhpw634ffe2",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 137,
            "scriptPubKey": {
              "asm": "0 2b2f7b9d47395f39ecf45b16ef4ba3897512475c",
              "desc": "addr(tb1q9vhhh828890nnm85tvtw7jar3963y36u22ktjd)#m0wgvggr",
              "hex": "00142b2f7b9d47395f39ecf45b16ef4ba3897512475c",
              "address": "tb1q9vhhh828890nnm85tvtw7jar3963y36u22ktjd",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 138,
            "scriptPubKey": {
              "asm": "1 b9247057113c1ae76930723fd4679fa42055f3c5e2402102392329209de2c707",
              "desc": "rawtr(b9247057113c1ae76930723fd4679fa42055f3c5e2402102392329209de2c707)#ddvy3pdw",
              "hex": "5120b9247057113c1ae76930723fd4679fa42055f3c5e2402102392329209de2c707",
              "address": "tb1phyj8q4c38sdww6fswglageul5ss9tu79ufqzzq3eyv5jp80zcursvj533x",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 139,
            "scriptPubKey": {
              "asm": "0 82f77c9a8bfcdef0c2e20baeb49a3f377930b062",
              "desc": "addr(tb1qstmhex5tln00pshzpwhtfx3lxaunpvrzh2qj6q)#3y93pdtq",
              "hex": "001482f77c9a8bfcdef0c2e20baeb49a3f377930b062",
              "address": "tb1qstmhex5tln00pshzpwhtfx3lxaunpvrzh2qj6q",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 140,
            "scriptPubKey": {
              "asm": "0 68631e4b6da8e82432a7b4faee74d19fb1048459",
              "desc": "addr(tb1qdp33ujmd4r5zgv48knawuax3n7csfpzehxjc6a)#hyhz64l5",
              "hex": "001468631e4b6da8e82432a7b4faee74d19fb1048459",
              "address": "tb1qdp33ujmd4r5zgv48knawuax3n7csfpzehxjc6a",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 141,
            "scriptPubKey": {
              "asm": "0 14a333eff64e20c1e57aed89f3e8918fecdb8a16",
              "desc": "addr(tb1qzj3n8mlkfcsvret6akyl86y33lkdhzsk6mu3y7)#c66ljl6l",
              "hex": "001414a333eff64e20c1e57aed89f3e8918fecdb8a16",
              "address": "tb1qzj3n8mlkfcsvret6akyl86y33lkdhzsk6mu3y7",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 142,
            "scriptPubKey": {
              "asm": "0 2b66752b3950f78f3fdfe59e5b926e483e36eafe",
              "desc": "addr(tb1q9dn822ee2rmc707luk09hynwfqlrd6h7y62krx)#3yufwlw7",
              "hex": "00142b66752b3950f78f3fdfe59e5b926e483e36eafe",
              "address": "tb1q9dn822ee2rmc707luk09hynwfqlrd6h7y62krx",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 143,
            "scriptPubKey": {
              "asm": "1 6d3714e0786ada55ec210ea98767c088f9f4125534e1912b66177f37c713e427",
              "desc": "rawtr(6d3714e0786ada55ec210ea98767c088f9f4125534e1912b66177f37c713e427)#47gkqeak",
              "hex": "51206d3714e0786ada55ec210ea98767c088f9f4125534e1912b66177f37c713e427",
              "address": "tb1pd5m3fcrcdtd9tmppp65cwe7q3rulgyj4xnsez2mxzaln03cnusnsmn68fl",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 144,
            "scriptPubKey": {
              "asm": "1 7ad680410729850fdb0de4b3b2333c272f0fba4dd81fdcdbb6f47941dd2bdfb9",
              "desc": "rawtr(7ad680410729850fdb0de4b3b2333c272f0fba4dd81fdcdbb6f47941dd2bdfb9)#4r770esl",
              "hex": "51207ad680410729850fdb0de4b3b2333c272f0fba4dd81fdcdbb6f47941dd2bdfb9",
              "address": "tb1p0ttgqsg89xzslkcdujemyveuyuhslwjdmq0aekak73u5rhftm7uslsdktu",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 145,
            "scriptPubKey": {
              "asm": "1 ae1f1d86aebc4b4232d89829904dc43ca34ad45e1a973d9796cd2ad9c88802e7",
              "desc": "rawtr(ae1f1d86aebc4b4232d89829904dc43ca34ad45e1a973d9796cd2ad9c88802e7)#f77d54qp",
              "hex": "5120ae1f1d86aebc4b4232d89829904dc43ca34ad45e1a973d9796cd2ad9c88802e7",
              "address": "tb1p4c03mp4wh395yvkcnq5eqnwy8j3544z7r2tnm9uke54dnjygqtnsa5j878",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 146,
            "scriptPubKey": {
              "asm": "1 08ea988fda6e8e4dadad135b64079073bc346c2c1d1b5f96ff588e080e57d940",
              "desc": "rawtr(08ea988fda6e8e4dadad135b64079073bc346c2c1d1b5f96ff588e080e57d940)#euzfvyk5",
              "hex": "512008ea988fda6e8e4dadad135b64079073bc346c2c1d1b5f96ff588e080e57d940",
              "address": "tb1ppr4f3r76d68ymtddzddkgpusww7rgmpvr5d4l9hltz8qsrjhm9qq30dzwp",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 147,
            "scriptPubKey": {
              "asm": "1 748dd38489aff64cbddbdcd0c21727630d2227126d5683ed2edc03821fc601a8",
              "desc": "rawtr(748dd38489aff64cbddbdcd0c21727630d2227126d5683ed2edc03821fc601a8)#fmjj0g89",
              "hex": "5120748dd38489aff64cbddbdcd0c21727630d2227126d5683ed2edc03821fc601a8",
              "address": "tb1pwjxa8pyf4lmye0wmmngvy9e8vvxjyfcjd4tg8mfwmspcy87xqx5qqwzypr",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 148,
            "scriptPubKey": {
              "asm": "1 a6216f12040a08926484edc6d9a062559dd510d6949ebc75238aa815f9e09040",
              "desc": "rawtr(a6216f12040a08926484edc6d9a062559dd510d6949ebc75238aa815f9e09040)#qsm5q40n",
              "hex": "5120a6216f12040a08926484edc6d9a062559dd510d6949ebc75238aa815f9e09040",
              "address": "tb1p5csk7ysypgyfyeyyahrdngrz2kwa2yxkjj0tcafr325pt70qjpqqwrngur",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 149,
            "scriptPubKey": {
              "asm": "0 03efd34a33103972e6e855a0a6a3bdd2b27d6da7",
              "desc": "addr(tb1qq0haxj3nzquh9ehg2ks2dgaa62e86md8kyvwh6)#h3yyt9d0",
              "hex": "001403efd34a33103972e6e855a0a6a3bdd2b27d6da7",
              "address": "tb1qq0haxj3nzquh9ehg2ks2dgaa62e86md8kyvwh6",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 150,
            "scriptPubKey": {
              "asm": "0 d3bc7a26b553f56f561485c23d332a084ca42be4",
              "desc": "addr(tb1q6w785f44206k74s5shpr6ve2ppx2g2lyvx0rn8)#etzuk4ce",
              "hex": "0014d3bc7a26b553f56f561485c23d332a084ca42be4",
              "address": "tb1q6w785f44206k74s5shpr6ve2ppx2g2lyvx0rn8",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 151,
            "scriptPubKey": {
              "asm": "1 95977cad8246d629566910c401463a8243cd496607c3e6f389f0b845b7f235e4",
              "desc": "rawtr(95977cad8246d629566910c401463a8243cd496607c3e6f389f0b845b7f235e4)#5e0spyk7",
              "hex": "512095977cad8246d629566910c401463a8243cd496607c3e6f389f0b845b7f235e4",
              "address": "tb1pjkthetvzgmtzj4nfzrzqz336sfpu6jtxqlp7duuf7zuytdljxhjqaqxzd3",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 152,
            "scriptPubKey": {
              "asm": "0 f0de0c2ff5369e99c50e24e15be53161d7ca63ac",
              "desc": "addr(tb1q7r0qctl4x60fn3gwyns4hef3v8tu5cavgela7n)#mcfgvt6l",
              "hex": "0014f0de0c2ff5369e99c50e24e15be53161d7ca63ac",
              "address": "tb1q7r0qctl4x60fn3gwyns4hef3v8tu5cavgela7n",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 153,
            "scriptPubKey": {
              "asm": "1 3880a13bc19fc7786754b7c0c68fcc39ca52af368656a089514e9b4276aac9b6",
              "desc": "rawtr(3880a13bc19fc7786754b7c0c68fcc39ca52af368656a089514e9b4276aac9b6)#hmlqj4u7",
              "hex": "51203880a13bc19fc7786754b7c0c68fcc39ca52af368656a089514e9b4276aac9b6",
              "address": "tb1p8zq2zw7pnlrhse65klqvdr7v88999tekset2pz23f6d5ya42exmqltsr6q",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 154,
            "scriptPubKey": {
              "asm": "1 e63db0b11c8fc2a39cf310881c4414b1d81cac38816e6fca73a0a631338bbf74",
              "desc": "rawtr(e63db0b11c8fc2a39cf310881c4414b1d81cac38816e6fca73a0a631338bbf74)#rm3amygw",
              "hex": "5120e63db0b11c8fc2a39cf310881c4414b1d81cac38816e6fca73a0a631338bbf74",
              "address": "tb1puc7mpvgu3lp2888nzzypc3q5k8vpetpcs9hxljnn5znrzvutha6qwvv05m",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 155,
            "scriptPubKey": {
              "asm": "0 a2393f0fb6277cebff87f57eabdbc9d92ac3419f",
              "desc": "addr(tb1q5gun7rakya7whlu874l2hk7fmy4vxsvlc7flfe)#cz9ltlgt",
              "hex": "0014a2393f0fb6277cebff87f57eabdbc9d92ac3419f",
              "address": "tb1q5gun7rakya7whlu874l2hk7fmy4vxsvlc7flfe",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 156,
            "scriptPubKey": {
              "asm": "1 6567a23a2b37c98a559e08635a88a86afbc13d64f3fe6e49fa5ca310b0cd958a",
              "desc": "rawtr(6567a23a2b37c98a559e08635a88a86afbc13d64f3fe6e49fa5ca310b0cd958a)#uzd5jdga",
              "hex": "51206567a23a2b37c98a559e08635a88a86afbc13d64f3fe6e49fa5ca310b0cd958a",
              "address": "tb1pv4n6yw3txlyc54v7pp344z9gdtauz0ty70lxuj06tj33pvxdjk9q3hymqj",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 157,
            "scriptPubKey": {
              "asm": "0 d6850af8efb5bf461dd2979b46626c8a8f3875b3",
              "desc": "addr(tb1q66zs4780kkl5v8wjj7d5vcnv328nsadn8nj7zd)#kcjrw6q7",
              "hex": "0014d6850af8efb5bf461dd2979b46626c8a8f3875b3",
              "address": "tb1q66zs4780kkl5v8wjj7d5vcnv328nsadn8nj7zd",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 158,
            "scriptPubKey": {
              "asm": "1 d7f0fa7d98b42ad61f7b68fe0bdf9fdce649223a444053c5705613772e4fbd3f",
              "desc": "rawtr(d7f0fa7d98b42ad61f7b68fe0bdf9fdce649223a444053c5705613772e4fbd3f)#23gcjuut",
              "hex": "5120d7f0fa7d98b42ad61f7b68fe0bdf9fdce649223a444053c5705613772e4fbd3f",
              "address": "tb1p6lc05lvcks4dv8mmdrlqhhulmnnyjg36g3q983ts2cfhwtj0h5lskjha7n",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 159,
            "scriptPubKey": {
              "asm": "1 6ee8b93e18727dcf5eb26ff9a5b8d2d92f3b82d9a575002f6133c4b70ae56fee",
              "desc": "rawtr(6ee8b93e18727dcf5eb26ff9a5b8d2d92f3b82d9a575002f6133c4b70ae56fee)#u3c0cscx",
              "hex": "51206ee8b93e18727dcf5eb26ff9a5b8d2d92f3b82d9a575002f6133c4b70ae56fee",
              "address": "tb1pdm5tj0scwf7u7h4jdlu6twxjmyhnhqke546sqtmpx0ztwzh9dlhqnwxa7u",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 160,
            "scriptPubKey": {
              "asm": "1 2bc652e652cb1487d176c8314cba4fda558de9322f64d02154ba7d306bb20b91",
              "desc": "rawtr(2bc652e652cb1487d176c8314cba4fda558de9322f64d02154ba7d306bb20b91)#hyxrk4u0",
              "hex": "51202bc652e652cb1487d176c8314cba4fda558de9322f64d02154ba7d306bb20b91",
              "address": "tb1p90r99ejjev2g05tkeqc5ewj0mf2cm6fj9ajdqg25hf7nq6ajpwgsl33ln3",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 161,
            "scriptPubKey": {
              "asm": "1 dde7bebc0bbd8d4864f03520292c05316c66a729ad5ecbcd9d344bda09d7033a",
              "desc": "rawtr(dde7bebc0bbd8d4864f03520292c05316c66a729ad5ecbcd9d344bda09d7033a)#ukzleuhl",
              "hex": "5120dde7bebc0bbd8d4864f03520292c05316c66a729ad5ecbcd9d344bda09d7033a",
              "address": "tb1pmhnma0qthkx5se8sx5szjtq9x9kxdfef440vhnvax39a5zwhqvaqg8hdc9",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 162,
            "scriptPubKey": {
              "asm": "0 e8702d03c1ecc5559bcd14587ed78e00200565c1",
              "desc": "addr(tb1qapcz6q7panz4tx7dz3v8a4uwqqsq2ewp2p8sjt)#kw5gj6hf",
              "hex": "0014e8702d03c1ecc5559bcd14587ed78e00200565c1",
              "address": "tb1qapcz6q7panz4tx7dz3v8a4uwqqsq2ewp2p8sjt",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 163,
            "scriptPubKey": {
              "asm": "0 e3a9ea956f7db807d4138b37cc0ece9eabaaa6cd",
              "desc": "addr(tb1quw5749t00kuq04qn3vmucrkwn6464fkd7w5rq0)#3nre4ju2",
              "hex": "0014e3a9ea956f7db807d4138b37cc0ece9eabaaa6cd",
              "address": "tb1quw5749t00kuq04qn3vmucrkwn6464fkd7w5rq0",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 164,
            "scriptPubKey": {
              "asm": "1 30972fa55c5aeb24194ccea526be4bd8db62a7eafa73eeefeb0fdc0208c4efaf",
              "desc": "rawtr(30972fa55c5aeb24194ccea526be4bd8db62a7eafa73eeefeb0fdc0208c4efaf)#63ldhc25",
              "hex": "512030972fa55c5aeb24194ccea526be4bd8db62a7eafa73eeefeb0fdc0208c4efaf",
              "address": "tb1pxztjlf2utt4jgx2ve6jjd0jtmrdk9fl2lfe7amltplwqyzxya7hs2z58m8",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 165,
            "scriptPubKey": {
              "asm": "1 ae3021a5a66cd55cfa001f3934c5904299783fef59fa8f031f2f8903a53aa99e",
              "desc": "rawtr(ae3021a5a66cd55cfa001f3934c5904299783fef59fa8f031f2f8903a53aa99e)#d50nydju",
              "hex": "5120ae3021a5a66cd55cfa001f3934c5904299783fef59fa8f031f2f8903a53aa99e",
              "address": "tb1p4cczrfdxdn24e7sqruunf3vsg2vhs0l0t8ag7qcl97ys8ff64x0qfgu4z7",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 166,
            "scriptPubKey": {
              "asm": "0 73fb8c500952c90ba316cf87c355b9c1599082ac",
              "desc": "addr(tb1qw0acc5qf2tyshgcke7rux4dec9vepq4v6h0j6n)#9thdle4r",
              "hex": "001473fb8c500952c90ba316cf87c355b9c1599082ac",
              "address": "tb1qw0acc5qf2tyshgcke7rux4dec9vepq4v6h0j6n",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 167,
            "scriptPubKey": {
              "asm": "1 854fa0bda4c15499b0517e8a86fc217473108aefe57051c894e7ddd37c5dbd9a",
              "desc": "rawtr(854fa0bda4c15499b0517e8a86fc217473108aefe57051c894e7ddd37c5dbd9a)#t7ts6k2d",
              "hex": "5120854fa0bda4c15499b0517e8a86fc217473108aefe57051c894e7ddd37c5dbd9a",
              "address": "tb1ps486p0dyc92fnvz3069gdlppw3e3pzh0u4c9rjy5ulwaxlzahkdqgsj9l0",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 168,
            "scriptPubKey": {
              "asm": "0 039fcefe51f1b43a470d0c91ca4c4dca7a6f25d5",
              "desc": "addr(tb1qqw0ualj37x6r53cdpjgu5nzdefax7fw49q2myz)#wk97290p",
              "hex": "0014039fcefe51f1b43a470d0c91ca4c4dca7a6f25d5",
              "address": "tb1qqw0ualj37x6r53cdpjgu5nzdefax7fw49q2myz",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 169,
            "scriptPubKey": {
              "asm": "1 6f4c514dfbaa44b98589db5527963fad867f554c157c577940a74c2cbd304107",
              "desc": "rawtr(6f4c514dfbaa44b98589db5527963fad867f554c157c577940a74c2cbd304107)#prpz9ura",
              "hex": "51206f4c514dfbaa44b98589db5527963fad867f554c157c577940a74c2cbd304107",
              "address": "tb1pdax9zn0m4fztnpvfmd2j093l4kr8742vz479w72q5axze0fsgyrs5teu9u",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 170,
            "scriptPubKey": {
              "asm": "0 34c5cd65ed5716be73884ea4cb1a34c84f694657",
              "desc": "addr(tb1qxnzu6e0d2uttuuugf6jvkx35ep8kj3jh0fgr4c)#9wk44jwm",
              "hex": "001434c5cd65ed5716be73884ea4cb1a34c84f694657",
              "address": "tb1qxnzu6e0d2uttuuugf6jvkx35ep8kj3jh0fgr4c",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 171,
            "scriptPubKey": {
              "asm": "1 8dbe6acec7a3cd38b6f241324de51f4d2e3aa4c3a15d0101e65f9c688acbcb90",
              "desc": "rawtr(8dbe6acec7a3cd38b6f241324de51f4d2e3aa4c3a15d0101e65f9c688acbcb90)#63vjsdzj",
              "hex": "51208dbe6acec7a3cd38b6f241324de51f4d2e3aa4c3a15d0101e65f9c688acbcb90",
              "address": "tb1p3klx4nk850xn3dhjgyeymeglf5hr4fxr59wszq0xt7wx3zktewgqm2ul3j",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 172,
            "scriptPubKey": {
              "asm": "1 855b12e4e3400bea202ef1ec273cbd01c7e0b0d1bd52396dc9065ef9010c4138",
              "desc": "rawtr(855b12e4e3400bea202ef1ec273cbd01c7e0b0d1bd52396dc9065ef9010c4138)#stzaf952",
              "hex": "5120855b12e4e3400bea202ef1ec273cbd01c7e0b0d1bd52396dc9065ef9010c4138",
              "address": "tb1ps4d39e8rgq975gpw78kzw09aq8r7pvx3h4frjmwfqe00jqgvgyuq43z6d5",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 173,
            "scriptPubKey": {
              "asm": "1 a99d54ce25ce55928808d29b384f16baab5bce72e2ebf4422d3ddfc1156c4ea4",
              "desc": "rawtr(a99d54ce25ce55928808d29b384f16baab5bce72e2ebf4422d3ddfc1156c4ea4)#y6pmf6e4",
              "hex": "5120a99d54ce25ce55928808d29b384f16baab5bce72e2ebf4422d3ddfc1156c4ea4",
              "address": "tb1p4xw4fn39ee2e9zqg62dnsnckh244hnnjut4lgs3d8h0uz9tvf6jqs4rudv",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 174,
            "scriptPubKey": {
              "asm": "0 746737654b6bd7d4f9344f3bb1327667adbe662f",
              "desc": "addr(tb1qw3nnwe2td0taf7f5fuamzvnkv7kmue300xhgu8)#zv5up0y6",
              "hex": "0014746737654b6bd7d4f9344f3bb1327667adbe662f",
              "address": "tb1qw3nnwe2td0taf7f5fuamzvnkv7kmue300xhgu8",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 175,
            "scriptPubKey": {
              "asm": "1 d444182ad5714a1fa108408a760217c54eed1d3c9aa639ac15fa62f6c08f58c9",
              "desc": "rawtr(d444182ad5714a1fa108408a760217c54eed1d3c9aa639ac15fa62f6c08f58c9)#w58qp97v",
              "hex": "5120d444182ad5714a1fa108408a760217c54eed1d3c9aa639ac15fa62f6c08f58c9",
              "address": "tb1p63zps2k4w99plggggz98vqshc48w68fun2nrntq4lf30dsy0trys8dn4dk",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 176,
            "scriptPubKey": {
              "asm": "1 2e1029120ca1a7c817cf1a7ab363a1ef255fae046b8dd7755f2fd5532eef44e1",
              "desc": "rawtr(2e1029120ca1a7c817cf1a7ab363a1ef255fae046b8dd7755f2fd5532eef44e1)#t7zmw94p",
              "hex": "51202e1029120ca1a7c817cf1a7ab363a1ef255fae046b8dd7755f2fd5532eef44e1",
              "address": "tb1p9cgzjysv5xnus970rfatxcapauj4ltsydwxawa2l9l24xth0gnss9fjaje",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 177,
            "scriptPubKey": {
              "asm": "1 f685f8649f4b659412f17bf116a351945684fa82d0d19db4e2f0b0c8f8c1c407",
              "desc": "rawtr(f685f8649f4b659412f17bf116a351945684fa82d0d19db4e2f0b0c8f8c1c407)#tlss3c7v",
              "hex": "5120f685f8649f4b659412f17bf116a351945684fa82d0d19db4e2f0b0c8f8c1c407",
              "address": "tb1p76zlseylfdjegyh300c3dg63j3tgf75z6rgemd8z7zcv37xpcsrsvcwqh3",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 178,
            "scriptPubKey": {
              "asm": "0 5e2fd9da4d468d7c035809c206821563693be39d",
              "desc": "addr(tb1qtchankjdg6xhcq6cp8pqdqs4vd5nhcuaatx8vz)#57z88jpp",
              "hex": "00145e2fd9da4d468d7c035809c206821563693be39d",
              "address": "tb1qtchankjdg6xhcq6cp8pqdqs4vd5nhcuaatx8vz",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 179,
            "scriptPubKey": {
              "asm": "1 e762ef28311918d45cc44f7b3953d1319abb615a16424f0c8b9ea7ff9b5c41c9",
              "desc": "rawtr(e762ef28311918d45cc44f7b3953d1319abb615a16424f0c8b9ea7ff9b5c41c9)#2umjpsuh",
              "hex": "5120e762ef28311918d45cc44f7b3953d1319abb615a16424f0c8b9ea7ff9b5c41c9",
              "address": "tb1pua3w72p3ryvdghxyfaanj573xxdtkc26zepy7rytn6nllx6ug8ysduz9yr",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 180,
            "scriptPubKey": {
              "asm": "0 13c51ca59d4541809271e5534ef998c8e6bebff3",
              "desc": "addr(tb1qz0z3efvag4qcpyn3u4f5a7vcernta0ln26km9m)#umzcsmn7",
              "hex": "001413c51ca59d4541809271e5534ef998c8e6bebff3",
              "address": "tb1qz0z3efvag4qcpyn3u4f5a7vcernta0ln26km9m",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 181,
            "scriptPubKey": {
              "asm": "1 a39ca67944416aa29a34868df1615820bb3705c67a406c5daac8ba02376ba50b",
              "desc": "rawtr(a39ca67944416aa29a34868df1615820bb3705c67a406c5daac8ba02376ba50b)#9jayhsym",
              "hex": "5120a39ca67944416aa29a34868df1615820bb3705c67a406c5daac8ba02376ba50b",
              "address": "tb1p5ww2v72yg9429x35s6xlzc2cyzanwpwx0fqxchd2ezaqydmt559s8y656z",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 182,
            "scriptPubKey": {
              "asm": "1 9bbc70a2dbbcef9aeb420cba44fa28e757122b46902f39c119ce28b0d6746607",
              "desc": "rawtr(9bbc70a2dbbcef9aeb420cba44fa28e757122b46902f39c119ce28b0d6746607)#cjxdf575",
              "hex": "51209bbc70a2dbbcef9aeb420cba44fa28e757122b46902f39c119ce28b0d6746607",
              "address": "tb1pnw78pgkmhnhe466zpjayf73guat3y26xjqhnnsgeec5tp4n5vcrs90yg0f",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 183,
            "scriptPubKey": {
              "asm": "0 3185d8c616a56eed6b4e378c1899f9038bbed437",
              "desc": "addr(tb1qxxza33sk54hw666wx7xp3x0eqw9ma4phvn30cs)#auv9tjmk",
              "hex": "00143185d8c616a56eed6b4e378c1899f9038bbed437",
              "address": "tb1qxxza33sk54hw666wx7xp3x0eqw9ma4phvn30cs",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 184,
            "scriptPubKey": {
              "asm": "1 2f731f3876cc378a7eeb10864373ac48d391b56f56d4547669a357e793066297",
              "desc": "rawtr(2f731f3876cc378a7eeb10864373ac48d391b56f56d4547669a357e793066297)#73q3knvv",
              "hex": "51202f731f3876cc378a7eeb10864373ac48d391b56f56d4547669a357e793066297",
              "address": "tb1p9ae37wrkesmc5lhtzzryxuavfrferdt02m29ganf5dt70ycxv2tsye3kgl",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 185,
            "scriptPubKey": {
              "asm": "1 728f9e3b11cd44ffaeec97172b6d57443153cf7a7c67005b28a162bb9500204c",
              "desc": "rawtr(728f9e3b11cd44ffaeec97172b6d57443153cf7a7c67005b28a162bb9500204c)#fm0j7xu7",
              "hex": "5120728f9e3b11cd44ffaeec97172b6d57443153cf7a7c67005b28a162bb9500204c",
              "address": "tb1pw28euwc3e4z0lthvjutjkm2hgsc48nm603nsqkeg593th9gqypxqvd2wz3",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 186,
            "scriptPubKey": {
              "asm": "1 b5cda5cd90f560b48d25f4ad39a09ff0876838c41de1ac91032391350afe52d8",
              "desc": "rawtr(b5cda5cd90f560b48d25f4ad39a09ff0876838c41de1ac91032391350afe52d8)#qj7akpa8",
              "hex": "5120b5cda5cd90f560b48d25f4ad39a09ff0876838c41de1ac91032391350afe52d8",
              "address": "tb1pkhx6tnvs74stfrf97jknngyl7zrkswxyrhs6eygrywgn2zh72tvqcp76v6",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 187,
            "scriptPubKey": {
              "asm": "1 ddfc568284cec53cd9a71bd64bcd1e93848ea36ba7b3b15b07b55a9fb76164b5",
              "desc": "rawtr(ddfc568284cec53cd9a71bd64bcd1e93848ea36ba7b3b15b07b55a9fb76164b5)#2mvc8mne",
              "hex": "5120ddfc568284cec53cd9a71bd64bcd1e93848ea36ba7b3b15b07b55a9fb76164b5",
              "address": "tb1pmh79dq5yemznekd8r0tyhng7jwzgagmt57emzkc8k4dfldmpvj6svj59g0",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 188,
            "scriptPubKey": {
              "asm": "1 41e48dbcc749768e67d8ba3d2bd88e57ccf4f6bd3682e13927cc691aff7ca272",
              "desc": "rawtr(41e48dbcc749768e67d8ba3d2bd88e57ccf4f6bd3682e13927cc691aff7ca272)#pg3th9ca",
              "hex": "512041e48dbcc749768e67d8ba3d2bd88e57ccf4f6bd3682e13927cc691aff7ca272",
              "address": "tb1pg8jgm0x8f9mgue7chg7jhkyw2lx0fa4ax6pwzwf8e3534lmu5feqwzkauu",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 189,
            "scriptPubKey": {
              "asm": "0 dd98c04cf81f81c02057e72dcb30d9c1aa389fb6",
              "desc": "addr(tb1qmkvvqn8cr7quqgzhuukukvxecx4r38akeh2qzw)#c3nepxaj",
              "hex": "0014dd98c04cf81f81c02057e72dcb30d9c1aa389fb6",
              "address": "tb1qmkvvqn8cr7quqgzhuukukvxecx4r38akeh2qzw",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 190,
            "scriptPubKey": {
              "asm": "1 0de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad911",
              "desc": "rawtr(0de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad911)#n8rlk3jc",
              "hex": "51200de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad911",
              "address": "tb1pphs9eqnqw45lh3ajdslmuh9wv3uvs0x3eel99xhlhkjq46u2mygsk2r362",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 191,
            "scriptPubKey": {
              "asm": "0 4c771f0cf6647be81073857675c6e8d73f7a523b",
              "desc": "addr(tb1qf3m37r8kv3a7syrns4m8t3hg6ulh553m6p4nqu)#eysrfns8",
              "hex": "00144c771f0cf6647be81073857675c6e8d73f7a523b",
              "address": "tb1qf3m37r8kv3a7syrns4m8t3hg6ulh553m6p4nqu",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 192,
            "scriptPubKey": {
              "asm": "1 90a2e1e4a685e27ca10726038257ee2710931efb9867e64d6f4f951751de6b5d",
              "desc": "rawtr(90a2e1e4a685e27ca10726038257ee2710931efb9867e64d6f4f951751de6b5d)#lmggwv2e",
              "hex": "512090a2e1e4a685e27ca10726038257ee2710931efb9867e64d6f4f951751de6b5d",
              "address": "tb1pjz3wre9xsh38egg8ycpcy4lwyugfx8hmnpn7vnt0f723w5w7ddwst742xp",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 193,
            "scriptPubKey": {
              "asm": "0 f85c35dbe780d41d81857758b59222fa4f290123",
              "desc": "addr(tb1qlpwrtkl8sr2pmqv9wavtty3zlf8jjqfru2nyw4)#eapyq40f",
              "hex": "0014f85c35dbe780d41d81857758b59222fa4f290123",
              "address": "tb1qlpwrtkl8sr2pmqv9wavtty3zlf8jjqfru2nyw4",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 194,
            "scriptPubKey": {
              "asm": "0 d9ede65c9109781394365cef36c4996b77c4a265",
              "desc": "addr(tb1qm8k7vhy3p9up89pktnhnd3yeddmufgn9rpkday)#87j3cn5f",
              "hex": "0014d9ede65c9109781394365cef36c4996b77c4a265",
              "address": "tb1qm8k7vhy3p9up89pktnhnd3yeddmufgn9rpkday",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 195,
            "scriptPubKey": {
              "asm": "0 685dc304d23a62f2d68a0bc6641a30b95e98bfb4",
              "desc": "addr(tb1qdpwuxpxj8f309452p0rxgx3sh90f30a57gtws6)#kp0s4mst",
              "hex": "0014685dc304d23a62f2d68a0bc6641a30b95e98bfb4",
              "address": "tb1qdpwuxpxj8f309452p0rxgx3sh90f30a57gtws6",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 196,
            "scriptPubKey": {
              "asm": "1 11dfb31ffe6058a83b23d6d3d6ac34a532d4fc9693c4cf8d570890696a9ed294",
              "desc": "rawtr(11dfb31ffe6058a83b23d6d3d6ac34a532d4fc9693c4cf8d570890696a9ed294)#z2g6n4gw",
              "hex": "512011dfb31ffe6058a83b23d6d3d6ac34a532d4fc9693c4cf8d570890696a9ed294",
              "address": "tb1pz80mx8l7vpv2swer6mfadtp555edflykj0zvlr2hpzgxj657622q5qnq0d",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 197,
            "scriptPubKey": {
              "asm": "0 45976196a77a96906c0ba71f3801bdf2ef81e445",
              "desc": "addr(tb1qgktkr94802tfqmqt5u0nsqda7thcrez9f9wmpw)#xdpe5f37",
              "hex": "001445976196a77a96906c0ba71f3801bdf2ef81e445",
              "address": "tb1qgktkr94802tfqmqt5u0nsqda7thcrez9f9wmpw",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 198,
            "scriptPubKey": {
              "asm": "0 a9c40ca336411d4d8468014c12fb3a0b809724a5",
              "desc": "addr(tb1q48zqegekgyw5mprgq9xp97e6pwqfwf99rpp60k)#7xqfgee9",
              "hex": "0014a9c40ca336411d4d8468014c12fb3a0b809724a5",
              "address": "tb1q48zqegekgyw5mprgq9xp97e6pwqfwf99rpp60k",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 199,
            "scriptPubKey": {
              "asm": "0 19c0b82b6fb6275416207aa7651185f7c3fac37b",
              "desc": "addr(tb1qr8qts2m0kcn4g93q02nk2yv97lpl4smm75p24k)#gdgkvfsy",
              "hex": "001419c0b82b6fb6275416207aa7651185f7c3fac37b",
              "address": "tb1qr8qts2m0kcn4g93q02nk2yv97lpl4smm75p24k",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 200,
            "scriptPubKey": {
              "asm": "1 bfb2af34fb0f9dfeda4d7708abd2db404af0b0e979ba73fc6d53e83833b4961b",
              "desc": "rawtr(bfb2af34fb0f9dfeda4d7708abd2db404af0b0e979ba73fc6d53e83833b4961b)#u5qft6xs",
              "hex": "5120bfb2af34fb0f9dfeda4d7708abd2db404af0b0e979ba73fc6d53e83833b4961b",
              "address": "tb1ph7e27d8mp7wlakjdwuy2h5kmgp90pv8f0xa88lrd205rsva5jcds2j5z8g",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 201,
            "scriptPubKey": {
              "asm": "1 43e2e7ee3220c05462ddec98b552f7f02c36ba08a703d6dea352c39ad9a55ed8",
              "desc": "rawtr(43e2e7ee3220c05462ddec98b552f7f02c36ba08a703d6dea352c39ad9a55ed8)#elusqdrc",
              "hex": "512043e2e7ee3220c05462ddec98b552f7f02c36ba08a703d6dea352c39ad9a55ed8",
              "address": "tb1pg03w0m3jyrq9gckaajvt25hh7qkrdwsg5upadh4r2tpe4kd9tmvqqq394m",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 202,
            "scriptPubKey": {
              "asm": "0 7ec0cb316050cc23792e98e0724ba0412b7ac60b",
              "desc": "addr(tb1q0mqvkvtq2rxzx7fwnrs8yjaqgy4h43stzvp58r)#4kxshfk6",
              "hex": "00147ec0cb316050cc23792e98e0724ba0412b7ac60b",
              "address": "tb1q0mqvkvtq2rxzx7fwnrs8yjaqgy4h43stzvp58r",
              "type": "witness_v0_keyhash"
            }
          },
          {
            "value": 0.25540109,
            "n": 203,
            "scriptPubKey": {
              "asm": "1 11bfadb5e1f16fd0ae626925e664673f6e34ba3105075037f9fd03d48fc23088",
              "desc": "rawtr(11bfadb5e1f16fd0ae626925e664673f6e34ba3105075037f9fd03d48fc23088)#sr7692s7",
              "hex": "512011bfadb5e1f16fd0ae626925e664673f6e34ba3105075037f9fd03d48fc23088",
              "address": "tb1pzxl6md0p79haptnzdyj7ver88ahrfw33q5r4qdlel5pafr7zxzyqqhryk7",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 204,
            "scriptPubKey": {
              "asm": "1 5b148f68535e6d4742d0e9908e343e09596b001f9695d28629a54ce47628b3bf",
              "desc": "rawtr(5b148f68535e6d4742d0e9908e343e09596b001f9695d28629a54ce47628b3bf)#fl6hcvsk",
              "hex": "51205b148f68535e6d4742d0e9908e343e09596b001f9695d28629a54ce47628b3bf",
              "address": "tb1ptv2g76zntek5wsksaxggudp7p9vkkqqlj62a9p3f54xwga3gkwls954vpr",
              "type": "witness_v1_taproot"
            }
          },
          {
            "value": 0.25540109,
            "n": 205,
            "scriptPubKey": {
              "asm": "0 e1309a63c20ac99086ebe11f2eb60979a3aaab51",
              "desc": "addr(tb1quycf5c7zptyepphtuy0jadsf0x364263t5p9t4)#079fn2em",
              "hex": "0014e1309a63c20ac99086ebe11f2eb60979a3aaab51",
              "address": "tb1quycf5c7zptyepphtuy0jadsf0x364263t5p9t4",
              "type": "witness_v0_keyhash"
            }
          }
        ]
      },
      "witness_script": {
        "asm": "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 0a237e896c58bf43d6eded629a4b2abb0c0a9175 OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0"
        },
        {
          "pubkey": "0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0"
        }
      ]
    }
  ],
  "outputs": [
    {
    },
    {
      "bip32_derivs": [
        {
          "pubkey": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1"
        },
        {
          "pubkey": "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
    },
    {
    }
  ],
  "fee": 0.00001920
}
```

We'll note the following of this PSBT:
* it has a `locktime` of 0
* the only input has a `sequence` of 0xfffffffd
* the `inputs` field has an entry for every vout in our faucet funding tx, even though only one of them belongs to this wallet.

</p>
</details>

```python

# assemble the inputs from our only utxo, note how much is spendable
>>> unspent = rpc.listunspent()[0]
>>> spendable = unspent['amount']
>>> txin = [dict(txid=unspent['txid'], vout=unspent['vout'])]

# assemble the outputs
>>> txout = []

# we'll need the internal wallet's receive descriptors
>>> rcv_desc = rpc.listdescriptors()['descriptors'][0]

# we'll send 80,000 sats to this wallet
>>> my_addr = rpc.deriveaddresses(rcv_desc['desc'], [rcv_desc['next'], rcv_desc['next']])[0]
>>> my_amt = Decimal("0.0008")
>>> txout.append({my_addr: my_amt})
>>> spendable -= my_amt

# we'll also send 25,400,000 sats to an external wallet
>>> send_addr = "tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve"
>>> send_amt = Decimal("0.254")
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
[{'txid': 'a371beff15d028353500b31448cc2923fc9e2560a37c46cddd48c71743da899a',
  'vout': 128}]

>>> pprint(txout)
[{'tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u': Decimal('0.0008')},
 {'tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve': Decimal('0.254')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00058549')}]

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAJwCAAAAAZqJ2kMXx0jdzUZ8o2AlnvwjKcxIFLMANTUo0BX/vnGjgAAAAAD9////A4A4AQAAAAAAIgAg5+0KGQJ3g9tbVEcwUsRKTAGcwiz3kxx0/p3i+mTNcVXAkoMBAAAAABYAFKR8CLnpMyV3pPH0yauuWcSqzS6GteQAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yWyRAwAAAQErDbaFAQAAAAAiACDN2Ry5l9ugZeXg7o2YFqVEjEQtmNRGieDJjmnuZ+h3FwEFQiEDPDQ+Bjgbfnkm/b5UxjqwBAxcLISR4tS9cjjLkGYEChusc2R2qRQKI36JbFi/Q9bt7WKaSyq7DAqRdYitASSyaCIGAzw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobHAf9gW0wAACAAQAAgAAAAIACAACAAAAAAAAAAAAiBgNGAjt4cuACdXfYpFE4m8WkMOcp8tOJoj7MoSN9w7f31hzahVofMAAAgAEAAIAAAACAAgAAgAAAAAAAAAAAAAEBQiECJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNCsc2R2qRTqOo95tKlpax8e17Vffy8WT8dPe4itASSyaCICAiSn5ICJmhTqAUBmfu4cOVxdIQUbVHoTja+rVMw39rTQHAf9gW0wAACAAQAAgAAAAIACAACAAAAAAAEAAAAiAgLK1hN8xIKM6roUgiexLTqum0QKfhKct5mnsyeI89kurRzahVofMAAAgAEAAIAAAACAAgAAgAAAAAABAAAAAAAA'

# result of signing by external signer (in this case, krux)
>>> signed_psbt = "cHNidP8BAJwCAAAAAZqJ2kMXx0jdzUZ8o2AlnvwjKcxIFLMANTUo0BX/vnGjgAAAAAD9////A4A4AQAAAAAAIgAg5+0KGQJ3g9tbVEcwUsRKTAGcwiz3kxx0/p3i+mTNcVXAkoMBAAAAABYAFKR8CLnpMyV3pPH0yauuWcSqzS6GteQAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yWyRAwAAAQErDbaFAQAAAAAiACDN2Ry5l9ugZeXg7o2YFqVEjEQtmNRGieDJjmnuZ+h3FyICAzw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobRzBEAiB0Hf6dTGePBkt2TWx9pSYOFZp6mkp0VWRn4VJaM+9/2wIgHZVO4ZnaJCcz+tzLWyVv5iVIQC26OSS1xq7wZ0saAdABAQVCIQM8ND4GOBt+eSb9vlTGOrAEDFwshJHi1L1yOMuQZgQKG6xzZHapFAojfolsWL9D1u3tYppLKrsMCpF1iK0BJLJoAAAAAA=="

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, signed_psbt])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de)

---

## Primary: Evolution of a PSBT

In the above session, the PSBT evolves with each step. Below these changes are explained.

### `psbt = rpc.createpsbt(...`
Bitcoin Core creates the initial psbt template, knowing only the input's `txid` and `vout`, but including no information about what that input is worth or what the spending conditions might be.
```json
{
  "tx": {
    "txid": "97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de",
    "hash": "97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de",
    "version": 2,
    "size": 156,
    "vsize": 156,
    "weight": 624,
    "locktime": 233836,
    "vin": [
      {
        "txid": "a371beff15d028353500b31448cc2923fc9e2560a37c46cddd48c71743da899a",
        "vout": 128,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967293
      }
    ],
    "vout": [
      {
        "value": 0.00080000,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
          "desc": "addr(tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u)#5tef56at",
          "hex": "0020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
          "address": "tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u",
          "type": "witness_v0_scripthash"
        }
      },
      {
        "value": 0.25400000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 a47c08b9e9332577a4f1f4c9abae59c4aacd2e86",
          "desc": "addr(tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve)#040gd8sk",
          "hex": "0014a47c08b9e9332577a4f1f4c9abae59c4aacd2e86",
          "address": "tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00058549,
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
        "amount": 0.25540109,
        "scriptPubKey": {
          "asm": "0 cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
          "desc": "addr(tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw)#7x8kgusx",
          "hex": "0020cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
          "address": "tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw",
          "type": "witness_v0_scripthash"
        }
      },
      "witness_script": {
        "asm": "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 0a237e896c58bf43d6eded629a4b2abb0c0a9175 OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0"
        },
        {
          "pubkey": "0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0"
        }
      ]

```
It also adds information pertaining to any of the outputs which are known to belong to the descriptors. In this case, it fills the first of three empty dictionaries with the following content -- within the outputs list.
```json
      "witness_script": {
        "asm": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1"
        },
        {
          "pubkey": "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```
Last of all, since the value of the inputs is now known, it can finally calculate and append the fee.
```json
   ,
  "fee": 0.00001560
```

### `signed_psbt = ...`
It is the job of the Signer to display pertinent information to the user so that they understand what is being signed, and to add `partial_signatures` to the PSBT.  In this case, the Signer adds a key to the existing dictionary in `inputs`
```json
      "partial_signatures": {
        "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b": "30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d001"
      },
```
The Signer's role in the PSBT is simply to "add" signatures, and done.  In fact,
[BIP-174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki#signer)
clearly states: "The Signer must only add data to a PSBT.", but some rules were meant to be broken?!?!

Because an air-gapped Signer which transmits data via QR-Code is constrained by this low-bandwidth medium, some have "optimized" to transfer less -- by stripping unnecessary fields from `inputs` and `outputs`.  Krux strips`bip32_derivs`from both `inputs` and `outputs` and also strips `witness_script from the `outputs`.

This is a non-standard `feature` of some air-gapped-via-qrcode Signers which is NOT practiced when a signed PSBT can be written efficiently to an sdcard or transfered electronically.  As expected, this non-standard feature must also be tolerated by whichever software is acting in the role of "Combiner", as we'll see below.

### `combined_psbt = rpc.combinepsbt(...`
Had the air-gapped signer not stripped any fields in its quest for an optimized transmit-via-qrocde user-experience, the resulting `signed_psbt` would have been the same as the `combined_psbt`.  Still, because other types of transactions may require multiple signatures from multiple Signers, it is important for the Combiner to consider combining all signed PSBTs with the original PSBT distributed to Signers.  In this case, Bitcoin Core simply replaces the dictionary keys that had been stripped out.

Recall that Krux had stripped from `inputs` -- it gets restored.
```json
       ,
      "bip32_derivs": [
        {
          "pubkey": "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0"
        },
        {
          "pubkey": "0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0"
        }
      ]
```
..and Bitcoin Core also restores content that had been stripped from the `outputs` field.
```json
      "witness_script": {
        "asm": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1"
        },
        {
          "pubkey": "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
As the Finalizer role, Bitcoin Core is tasked to determine whether or not the PSBT is complete and final, by verifying that spending conditions for all inputs have been met.  In this case, finalizing the complete PSBT strips-out previously added `script_witness` and `bip32_derivs` and replaces `partial_signatures` with `final_scriptwitness` within the `inputs` field.
```json
      "final_scriptwitness": [
        "30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d001",
        "21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b268"
      ]
```

By default, when the PSBT is complete and final, Bitcoin Core's `finalizepsbt` will also extract the final rawtransaction hex so that it's ready to be broadcasted into the mempool.

```json
{
  "txid": "97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de",
  "hash": "3d8262f928fb0077e9d368163ecf0c2eb7667a9d38b29d444bf716329dc538b7",
  "version": 2,
  "size": 298,
  "vsize": 192,
  "weight": 766,
  "locktime": 233836,
  "vin": [
    {
      "txid": "a371beff15d028353500b31448cc2923fc9e2560a37c46cddd48c71743da899a",
      "vout": 128,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d001",
        "21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b268"
      ],
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00080000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
        "desc": "addr(tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u)#5tef56at",
        "hex": "0020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
        "address": "tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u",
        "type": "witness_v0_scripthash"
      }
    },
    {
      "value": 0.25400000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 a47c08b9e9332577a4f1f4c9abae59c4aacd2e86",
        "desc": "addr(tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve)#040gd8sk",
        "hex": "0014a47c08b9e9332577a4f1f4c9abae59c4aacd2e86",
        "address": "tb1q537q3w0fxvjh0f837ny6htjecj4v6t5x2mh5ve",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 0.00058549,
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

In a nutshell, spending programmable money is about the owner providing `bitcoin script` "inputs" that will be completed by the utxo's `bitcoin scriptPubKey`, completing without error and leaving a `True` value on top of the stack.  It's necessary to know about the `scriptPubKey` that is part of the address from the input utxo -- whose source was the input transaction and which is stored within each node's `utxoset`.  Below is our input (the one-hundred-and-twenty-ninth output and utxo of our alt-signet-faucet funding transaction):
```
    {
      "value": 0.25540109,
      "n": 128,
      "scriptPubKey": {
        "asm": "0 cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
        "desc": "addr(tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw)#7x8kgusx",
        "hex": "0020cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e87717",
        "address": "tb1qehv3ewvhmwsxte0qa6xes949gjxygtvc63rgncxf3e57uelgwutsje79fw",
        "type": "witness_v0_scripthash"
      }
    },
```

Below, we will start a `btcdeb` session at the bash commandline using the rawtransaction hex of the above transaction, as well as the same for this transaction's only input.
```
btcdeb --quiet --tx=020000000001019a89da4317c748ddcd467ca360259efc2329cc4814b300353528d015ffbe71a38000000000fdffffff038038010000000000220020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155c092830100000000160014a47c08b9e9332577a4f1f4c9abae59c4aacd2e86b5e4000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac9024730440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d0014221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b2686c910300 --txin=03000000000101df698ebc39dc68c2f498f8be543998474b92683a15a153e2c88ad85e05c4f4640000000000fdffffffce7efcfb703b000000225120aac35fe91f20d48816b3c83011d117efa35acd2414d36c1e02b0f29fc3106d900000000000000000166a14616c742e7369676e65746661756365742e636f6d0db6850100000000160014d546529ed49fbb6ef9ae96f93719b00cb206f3f20db68501000000002251201a1f4da28a52826763fb10c99a8d622f4878fc5fc9cf859038b5ea6a1a032e620db6850100000000225120f947fbf9a49fe4f1148a14a49122e327bfd4a26c8e1919eba9dc12c1d95f50b60db6850100000000160014780c8d3869423c5d970a052acec7b4b6eaf166db0db6850100000000160014abc0c275abcae88a571dd921e2b0a00bc7a716330db6850100000000225120b34c88ee56ae027b5f6070f83f0eaba28f03108f23fc033ea007ca655956a56a0db685010000000022512026a312cc93ca503c1ab630fb7b6f6437f26176f86ac5f41a47238417de7cac6d0db6850100000000160014aed16c0a3815034b6932625f5e3e84cba4ea2e3f0db68501000000002251206ab2154424ce007a343523d9b22e03ed1b8f69db1d9f783446d4590acdf7ccef0db6850100000000160014e4b8586879e3d82ceb1074cb1e675e07bd6552050db6850100000000160014c11eac2f67bd0e5a4df6f15d554e992cc3d970e10db685010000000022512009d55a1144284dbb2595362365a48c66a620f54330cd1db2ff6345daac3dbfdd0db6850100000000225120caa84a0715d7346856e9bd58f4a556ea16244eafc98e3332a8514e4a29be243f0db685010000000022512031dbb15650bc94de1f9757bf282d460e7dbf0adb2e0cf71a92a5902898ebd7a50db6850100000000160014690a4cb2d4f04ce6f4a01eecb49a2236101a88570db6850100000000225120a70bb56c9fb853e8a3b715c716a075bb400330f8a2d794d7f482f521e66611d10db68501000000001600148f6389075f85daf64485ddaa9a9e3ee8b467959c0db6850100000000160014e2b41c3476fc131307e6a015866e5f1d6a6479a40db68501000000002251201ddc0f2a73a1478a1317810f1d2a5a280a0c7186ef38dfb6924de0b6e56e72330db6850100000000160014a43a5969f5867af70da345c552cb6a49116d2b910db6850100000000160014da97cb4d71d3748cfaad133291894a3de285d2ed0db6850100000000225120e07783777e903d3b0c5ceb0ada0cb1a4f22ce824d7849a7fba222ffe97d73a870db68501000000001600142053023dfa67c90b1c0fab8a492180292e40d0370db6850100000000160014ad0490ede4e87c09c8196f2f101710c1454e9e320db68501000000002251203817d64c803c3a364d96d69b93a9e321d6a62fb49f3f38550afe84d03e4aca390db6850100000000225120a8caa3782ac5c46679ee75b6f2741c5c3eda37d16c9d60d9a6b69e4ecd55458c0db685010000000016001432524d813b5ef0640b76ffe0fe217ce48288046f0db68501000000002251209693776751e9feac0b8c3a45cc8f809bd6cb99ea451bffb07b89d33a9cbaea3e0db6850100000000225120bad1e73367beb579f5e485d033437850925df568264723651569dab874d616060db68501000000002251209a5ad97b12fb10032a778e371c8828a2343e18cf477b38c9e6bf8f0125d58e760db68501000000001600145dab736568f12e173a36171852446bd27b5bf0870db6850100000000225120bf65d2a47cd348fe2fbe92f5037235dda2ea2559f5643035a22a896cafe3f4f30db6850100000000160014b5f7c00b3019ef021a2c537130ab0330418933500db6850100000000160014b689bdc65dc4879285b487a71c04bb93bb85ac140db6850100000000160014231e4507eda75d10abac31fd3af93a5e3fe1bcc70db6850100000000225120065a634eec5b64f01126ce6682eb4dace50d5d35ee8085460396f1b12ddf5a340db68501000000001600145d52bddaca6436ee7f6b29d1fabae7f59577624c0db685010000000022512009a70d381f5f68bbf520447f5316c001df4511ca5243fcef7f7b987266aae4ac0db68501000000002251208c43ccb26016cf0da7bd1f92dbaf6b353fe4d2077dae63020f329b90423572480db68501000000002251200c5d9251b4078f2b027a371e1f083030afe87b0d9e1e0cb9a7695bcc9cbb1c5b0db68501000000002251207ed5f1ddf5aa10808c7d4574f803a768df5fa7ed62dd10afa87264de0c9866a20db6850100000000160014917d1b7f4b9fdaf38c32690eb21e81258636613d0db6850100000000225120fce3007a0e51bb1078e2b36f51d985c789ac7455208f9ee306e09d8d5a40167d0db6850100000000160014094167dda649cd469f8f552f4116304896a5a17c0db685010000000016001486544590c80ec1aec9840f427f13efdc52a9eef20db685010000000022512040a193e16713dcd8765699c42b33891818aa4b7d9483f7f0b4be9116fb126f400db685010000000016001403c07ddf9d8d296cf5015865dae1a086fe4fa98c0db6850100000000225120c716194abfbd48aa18786db839863c823c2aaba0a6e46a6c11f32de4cc102c3c0db685010000000016001434857cf0a8d0903524ac378923b699b38c69428c0db685010000000016001469ab66c6a1374e1bdfbf6bff28587b6f62534ea20db6850100000000160014fa5d37901786bbecdb424cb7f8b901c0cce22cdf0db6850100000000225120a1ffe8d7e0f369cbf956fd516c5cc7241be728fe6fb669a2b9a630de12b154420db6850100000000225120487b9a4b6ccdb9cbc1e87a5bfb90aab6548bd6c5ef959d9d0f57b9dd973889080db6850100000000160014caf9e9464b29d0518960ef91edf0c7ea0913f4520db685010000000022512098d52226debc775b60c37b762178560cdb2f2885c0cf4da9b1dca0f9ecebb8f10db68501000000002251200156eeb8bf2fc72c9d0a51d61dbc9ea89b9d29a045bf4e7ddd585ea2f4713ce50db685010000000016001487047589aa035febcc8dd30736486bbbf69f6f050db685010000000022512011c9cc957ea6e46b9ddffec877c1e83e7b7b5f86bb929744bde602b78b4139c70db68501000000001600144becdfda6fe04d4c8a06e356046672fdc1e2ba6c0db68501000000001600145aedf7ea48a8200e1f50d90cc7cdfe44f940c7e40db685010000000022512006fc8a4f72bea101214dbe3426b1542801fdbb067fc1f6b57da4ada88dcbbeb10db685010000000016001424ed05ec42868a97100bb57d5ad452cd2d1399df0db68501000000001600146767c9d7f5562486d59c082d055570ff1ffe41100db6850100000000225120c13e909c21798494f6167f7a348cdca35205ef9bb17805325fd6673e908e2f4b0db6850100000000225120b8653d2f22b1d30970e0b281f0b740647e9afd20ad7fa6855b195aa102594be40db6850100000000160014812796319cbb8c73fabd038ace40dbb4a560e1320db6850100000000225120afe7ee1932ef5ed18bf299c19d3324b0ce6c4ce53bf52724b234a5b40d3cd3780db6850100000000160014f6b91767aa08662fa6e375f9da2401863b5860470db68501000000001600140f77d4e4988515215b72cd85d09c9a710df859640db68501000000001600144664d9d2b9d192bd90282d8767ef8d60669fda740db685010000000016001468fe1794c70b5b48366660062ef73d25b6f90ed20db6850100000000160014276b37212ffbeea8cc920dfeddb3e521824a14a80db685010000000016001457f77327b3bca7e64952e767cfdb971c1700303b0db6850100000000160014e614787d373d3480ace5f85b3b6f38f1d5ffdecb0db685010000000022512035e00ba751b0ca2d9ebdfc4569c3f0d561c0a145273a24a3b284f104d0d6db420db68501000000002251201d7b6e6bc1e73c75c9aba8a9935f7487672695c93f671fee8f3fd4c6c46f10b90db6850100000000160014f405045ef59204b7c828e187972edb9e1ea64cf50db6850100000000225120bdfa80719516d0b0bffd08f804b10e7958b917655741e626015ef398f57959a20db685010000000016001465518b67b59000767cc04a262ae10b6daa9caf090db6850100000000225120e181cc774a6de16ca2bf1f816817581ff61a4812b3b02dfd70a992a3f0e397ea0db6850100000000225120110ceaa9a80b3f8c55308ceef473481c05a7151eb57c0205835ae836de79d9f20db68501000000002251207d232f84d3282c1bf70ecb3d2e2009e58aa3627f88a7f4c3961e6d01a18139fa0db68501000000002251208d758a6e83f4c71de1642e0f231d4c341a76c63dbb91e6ed485421a0d6bc9ab60db6850100000000160014f4c6fd3890b39f31a1b8d290f1e3d7f17f6d92240db6850100000000225120524848cfde02027012221fda20dd1dbfc7b025a0506490ac461d5ea0899fbd130db68501000000001600145a8527c0a08f07a3b6d3fed75570c7e8ece733380db6850100000000225120255ed1e617ccf4c1c4420e018eb3c9e306922c0240f671860ec6c02d8ec034f80db685010000000016001491353994e4777fa9867805023959c5132b72bc1a0db6850100000000160014f8991111950543c01ecbb106238f5ef29c32ccfc0db6850100000000160014a3b2e051c0c5405276fe91833e0f06563bfc0e490db68501000000001600143e554ff09664b4e5b29864c25e28b4e7a38e28f90db6850100000000225120e60085afd0289f5664d294e4df68a930b07ffb84a5d579c32414d174c4b2f37c0db68501000000002251201f2e7dbb6780a7bba4c8cc235843d6396715f141cbfc8e42c597aa14bd4a383a0db6850100000000160014d4f6bebdc529d590f14b8061a86d9eae5e92d0fd0db6850100000000160014f539a661c27ed95fa55c6b7e76ceb9fe42900f470db68501000000001600148a5f63d2bba2f8b2e6b80b4c80e41cfd210a02d70db6850100000000160014c54bfe33a04606998496ac258fa9d8181b084e5b0db68501000000002251209fe0e22abdbe25a86f47758ea1cd00850979f0f3a8609f337cb01b8627658eab0db6850100000000160014a97e952e90f9d62f1428b5f871b46b89d5f1faff0db6850100000000225120a6f2853cef429f5d47da0fac527078ad8befdd2d3108d6e04556e1d2de3858400db68501000000001600148ead8c73ae5e7db441d3a1bfe9f64b44dbe3337e0db68501000000001600143d9997ed9d5041945accafbfa03810e852c93c280db68501000000001600148cb6a7b97f301eba3521bd11debd8746b9f772660db68501000000001600148a2ab926c3bba96867e20d021e0ce9ef402ca7320db68501000000002251207112a5301dc8504446af011181a724a46419dfd2594600824c33c1d79d1e312d0db6850100000000225120ad01b571c7f7de2ce9d14f4f23eed33124a4f035fdfc1f09e755f0f732e721430db685010000000022512087a1452c7430b0bd510f168971e484d11ee8d356267adf15adb06833f9c39ff10db6850100000000225120608a982c549dc8fd7305ad09359dd6d68fc99dfc7c845397b49b53b5e2731c930db68501000000001600147eaccf8f20192ba51b7505f432f77676ef12ca330db685010000000016001472edf05b63110aee33a85863762dd209398b15770db6850100000000160014c7cb01cd648083f9df30d28ca786b479347dc51a0db6850100000000225120ad310751da5175cc71f8b2443d359898ffc6741f5314d592165c83b52427306e0db6850100000000225120253db23f14f8473562a9b6c3b98b61962b62fbdb0c17032755a4c0fc4cd3b11a0db68501000000002251204822a2bf9b9373d6feb925378b1d5d8cb410a0a1da6d7a2b04d0dc7f3c6aba4b0db68501000000001600146b43c2932a3ee165dcef44355a43d5100c866ee80db685010000000016001401ad816304a9775b5af02af49b243497104f67fd0db6850100000000160014cca5a427f2b2f5c104c053877f1654edfc03f2cd0db6850100000000160014e7b1f8aaf18624267f452ba89f62befb6a8368230db6850100000000225120cf81df54ca395a0e2a2485da1af4085ca71bca1f7dde31e117b11687d63d1cfa0db68501000000001600142c958620b836492676b67087a773e8e8d63dc4f20db6850100000000160014b1a561376b8b3fb347da7017b38c4bd439f6f0eb0db68501000000001600142f037191dbbe002f668d571b10197216d44167430db6850100000000160014919df7a43f93eb322fd8ebcefae89e350a9deca90db6850100000000160014610d1d6a5b0a7009297ac47f6d329f229e8a2f860db68501000000001600146dbb51d9e9fc6a292985eb2105d1916cb59224c70db68501000000002251202f28d117159fd4a6b2a21e5c3dfb5dfaeabf24aa20a3c6da2025bac6ef78edbf0db6850100000000220020cdd91cb997dba065e5e0ee8d9816a5448c442d98d44689e0c98e69ee67e877170db6850100000000225120708599406e122d2b5ffa7c351edbee0ff542df0c99b700202568b3daddd1130c0db6850100000000225120554dfd648df03db69927e0f8ebba280e6ad073cbeca9bca9024b5b89857da7370db68501000000002251202d8f56c7c7d9256482fd0ca398ae29ab45e1e88db43d9d713a44f3ac0d5fc0970db6850100000000225120208c3f528fc15edb0c9e13528400035db26999a9b97a12e30a2af0fa80392a310db68501000000002251200de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad9110db685010000000016001496799725503e9844bb4f4ae9603f4e4036c3c0640db685010000000016001487c219e169d8f6ec3884bb7ab0b56aa5fc1dfc8a0db6850100000000160014d46c8e0c8a6ec9b93ca4d867b4bdaf6d373b85da0db68501000000001600142b2f7b9d47395f39ecf45b16ef4ba3897512475c0db6850100000000225120b9247057113c1ae76930723fd4679fa42055f3c5e2402102392329209de2c7070db685010000000016001482f77c9a8bfcdef0c2e20baeb49a3f377930b0620db685010000000016001468631e4b6da8e82432a7b4faee74d19fb10484590db685010000000016001414a333eff64e20c1e57aed89f3e8918fecdb8a160db68501000000001600142b66752b3950f78f3fdfe59e5b926e483e36eafe0db68501000000002251206d3714e0786ada55ec210ea98767c088f9f4125534e1912b66177f37c713e4270db68501000000002251207ad680410729850fdb0de4b3b2333c272f0fba4dd81fdcdbb6f47941dd2bdfb90db6850100000000225120ae1f1d86aebc4b4232d89829904dc43ca34ad45e1a973d9796cd2ad9c88802e70db685010000000022512008ea988fda6e8e4dadad135b64079073bc346c2c1d1b5f96ff588e080e57d9400db6850100000000225120748dd38489aff64cbddbdcd0c21727630d2227126d5683ed2edc03821fc601a80db6850100000000225120a6216f12040a08926484edc6d9a062559dd510d6949ebc75238aa815f9e090400db685010000000016001403efd34a33103972e6e855a0a6a3bdd2b27d6da70db6850100000000160014d3bc7a26b553f56f561485c23d332a084ca42be40db685010000000022512095977cad8246d629566910c401463a8243cd496607c3e6f389f0b845b7f235e40db6850100000000160014f0de0c2ff5369e99c50e24e15be53161d7ca63ac0db68501000000002251203880a13bc19fc7786754b7c0c68fcc39ca52af368656a089514e9b4276aac9b60db6850100000000225120e63db0b11c8fc2a39cf310881c4414b1d81cac38816e6fca73a0a631338bbf740db6850100000000160014a2393f0fb6277cebff87f57eabdbc9d92ac3419f0db68501000000002251206567a23a2b37c98a559e08635a88a86afbc13d64f3fe6e49fa5ca310b0cd958a0db6850100000000160014d6850af8efb5bf461dd2979b46626c8a8f3875b30db6850100000000225120d7f0fa7d98b42ad61f7b68fe0bdf9fdce649223a444053c5705613772e4fbd3f0db68501000000002251206ee8b93e18727dcf5eb26ff9a5b8d2d92f3b82d9a575002f6133c4b70ae56fee0db68501000000002251202bc652e652cb1487d176c8314cba4fda558de9322f64d02154ba7d306bb20b910db6850100000000225120dde7bebc0bbd8d4864f03520292c05316c66a729ad5ecbcd9d344bda09d7033a0db6850100000000160014e8702d03c1ecc5559bcd14587ed78e00200565c10db6850100000000160014e3a9ea956f7db807d4138b37cc0ece9eabaaa6cd0db685010000000022512030972fa55c5aeb24194ccea526be4bd8db62a7eafa73eeefeb0fdc0208c4efaf0db6850100000000225120ae3021a5a66cd55cfa001f3934c5904299783fef59fa8f031f2f8903a53aa99e0db685010000000016001473fb8c500952c90ba316cf87c355b9c1599082ac0db6850100000000225120854fa0bda4c15499b0517e8a86fc217473108aefe57051c894e7ddd37c5dbd9a0db6850100000000160014039fcefe51f1b43a470d0c91ca4c4dca7a6f25d50db68501000000002251206f4c514dfbaa44b98589db5527963fad867f554c157c577940a74c2cbd3041070db685010000000016001434c5cd65ed5716be73884ea4cb1a34c84f6946570db68501000000002251208dbe6acec7a3cd38b6f241324de51f4d2e3aa4c3a15d0101e65f9c688acbcb900db6850100000000225120855b12e4e3400bea202ef1ec273cbd01c7e0b0d1bd52396dc9065ef9010c41380db6850100000000225120a99d54ce25ce55928808d29b384f16baab5bce72e2ebf4422d3ddfc1156c4ea40db6850100000000160014746737654b6bd7d4f9344f3bb1327667adbe662f0db6850100000000225120d444182ad5714a1fa108408a760217c54eed1d3c9aa639ac15fa62f6c08f58c90db68501000000002251202e1029120ca1a7c817cf1a7ab363a1ef255fae046b8dd7755f2fd5532eef44e10db6850100000000225120f685f8649f4b659412f17bf116a351945684fa82d0d19db4e2f0b0c8f8c1c4070db68501000000001600145e2fd9da4d468d7c035809c206821563693be39d0db6850100000000225120e762ef28311918d45cc44f7b3953d1319abb615a16424f0c8b9ea7ff9b5c41c90db685010000000016001413c51ca59d4541809271e5534ef998c8e6bebff30db6850100000000225120a39ca67944416aa29a34868df1615820bb3705c67a406c5daac8ba02376ba50b0db68501000000002251209bbc70a2dbbcef9aeb420cba44fa28e757122b46902f39c119ce28b0d67466070db68501000000001600143185d8c616a56eed6b4e378c1899f9038bbed4370db68501000000002251202f731f3876cc378a7eeb10864373ac48d391b56f56d4547669a357e7930662970db6850100000000225120728f9e3b11cd44ffaeec97172b6d57443153cf7a7c67005b28a162bb9500204c0db6850100000000225120b5cda5cd90f560b48d25f4ad39a09ff0876838c41de1ac91032391350afe52d80db6850100000000225120ddfc568284cec53cd9a71bd64bcd1e93848ea36ba7b3b15b07b55a9fb76164b50db685010000000022512041e48dbcc749768e67d8ba3d2bd88e57ccf4f6bd3682e13927cc691aff7ca2720db6850100000000160014dd98c04cf81f81c02057e72dcb30d9c1aa389fb60db68501000000002251200de05c82607569fbc7b26c3fbe5cae6478c83cd1ce7e529affbda40aeb8ad9110db68501000000001600144c771f0cf6647be81073857675c6e8d73f7a523b0db685010000000022512090a2e1e4a685e27ca10726038257ee2710931efb9867e64d6f4f951751de6b5d0db6850100000000160014f85c35dbe780d41d81857758b59222fa4f2901230db6850100000000160014d9ede65c9109781394365cef36c4996b77c4a2650db6850100000000160014685dc304d23a62f2d68a0bc6641a30b95e98bfb40db685010000000022512011dfb31ffe6058a83b23d6d3d6ac34a532d4fc9693c4cf8d570890696a9ed2940db685010000000016001445976196a77a96906c0ba71f3801bdf2ef81e4450db6850100000000160014a9c40ca336411d4d8468014c12fb3a0b809724a50db685010000000016001419c0b82b6fb6275416207aa7651185f7c3fac37b0db6850100000000225120bfb2af34fb0f9dfeda4d7708abd2db404af0b0e979ba73fc6d53e83833b4961b0db685010000000022512043e2e7ee3220c05462ddec98b552f7f02c36ba08a703d6dea352c39ad9a55ed80db68501000000001600147ec0cb316050cc23792e98e0724ba0412b7ac60b0db685010000000022512011bfadb5e1f16fd0ae626925e664673f6e34ba3105075037f9fd03d48fc230880db68501000000002251205b148f68535e6d4742d0e9908e343e09596b001f9695d28629a54ce47628b3bf0db6850100000000160014e1309a63c20ac99086ebe11f2eb60979a3aaab5101407c0f1f2fa0f562a21a16901349d506ec52f3b7cb88137ea613b64066b2b74148428e9408dcda94031104d5e997030e9874365f1536bbccb8dcfc43a2c82d54ca5b910300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 128; value = 25540109
got witness stack of size 2
34 bytes (v0=P2WSH, v1=taproot/tapscript)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(a371beff15, 128)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b | 30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525...
OP_CHECKSIG                                                        | 
OP_IFDUP                                                           | 
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0000 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
```

Even though we used `--quiet`, we have plenty of output.
* we're signing segwit or taproot
* btcdeb will work one input at a time, and it knows the vout and the value in sats
* It knows that signing this transaction initialized the `script` with a signature which it pushed onto the stack (last-in-first-out)
* It knows it's dealing with a p2wsh address and that the script seems valid
* It calcs the input's txid and acknowledges it w/ the vout index
* and then it shows us the script (one step per line) on the left with the initial stack from our txinwitness values.

Let's now walk thru, one step at a time (we'll just type: `step` and hit `<enter>` within the btcdeb session)

A compressed pubkey gets pushed onto the stack
```
		<> PUSH stack 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
OP_IFDUP                                                           | 30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525...
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0001 OP_CHECKSIG
```

Pubkey and Signature on stack are validated, with a successful `1` pushed on top
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d001
  pub key     = 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
  script code = 21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b268
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=25540109)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = f57f7d0003770163e77ece08591a2d1827caa21469f5092600ab721109d1ae4a
  pubkey.VerifyECDSASignature(sig=30440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d0, sighash=f57f7d0003770163e77ece08591a2d1827caa21469f5092600ab721109d1ae4a):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_IFDUP                                                           |                                                                 01
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0002 OP_IFDUP
```

Top value is true so it gets duplicated
```
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NOTIF                                                           |                                                                 01
OP_DUP                                                             |                                                                 01
OP_HASH160                                                         | 
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0003 OP_NOTIF
```

OP_NOTIF checks and consumes the top stack item, which is true. So execution of the recovery path will be ignored until OP_ENDIF
```
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_DUP                                                             |                                                                 01
OP_HASH160                                                         | 
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0004 OP_DUP

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_HASH160                                                         |                                                                 01
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0005 OP_HASH160

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
0a237e896c58bf43d6eded629a4b2abb0c0a9175                           |                                                                 01
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0006 0a237e896c58bf43d6eded629a4b2abb0c0a9175

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                                                     |                                                                 01
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0007 OP_EQUALVERIFY

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIGVERIFY                                                  |                                                                 01
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0008 OP_CHECKSIGVERIFY

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
24                                                                 |                                                                 01
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0009 24

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSEQUENCEVERIFY                                             |                                                                 01
OP_ENDIF                                                           | 
#0010 OP_CHECKSEQUENCEVERIFY

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_ENDIF                                                           |                                                                 01
#0011 OP_ENDIF

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 01
```
Since the script completed without failure and left a non-zero value on top of the stack... this transaction is valid.

---

## Recovery: Spend Funds

This time, we'll spend our only utxo to 2 outputs and we'll sign with the Recovery Key B.  Because this path has an OP_CSV,
(See [BIP-68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki)
and [BIP-112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) for how this works),
we must be careful setting the input's nSequence value to be greater-than-or-equal-to 36 (recall `older(36)` in our Miniscript descriptor).  When it's time to spend -- at least 36 blocks after our utxo's first confirmation, we'll also set our nLockTime to the current blockheight (or if pre-signing in advance, we'll extend the nLockTime appropriately).

```python
# assemble the inputs from our only utxo, note how much is spendable
>>> unspent = rpc.listunspent()[0]
>>> spendable = unspent['amount']

# WE MUST EXPLICITELY SET nSequence FOR OUR INPUT!
>>> txin = [dict(txid=unspent['txid'], vout=unspent['vout'], sequence=36)]

# assemble the outputs
>>> txout = []

# we'll need the internal wallet's receive descriptors
>>> rcv_desc = rpc.listdescriptors()['descriptors'][0]

# we'll send 50,000 sats to this wallet
>>> my_addr = rpc.deriveaddresses(rcv_desc['desc'], [rcv_desc['next'], rcv_desc['next']])[0]
>>> my_amt = Decimal("0.0005")
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
  'txid': '97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de',
  'vout': 0}]

>>> pprint(txout)
[{'tb1q02lgs98r36vpzpkpukndx45l00fyngjmt86dky9czygefcsfdunqhqr8sj': Decimal('0.0005')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00028750')}]

# since we're setting up in advance of when the recovery path can be spent
>>> unspent['confirmations']
28

# create the initial PSBT, pushing nLocktime another 8 blocks -- to 36 confirmations,
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount() + 8)

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAH0CAAAAAd6nf9pFhkXiiEeohEEz/tGefYrCHgbGbeqzsda8XPaXAAAAAAAkAAAAAlDDAAAAAAAAIgAger6IFOOOmBEGweWm01afe9JJoltZ9NsQuBERlOIJbyZOcAAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJkpEDAAABASuAOAEAAAAAACIAIOftChkCd4PbW1RHMFLESkwBnMIs95McdP6d4vpkzXFVAQVCIQIkp+SAiZoU6gFAZn7uHDlcXSEFG1R6E42vq1TMN/a00KxzZHapFOo6j3m0qWlrHx7XtV9/LxZPx097iK0BJLJoIgYCJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNAcB/2BbTAAAIABAACAAAAAgAIAAIAAAAAAAQAAACIGAsrWE3zEgozquhSCJ7EtOq6bRAp+Epy3maezJ4jz2S6tHNqFWh8wAACAAQAAgAAAAIACAACAAAAAAAEAAAAAAQFCIQOyONiZDcTrcyldfweQCRrTcVs+mg5u3w2hmAoCIjQ7uKxzZHapFIH7R0A04v7hlo1PkexwWpHK+v38iK0BJLJoIgICzHeiZjNdmGqMeCXtZG8fJq4bZqtJrRXqJ7drHJi2KDIc2oVaHzAAAIABAACAAAAAgAIAAIAAAAAAAgAAACICA7I42JkNxOtzKV1/B5AJGtNxWz6aDm7fDaGYCgIiNDu4HAf9gW0wAACAAQAAgAAAAIACAACAAAAAAAIAAAAAAA=='

# result of signing by external signer (in this case, krux)
>>> signed_psbt = "cHNidP8BAH0CAAAAAd6nf9pFhkXiiEeohEEz/tGefYrCHgbGbeqzsda8XPaXAAAAAAAkAAAAAlDDAAAAAAAAIgAger6IFOOOmBEGweWm01afe9JJoltZ9NsQuBERlOIJbyZOcAAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJkpEDAAABASuAOAEAAAAAACIAIOftChkCd4PbW1RHMFLESkwBnMIs95McdP6d4vpkzXFVIgICytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq1HMEQCIEVE0jprISpAt19DieGouhP8Wdwk/nUWMF6913dpkWKxAiBOY5wRH6P6oDnBy5APnr5uHuhwzoXfv2KXmA3AEFm2DQEBBUIhAiSn5ICJmhTqAUBmfu4cOVxdIQUbVHoTja+rVMw39rTQrHNkdqkU6jqPebSpaWsfHte1X38vFk/HT3uIrQEksmgAAAA="

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, signed_psbt])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# try to broadcast early...
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
  File "/home/smh/bclipy/jgarzik_bitcoinrpc_authproxy.py", line 143, in __call__
    raise JSONRPCException(response['error'])
jgarzik_bitcoinrpc_authproxy.JSONRPCException: -26: non-final

# Oooops, "non-final" means we have to wait till 36 confirmations, or block 233874, we wait...

>>> rpc.getblockcount()
233875

# since it was already complete, we can now broadcast it
if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'23302ee0025941a512742b9d1269a4ba326c2b257e8b7e1869aff76c4d065524'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/23302ee0025941a512742b9d1269a4ba326c2b257e8b7e1869aff76c4d065524)

---

## Recovery: Evolution of a PSBT

### `psbt = rpc.createpsbt(...`

```json
{
  "tx": {
    "txid": "23302ee0025941a512742b9d1269a4ba326c2b257e8b7e1869aff76c4d065524",
    "hash": "23302ee0025941a512742b9d1269a4ba326c2b257e8b7e1869aff76c4d065524",
    "version": 2,
    "size": 125,
    "vsize": 125,
    "weight": 500,
    "locktime": 233874,
    "vin": [
      {
        "txid": "97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de",
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
        "value": 0.00050000,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 7abe8814e38e981106c1e5a6d3569f7bd249a25b59f4db10b8111194e2096f26",
          "desc": "addr(tb1q02lgs98r36vpzpkpukndx45l00fyngjmt86dky9czygefcsfdunqhqr8sj)#vl8z9qc6",
          "hex": "00207abe8814e38e981106c1e5a6d3569f7bd249a25b59f4db10b8111194e2096f26",
          "address": "tb1q02lgs98r36vpzpkpukndx45l00fyngjmt86dky9czygefcsfdunqhqr8sj",
          "type": "witness_v0_scripthash"
        }
      },
      {
        "value": 0.00028750,
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
adds to `inputs`,
```json
      "witness_utxo": {
        "amount": 0.00080000,
        "scriptPubKey": {
          "asm": "0 e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
          "desc": "addr(tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u)#5tef56at",
          "hex": "0020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
          "address": "tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u",
          "type": "witness_v0_scripthash"
        }
      },
      "witness_script": {
        "asm": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1"
        },
        {
          "pubkey": "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```
adds to `outputs`,
```json
      "witness_script": {
        "asm": "03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8 OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 81fb474034e2fee1968d4f91ec705a91cafafdfc OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "2103b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8ac736476a91481fb474034e2fee1968d4f91ec705a91cafafdfc88ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "02cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2"
        },
        {
          "pubkey": "03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2"
        }
      ]
```
adds `fee`
```json
   ,
  "fee": 0.00001250
```

### `signed_psbt = ...`
Krux strips `bip32_derivs` from `inputs`, strips `witness_script` and `bip32_derivs` from `outputs`, and adds to `inputs`
```json
      "partial_signatures": {
        "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead": "304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd777699162b102204e639c111fa3faa039c1cb900f9ebe6e1ee870ce85dfbf6297980dc01059b60d01"
      },
```

### `combined_psbt = rpc.combinepsbt(...`
Since krux stripped some fields, Bitcoin Core replaces them,  To `inputs` it replaces:
```json
       ,
      "bip32_derivs": [
        {
          "pubkey": "0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1"
        },
        {
          "pubkey": "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```
... and to `outputs`, it replaces:
```json
      "witness_script": {
        "asm": "03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8 OP_CHECKSIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 81fb474034e2fee1968d4f91ec705a91cafafdfc OP_EQUALVERIFY OP_CHECKSIGVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "2103b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8ac736476a91481fb474034e2fee1968d4f91ec705a91cafafdfc88ad0124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "02cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2"
        },
        {
          "pubkey": "03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2"
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
Strips `witness_script` and `bip32_derivs`, and replaces `partial_signatures` in the `inputs` field w/
```json
      "final_scriptwitness": [
        "304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd777699162b102204e639c111fa3faa039c1cb900f9ebe6e1ee870ce85dfbf6297980dc01059b60d01",
        "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
        "",
        "210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268"
```

Final transaction:
```json
{
  "txid": "23302ee0025941a512742b9d1269a4ba326c2b257e8b7e1869aff76c4d065524",
  "hash": "5b780034f18a95298c7c27726d246121f2d773288f9aee8860287c8ce0c023b0",
  "version": 2,
  "size": 302,
  "vsize": 170,
  "weight": 677,
  "locktime": 233874,
  "vin": [
    {
      "txid": "97f65cbcd6b1b3ea6dc6061ec28a7d9ed1fe334184a84788e2458645da7fa7de",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd777699162b102204e639c111fa3faa039c1cb900f9ebe6e1ee870ce85dfbf6297980dc01059b60d01",
        "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
        "",
        "210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268"
      ],
      "sequence": 36
    }
  ],
  "vout": [
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 7abe8814e38e981106c1e5a6d3569f7bd249a25b59f4db10b8111194e2096f26",
        "desc": "addr(tb1q02lgs98r36vpzpkpukndx45l00fyngjmt86dky9czygefcsfdunqhqr8sj)#vl8z9qc6",
        "hex": "00207abe8814e38e981106c1e5a6d3569f7bd249a25b59f4db10b8111194e2096f26",
        "address": "tb1q02lgs98r36vpzpkpukndx45l00fyngjmt86dky9czygefcsfdunqhqr8sj",
        "type": "witness_v0_scripthash"
      }
    },
    {
      "value": 0.00028750,
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

The value and scriptPubKey from our existing utxo being spent:
```
    {
      "value": 0.00080000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
        "desc": "addr(tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u)#5tef56at",
        "hex": "0020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155",
        "address": "tb1qulks5xgzw7pakk65guc993z2fsqees3v77f3ca87nh305exdw92sapa96u",
        "type": "witness_v0_scripthash"
      }
    }
```

Starting `btcdeb`
```
btcdeb --tx=02000000000101dea77fda458645e28847a8844133fed19e7d8ac21e06c66deab3b1d6bc5cf6970000000000240000000250c30000000000002200207abe8814e38e981106c1e5a6d3569f7bd249a25b59f4db10b8111194e2096f264e70000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90447304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd777699162b102204e639c111fa3faa039c1cb900f9ebe6e1ee870ce85dfbf6297980dc01059b60d012102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead0042210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b26892910300 --txin=020000000001019a89da4317c748ddcd467ca360259efc2329cc4814b300353528d015ffbe71a38000000000fdffffff038038010000000000220020e7ed0a19027783db5b54473052c44a4c019cc22cf7931c74fe9de2fa64cd7155c092830100000000160014a47c08b9e9332577a4f1f4c9abae59c4aacd2e86b5e4000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac9024730440220741dfe9d4c678f064b764d6c7da5260e159a7a9a4a74556467e1525a33ef7fdb02201d954ee199da242733fadccb5b256fe62548402dba3924b5c6aef0674b1a01d0014221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac736476a9140a237e896c58bf43d6eded629a4b2abb0c0a917588ad0124b2686c910300

btcdeb 5.0.24 -- type `btcdeb -h` for start up options
LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 0; value = 80000
got witness stack of size 4
34 bytes (v0=P2WSH, v1=taproot/tapscript)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(97f65cbcd6, 0)
12 op script loaded. type `help` for usage information
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 |                                                                 0x
OP_CHECKSIG                                                        | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_IFDUP                                                           | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0000 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
```
Push a compressed pubkey onto stack
```
step
		<> PUSH stack 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        | 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
OP_IFDUP                                                           |                                                                 0x
OP_NOTIF                                                           | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_DUP                                                             | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
OP_HASH160                                                         | 
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0001 OP_CHECKSIG
```
check empty signature for pubkey, push 0 since it's not valid (and continues on)
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(0 len sig, 33 len pubkey, sigversion=1)
  sig         = 
  pub key     = 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
  script code = 210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268
- failed: signature is empty
		<> POP  stack
		<> POP  stack
		<> PUSH stack 
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_IFDUP                                                           |                                                                 0x
OP_NOTIF                                                           | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_DUP                                                             | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
OP_HASH160                                                         | 
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0002 OP_IFDUP
```
Do nothing, since the top of stack is false
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NOTIF                                                           |                                                                 0x
OP_DUP                                                             | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_HASH160                                                         | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0003 OP_NOTIF
```
Consume the top of stack, since it's false, will execute the recovery path to OP_ENDIF
```
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_DUP                                                             | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_HASH160                                                         | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0004 OP_DUP
```
Push the recovery's compressed pubkey
```
		<> PUSH stack 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_HASH160                                                         | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_EQUALVERIFY                                                     | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
OP_CHECKSIGVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0005 OP_HASH160
```
Calculate the digest of `ripemd160(sha256(B-pubkey))` and push it onto the stack
```
		<> POP  stack
		<> PUSH stack ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b                           |                           ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b
OP_EQUALVERIFY                                                     | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_CHECKSIGVERIFY                                                  | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0006 ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b
```
Push the hash (originally from inputs.witness-script) onto the stack
```
		<> PUSH stack ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                                                     |                           ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b
OP_CHECKSIGVERIFY                                                  |                           ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b
24                                                                 | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_CHECKSEQUENCEVERIFY                                             | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
OP_ENDIF                                                           | 
#0007 OP_EQUALVERIFY
```
Compare the calculated hash to the one from inputs, continue if equal, fail otherwise
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIGVERIFY                                                  | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
24                                                                 | 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd77...
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0008 OP_CHECKSIGVERIFY
```
Check recovery key and signature, leave nothing new on stack but fail if not valid
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd777699162b102204e639c111fa3faa039c1cb900f9ebe6e1ee870ce85dfbf6297980dc01059b60d01
  pub key     = 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
  script code = 210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac736476a914ea3a8f79b4a9696b1f1ed7b55f7f2f164fc74f7b88ad0124b268
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=80000)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = db64edc91e805fc4d1135f97c18ba7049fbcf4e8e03f5968d6d88b5203cbabf8
  pubkey.VerifyECDSASignature(sig=304402204544d23a6b212a40b75f4389e1a8ba13fc59dc24fe7516305ebdd777699162b102204e639c111fa3faa039c1cb900f9ebe6e1ee870ce85dfbf6297980dc01059b60d, sighash=db64edc91e805fc4d1135f97c18ba7049fbcf4e8e03f5968d6d88b5203cbabf8):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0009 24
```
push the nSequence (0x24 == 36) onto stack
```
		<> PUSH stack 24
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSEQUENCEVERIFY                                             |                                                                 24
OP_ENDIF                                                           | 
#0010 OP_CHECKSEQUENCEVERIFY
```
check all OP_CSV rules, fail if an error, else continue
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_ENDIF                                                           |                                                                 24
#0011 OP_ENDIF
```
End of script
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 24
```
Since the script completed without failure and left a non-zero value on top of the stack... this transaction is valid.
