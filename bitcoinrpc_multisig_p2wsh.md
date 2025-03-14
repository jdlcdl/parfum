# Bitcoin Core Watch-Only 2-of-3 Multisig Native Segwit

Exploring how to use Bitcon Core as a Watch-Only wallet, accessed via
[python-bitcoinrpc](https://github.com/jgarzik/python-bitcoinrpc),
"glued" w/
[SeedQReader](https://github.com/pythcoiner/SeedQReader)
for air-gapped QR-Code signing w/ stateless devices like
[Krux](https://github.com/selfcustody/krux)
and
[SeedSigner](https://github.com/SeedSigner/seedsigner)

Note: The sample python sessions below have been done on `signet`, a bitcoin testnet where coins have no value.

---

## Table of Contents
* [Secrets](#secrets): these would NEVER go online. They're cold-storage backups to be kept secure, used by a stateless signer.
* [Extended Public Keys](#extended-public-keys): The `privacy-sensitive` public portion of keys.
* [Output Descriptor](#2-of-3-multisig-output-descriptor): Used to create and restore a watch-only wallet, THIS MUST BE BACKED-UP.
* [Watch-Only Wallet](#watch-only-wallet): Used to get new addresses, watch for UTXOs, propose and coordinate PSBTs.
* [Spend Funds](#spend-funds): Create, update, sign, combine, finalize, and extract a PSBT, then broadcast it.
* [Evolution of a PSBT](#evolution-of-a-psbt): How a PSBT evolves - from creation to broadcast.
* [Understanding Programmable Money](#understanding-programmable-money): stepping thru validation of this transaction in btcdeb.

---

## Secrets

**BIP-39 mnemonics**:
* **A**: `auction crucial trend safe faith barrel orbit roast source stereo discover cart`
* **B**: `orange enter age rug chef denial legend topic identify sign always mother`
* **C**: `news lecture under adapt inspire chunk tongue fun party build defense receive`
![secrets-2of3](https://gist.github.com/user-attachments/assets/9a36564b-57d0-4f12-9012-36a45c6ee33a)


**BIP-39 passphrases**: ""

---

## Extended Public Keys

**BIP-32 master fingerprints**: 
* **A**: `07fd816d`
* **B**: `da855a1f`
* **C**: `cdef7cd9`

**Multisig Native Segwit Derivation Path**: `m/48'/1'/0'/2'`

**xpubs**
* **A**:
```
tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5
```
* **B**:
```
tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5
```
* **C**:
```
tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2
```

**xpubs w/ key-origin**:
* **A**:
```
[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5
```
* **B**:
```
[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5
```
* **C**:
```
[cdef7cd9/48H/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2
```

Note: Above, the signers exported xpubs w/ key-origin in two different formats. Both A and C use '`'` as the hardened derivation indicator while B uses `h`.  Bitcoin core supports both `h` and `'` however BIP-174 also includes `H` as an alternative hardened indicator -- which Bitcoin Core does NOT support.  All hardened indicators have the same meaning (add 2^31 to this node so that only an Extended Public Key can NOT derive children) but support for these `expressions` are not always fully supported across all tools.

---

## 2-of-3 Multisig Output Descriptor:

Using the xpub as a basis to the descriptor,
* prefix each of the xpubs with key-origin info, wrapped in square brackets
* append `/<0;1>/*` to the end of each xpub (as the BIP-389 receive and change address path)
* using `,` as a separator, join each of the xpub expressions with a comma
* wrap above with a prefix of `wsh(sortedmulti(2,` and a suffix of `))` (as the BIP-48 multisig native segwit script wrapper)
* append the BIP-380 descriptor checksum prefixed by `#`

```python
# start with a list of xpubs w/ key-origin information
>>> xpubs = [
        "[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5",
        "[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5",
        "[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2",
    ]

# append the receive/change path
>>> xpubs = [x + "/<0;1>/*" for x in xpubs]

# wrap above with wsh script for a sorted 2-of-n multisig, wth comma separated xpubs
>>> descriptor = "wsh(sortedmulti(2," + ",".join(xpubs) + "))"

# append the checksum
>>> import bip380_checksum
>>> descriptor = bip380_checksum.descsum_create(descriptor)

>>> print(repr(descriptor))
"wsh(sortedmulti(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*))#2n8mk54n"
```
![descriptor-2of3](https://gist.github.com/user-attachments/assets/284ff092-6f7c-4568-b539-1a11c66707a0)


Note: `sortedmulti()` has a first paramater of `2` for this `2-of-3` multisig, followed by all 3 xpubs.  Sorted multi means that these xpubs may be presented in any order and that derived pubkeys will always be presented in a deterministic `sorted` order within bitcoin script.  That is, we could re-create another descriptor, presenting the xpubs ordered like `B,C,A` or any other order and we'd still be restoring the same wallet with the same addresses.  If however, we had used `multi()` instead, then we could NOT restore the same wallet without presenting the xpubs in same order as orginal; any different order would be a completely different wallet with its own addresses.

### Split the descriptor into receive and change `descriptors list` for Bitcoin Core

```python
>>> descriptors = [
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/0/*").split("#")[0]
    ),
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/1/*").split("#")[0]
    )]

>>> print(repr(descriptors))
["wsh(sortedmulti(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*))#lgthxqwq", "wsh(sortedmulti(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*,[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/1/*))#6aqdcx7g"]
```

---

## Watch-Only Wallet

Instead of using `bitcoin-cli` at the command line, we'll be connecting to Bitcoin Core with 
python-bitcoinrpc as is done in this [BlockchainCommons tutorial](https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/master/18_4_Accessing_Bitcoind_with_Python.md).  If you're more familiar with `bitcoin-cli`, then know that all `rpc.method_name()` calls below can be made like `bitcoin-cli command` with similar paramaters.

In the examples below, `rpc.` is an authproxy object that is still connected to `bitcoind` -- thanks to an exaggerated configuration setting `rpcservertimeout=600` so that connections remain open for 10m, instead of closing once idle for 30s.

```python
# create the wallet
>>> rpc.createwallet(
    "multi-p2wsh",  # wallet_name
    True,  # disable_private_keys
    True,  # blank
    "",  # passphrase
    True,  # avoid_reuse
    True,  # descriptors
    False,  # load_on_startup
    False)  # external_signer
{'name': 'multi-p2wsh', 'warnings': ['Empty string given as passphrase, wallet will not be encrypted.']}

# import its descriptors
>>> rpc.importdescriptors([dict(desc=x, timestamp="now") for x in descriptors])
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]

# view the wallet
>>> pprint(rpc.getwalletinfo())
{'avoid_reuse': True,
 'balance': Decimal('0E-8'),
 'birthtime': 1738266906,
 'blank': True,
 'descriptors': True,
 'external_signer': False,
 'format': 'sqlite',
 'immature_balance': Decimal('0E-8'),
 'keypoolsize': 0,
 'keypoolsize_hd_internal': 0,
 'lastprocessedblock': {'hash': '000000527d75ef1dd5ae72d0dfe28a29b93692d61208a93ea47659d1eb9035d7',
                        'height': 233303},
 'paytxfee': Decimal('0E-8'),
 'private_keys_enabled': False,
 'scanning': False,
 'txcount': 0,
 'unconfirmed_balance': Decimal('0E-8'),
 'walletname': 'multi-p2wsh',
 'walletversion': 169900}

# view first 10 receive addresses
>>> pprint(rpc.deriveaddresses(descriptors[0], 9))
['tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn',
 'tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z',
 'tb1qvmweuz9kxcwfh2smrlg60szepzjs6gl3a07y8qwatk560se9trus6sk4xw',
 'tb1qe55sdw2uut47ndx6pkwua8a4ry5p506jq5mpmha8gnxdfkwt0d4swea3v3',
 'tb1q0nr5n6vfjtv6vq6h93s8rwz5zlvmsgj4eh9mu9c6xfvuzdmy4yjqrvx8ck',
 'tb1q3h5q9gdsc5a9twjyxukvwlhqf9t5y827xtkdxqnrxkqcw09xdxdqdqpmu2',
 'tb1q2al0jmq8vmtpuj4vg9pk6v5a8qkm03kv74ku6c43lxp0kkk6gknswuktme',
 'tb1qamltz28wcy3hyqkzz572elvdv62jteyh86k65sf8s4rpdthez3eq443a09',
 'tb1qrspln3gm27llvwm5cqg4udvq2het4gmzy4ztlzar2t2w5645z7ysrlq8kn',
 'tb1q0lxzcwwfr37eysmdphn02lhqqt3knx6lm3c4jfm2rm0uqw9q459s0mt0na']

# view first 10 change addresses
>>> pprint(rpc.deriveaddresses(descriptors[1], 9))
['tb1q6gakuxqdpy0r567gg7ehvajse33cdn48h8g437dyu59tejmzr4eqqulwfy',
 'tb1q9cf3xqgrluexsufsrr2l6cgvp63l8n3latx435uljemmxd9dhmksn3ejtl',
 'tb1qgmq598karavlae72xnz4lw3zsa5j58g66rt3x5ctytyfntrwflus9ceetn',
 'tb1qrsauvsenapumpa2837hu8apndmejvp7hys9vczqmgzxh9ux7pf6qmfw3tf',
 'tb1qp2zmvfdqvez6klqsuetjaln04muexx99w28mm77jkfwulns04meq26h4ra',
 'tb1qfcxzs2ztcqg4y6g5c5qz9r2kvyphlugmduu46wz0fg7gumgmqnqq4s05ep',
 'tb1qhl444pt7evnxurwg0crkdcvclj6pnuxu05jt2ykmpwnwk74gyvcsejrj8a',
 'tb1qlcmxvva3kwcqks0r7qqqhg3j2lmt7xwxsj5fzwf7tchktlv6uqeq6lqqpp',
 'tb1q2td3de9s2cq28rvs3u090gxw6c5jmkp0jdxtxvyaree3ry2qjdqs4ymw20',
 'tb1qkgjw40l9vu0hmqq4xdhxqa7jpp3genyhnmgwcu29tj7uwkmeg3rquj8sdq']

# As seen in a previous exercise -- Bitcoin Core alters descriptors to use `h` hardened
# indicators in the key-origin.  Still receive and change addresses remain the same, 
# as we'll verify below:
>>> rpc.listdescriptors()
{'wallet_name': 'multi-p2wsh', 'descriptors': [{'desc': 'wsh(sortedmulti(2,[07fd816d/48h/1h/0h/2h]tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*,[cdef7cd9/48h/1h/0h/2h]tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*))#t6rtvud8', 'timestamp': 1738266906, 'active': False, 'range': [0, 999], 'next': 0, 'next_index': 0}, {'desc': 'wsh(sortedmulti(2,[07fd816d/48h/1h/0h/2h]tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*,[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*,[cdef7cd9/48h/1h/0h/2h]tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/1/*))#w0g3j6a0', 'timestamp': 1738266906, 'active': False, 'range': [0, 999], 'next': 0, 'next_index': 0}]}

>>> pprint(rpc.deriveaddresses(rpc.listdescriptors()['descriptors'][0]['desc'], 2))
['tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn',
 'tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z',
 'tb1qvmweuz9kxcwfh2smrlg60szepzjs6gl3a07y8qwatk560se9trus6sk4xw']

>>> pprint(rpc.deriveaddresses(rpc.listdescriptors()['descriptors'][1]['desc'], 2))
['tb1q6gakuxqdpy0r567gg7ehvajse33cdn48h8g437dyu59tejmzr4eqqulwfy',
 'tb1q9cf3xqgrluexsufsrr2l6cgvp63l8n3latx435uljemmxd9dhmksn3ejtl',
 'tb1qgmq598karavlae72xnz4lw3zsa5j58g66rt3x5ctytyfntrwflus9ceetn']

#
# funded first receive address from alt.signetfaucet.com
# recycle-to: tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn
#

# view UTXOs
>>> pprint(rpc.listunspent())
[{'address': 'tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn',
  'amount': Decimal('0.05555555'),
  'confirmations': 1,
  'desc': 'wsh(multi(2,[cdef7cd9/48h/1h/0h/2h/0/0]029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c,[07fd816d/48h/1h/0h/2h/0/0]033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b,[da855a1f/48h/1h/0h/2h/0/0]0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6))#gaa3erd8',
  'parent_descs': ["wsh(sortedmulti(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48h/1h/0h/2h]tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*))#lgthxqwq"],
  'reused': False,
  'safe': True,
  'scriptPubKey': '002099e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb',
  'solvable': True,
  'spendable': True,
  'txid': '235f0250fd3925ea81bac13fdbf5e0dd95c7be1f53bf7ce3432447b4b0ebf23f',
  'vout': 6,
  'witnessScript': '5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae'}]
```

---

## Spend Funds

We'll go from start to finish, spending a single UTXO to three outputs.  We'll `cut` from our python-bitcoinrpc session and `paste` into SeedQReader to create QR-Codes for importing into our stateless air-gapped Signers.  We'll also use SeedQReader to read the signed PSBTs from our Signers, `cutting` and `pasting` them back into our python-bitcoinrpc session.

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
>>> my_amt = Decimal(".0005")
>>> txout.append({my_addr: my_amt})
>>> spendable -= my_amt

# we'll also send 5,470,000 sats to an external wallet
>>> send_addr = "tb1qvt3sv69hlq269gp737kf593243hxyu7ssp993a"
>>> send_amt = Decimal("0.0547")
>>> txout.append({send_addr: send_amt})
>>> spendable -= send_amt

# we'll send the rest back to the faucet recycle address, assuming no fees yet
>>> faucet_addr = "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn"
>>> faucet_amt = spendable
>>> txout.append({faucet_addr: spendable})
>>> spendable -= faucet_amt

# we'll pay TODO sats per vbyte
>>> fee_rate = Decimal("0.00000015")
>>> vsize = rpc.decodepsbt(rpc.createpsbt(txin, txout))['tx']['vsize']
>>> fees = vsize * fee_rate
>>> faucet_amt -= fees
>>> spendable += fees
>>> txout[-1] = {faucet_addr: faucet_amt}
>>> assert my_amt + send_amt + faucet_amt + fees == unspent['amount']

>>> pprint(txin)
[{'txid': '235f0250fd3925ea81bac13fdbf5e0dd95c7be1f53bf7ce3432447b4b0ebf23f',
  'vout': 6}]

>>> pprint(txout)
[{'tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z': Decimal('0.0005')},
 {'tb1qvt3sv69hlq269gp737kf593243hxyu7ssp993a': Decimal('0.0547')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00033215')}]

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAJwCAAAAAT/y67C0RyRD43y/Ux++x5Xd4PXbP8G6geolOf1QAl8jBgAAAAD9////A1DDAAAAAAAAIgAgt3MMEFB4Cq7ByUyGrv7p9UcwjW8iYrvBRTs2xXEnFDIwd1MAAAAAABYAFGLjBmi3+BWioD6PrJoWKqxuYnPQv4EAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6ybuPAwAAAQErY8VUAAAAAAAiACCZ4vdsfVjagXIBwMsE+RxhrAWCSJG06p1ZJ78iOidaywEFaVIhAp3QRfho/PjPoNGU+f7JkNfQvfAdKkLklOh27LbdSC9cIQM8ND4GOBt+eSb9vlTGOrAEDFwshJHi1L1yOMuQZgQKGyEDRgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399ZTriIGAp3QRfho/PjPoNGU+f7JkNfQvfAdKkLklOh27LbdSC9cHM3vfNkwAACAAQAAgAAAAIACAACAAAAAAAAAAAAiBgM8ND4GOBt+eSb9vlTGOrAEDFwshJHi1L1yOMuQZgQKGxwH/YFtMAAAgAEAAIAAAACAAgAAgAAAAAAAAAAAIgYDRgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399Yc2oVaHzAAAIABAACAAAAAgAIAAIAAAAAAAAAAAAABAWlSIQIkp+SAiZoU6gFAZn7uHDlcXSEFG1R6E42vq1TMN/a00CECytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq0hA/jJrzFOdG6Zs5e67n/W7j1ZBBM7WgeIO4qfNucl70w8U64iAgIkp+SAiZoU6gFAZn7uHDlcXSEFG1R6E42vq1TMN/a00BwH/YFtMAAAgAEAAIAAAACAAgAAgAAAAAABAAAAIgICytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq0c2oVaHzAAAIABAACAAAAAgAIAAIAAAAAAAQAAACICA/jJrzFOdG6Zs5e67n/W7j1ZBBM7WgeIO4qfNucl70w8HM3vfNkwAACAAQAAgAAAAIACAACAAAAAAAEAAAAAAAA='

# result of signing by krux
>>> psbtA = "cHNidP8BAJwCAAAAAT/y67C0RyRD43y/Ux++x5Xd4PXbP8G6geolOf1QAl8jBgAAAAD9////A1DDAAAAAAAAIgAgt3MMEFB4Cq7ByUyGrv7p9UcwjW8iYrvBRTs2xXEnFDIwd1MAAAAAABYAFGLjBmi3+BWioD6PrJoWKqxuYnPQv4EAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6ybuPAwAAAQErY8VUAAAAAAAiACCZ4vdsfVjagXIBwMsE+RxhrAWCSJG06p1ZJ78iOidayyICAzw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobRzBEAiA+Jqvp6Gc+ZDGw64VqR1V3swJqfSEiL7MlyIYvOyfOSgIgGPPHsZY6IiEZb0FmCiCtR8zEoYBZS8t7ItjXsdF/iX0BAQVpUiECndBF+Gj8+M+g0ZT5/smQ19C98B0qQuSU6Hbstt1IL1whAzw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobIQNGAjt4cuACdXfYpFE4m8WkMOcp8tOJoj7MoSN9w7f31lOuAAAAAA=="

# result of signing by seedsigner
>>> psbtB = "cHNidP8BAJwCAAAAAT/y67C0RyRD43y/Ux++x5Xd4PXbP8G6geolOf1QAl8jBgAAAAD9////A1DDAAAAAAAAIgAgt3MMEFB4Cq7ByUyGrv7p9UcwjW8iYrvBRTs2xXEnFDIwd1MAAAAAABYAFGLjBmi3+BWioD6PrJoWKqxuYnPQv4EAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6ybuPAwAAIgIDRgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399ZHMEQCIA3qjJPtULIbItl6jJqnPOHfdNUs+FdOYcheYtzOPCiLAiAC4ifiG0yJSH4exlChAnhPrrtqRM4NomGQF3dsan/jegEAAAAA"

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, psbtA, psbtB])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'628f9dab6699775271ce5ceec1e1277665b1890a628802068d2d01d03a6b23e1'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/628f9dab6699775271ce5ceec1e1277665b1890a628802068d2d01d03a6b23e1)

---

## Evolution of a PSBT

In the above session, the PSBT evolves with each step. Below these changes are explained.

### `psbt = rpc.createpsbt(...`
Bitcoin Core creates the initial psbt template, knowing only the input's `txid` and `vout`, but including no information about what that input is worth or what the spending conditions might be.  Note: because this is segwit, the final pre-segwit `txid` is already known, but not the `hash`.
```json
{
  "tx": {
    "txid": "628f9dab6699775271ce5ceec1e1277665b1890a628802068d2d01d03a6b23e1",
    "hash": "628f9dab6699775271ce5ceec1e1277665b1890a628802068d2d01d03a6b23e1",
    "version": 2,
    "size": 156,
    "vsize": 156,
    "weight": 624,
    "locktime": 233403,
    "vin": [
      {
        "txid": "235f0250fd3925ea81bac13fdbf5e0dd95c7be1f53bf7ce3432447b4b0ebf23f",
        "vout": 6,
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
          "asm": "0 b7730c1050780aaec1c94c86aefee9f547308d6f2262bbc1453b36c571271432",
          "desc": "addr(tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z)#33t8hezt",
          "hex": "0020b7730c1050780aaec1c94c86aefee9f547308d6f2262bbc1453b36c571271432",
          "address": "tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z",
          "type": "witness_v0_scripthash"
        }
      },
      {
        "value": 0.05470000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 62e30668b7f815a2a03e8fac9a162aac6e6273d0",
          "desc": "addr(tb1qvt3sv69hlq269gp737kf593243hxyu7ssp993a)#5h7lftx3",
          "hex": "001462e30668b7f815a2a03e8fac9a162aac6e6273d0",
          "address": "tb1qvt3sv69hlq269gp737kf593243hxyu7ssp993a",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00033215,
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
        "amount": 0.05555555,
        "scriptPubKey": {
          "asm": "0 99e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb",
          "desc": "addr(tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn)#sy8k2amh",
          "hex": "002099e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb",
          "address": "tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn",
          "type": "witness_v0_scripthash"
        }
      },
      "witness_script": {
        "asm": "2 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 3 OP_CHECKMULTISIG",
        "hex": "5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0"
        },
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
It also adds information pertaining to any of the outputs which are known to belong to the descriptors.  In this case, it fills the first of three empty dictionaries with the following content -- within the `outputs` list.
```json
      "witness_script": {
        "asm": "2 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c 3 OP_CHECKMULTISIG",
        "hex": "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead2103f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c53ae",
        "type": "multisig"
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
        },
        {
          "pubkey": "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```
Last of all, since the value of the inputs is now known, it can finally calculate and append the `fee`.
```json
   ,
  "fee": 0.00002340
```

### `psbtA = ...` and `psbtB = ...`
It is the job of the Signer to display pertinent information to the user so that they understand what is being signed, and to add `partial_signatures` to the PSBT.  In this case, Krux adds a signature for Key A to the existing dictionary in `inputs`
```json
      "partial_signatures": {
        "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b": "304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d01"
      },
```
...and SeedSigner adds a signature for Key B to `inputs`
```json
      "partial_signatures": {
        "0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6": "304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a01"
      }
```
The Signer's role in the PSBT is simply to "add" signatures, and done.  In fact,
[BIP-174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki#signer)
clearly states: "The Signer must only add data to a PSBT.", but some rules were meant to be broken?!?!

Because an air-gapped Signer which transmits data via QR-Code is constrained by this low-bandwidth medium, some have "optimized" to transfer less -- by stripping unnecessary fields from `inputs` and `outputs`.  Krux strips `witness_script` and `bip32_derivs` from both `inputs` and `outputs`, while SeedSigner does the same and also strips `witness_utxo` from `inputs`.

This is a non-standard `feature` of some air-gapped-via-qrcode Signers which is NOT practiced when a signed PSBT can be written efficiently to an sdcard or transfered electronically.  As expected, this non-standard feature must also be tolerated by whichever software is acting in the role of "Combiner", as we'll see below.

### `combined_psbt = rpc.combinepsbt(...`
Had the air-gapped signers not stripped any fields in their quest for an optimized transmit-via-qrocde user-experience, the resulting signed `psbtA` and `psbtB` would have been much like the the `combined_psbt`.  However, because multiple signatures from multiple Signers may be signed in parallel, instead of in series adding signatures one after the next, it is important for the Combiner to consider combining all signed PSBTs with the original PSBT distributed to Signers.  In this case, Bitcoin Core simply replaces the dictionary keys that had been stripped out.

Recall that SeedSigner had stripped `witness_utxo` from `inputs`, therefore Bitcoin Core restores it,
```json
      "witness_utxo": {
        "amount": 0.05555555,
        "scriptPubKey": {
          "asm": "0 99e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb",
          "desc": "addr(tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn)#sy8k2amh",
          "hex": "002099e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb",
          "address": "tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn",
          "type": "witness_v0_scripthash"
        }
      },
```
...and both Krux and SeedSigner had also stripped `witness_script` and `bip32_derivs` from `inputs`, which gets restored.
```json
      "witness_script": {
        "asm": "2 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 3 OP_CHECKMULTISIG",
        "hex": "5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0"
        },
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
Lastly, Bitcoin Core restores content in the first dictionary that had been stripped from the `outputs` field.
```json
      "witness_script": {
        "asm": "2 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c 3 OP_CHECKMULTISIG",
        "hex": "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead2103f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c53ae",
        "type": "multisig"
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
        },
        {
          "pubkey": "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
As the Finalizer role, Bitcoin Core is tasked to determine whether or not the PSBT is complete and final, by verifying that spending conditions for all inputs have been met.  In this case, finalizing the complete PSBT strips-out previously added `witness_script` and `bip32_derivs` and replaces `partial_signatures` with `final_scriptwitness` within the `inputs` field.
```json
      "final_scriptwitness": [
        "",
        "304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d01",
        "304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a01",
        "5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae"
```

By default, when the PSBT is complete and final, Bitcoin Core's `finalizepsbt` will also extract the final rawtransaction hex so that it's ready to be broadcasted into the mempool.  Once extracted, the transaction's `hash` is finally known, while the pre-segwit `txid` had never changed.

```json
{
  "txid": "628f9dab6699775271ce5ceec1e1277665b1890a628802068d2d01d03a6b23e1",
  "hash": "07c423744f7ece03fee583a625a5b480eff2f4b8a1b58a151771f4fa15d4bdd5",
  "version": 2,
  "size": 410,
  "vsize": 220,
  "weight": 878,
  "locktime": 233403,
  "vin": [
    {
      "txid": "235f0250fd3925ea81bac13fdbf5e0dd95c7be1f53bf7ce3432447b4b0ebf23f",
      "vout": 6,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "",
        "304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d01",
        "304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a01",
        "5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae"
      ],
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 b7730c1050780aaec1c94c86aefee9f547308d6f2262bbc1453b36c571271432",
        "desc": "addr(tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z)#33t8hezt",
        "hex": "0020b7730c1050780aaec1c94c86aefee9f547308d6f2262bbc1453b36c571271432",
        "address": "tb1qkaescyzs0q92aswffjr2alhf74rnprt0yf3ths298vmv2uf8zseqa6y94z",
        "type": "witness_v0_scripthash"
      }
    },
    {
      "value": 0.05470000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 62e30668b7f815a2a03e8fac9a162aac6e6273d0",
        "desc": "addr(tb1qvt3sv69hlq269gp737kf593243hxyu7ssp993a)#5h7lftx3",
        "hex": "001462e30668b7f815a2a03e8fac9a162aac6e6273d0",
        "address": "tb1qvt3sv69hlq269gp737kf593243hxyu7ssp993a",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 0.00033215,
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

## Understanding Programmable Money

We'll use [btcdeb](https://github.com/bitcoin-core/btcdeb), a bitcoin script debugger, to understand how validation of the above transaction works.

In a nutshell, spending programmable money is about the owner providing bitcoin script "inputs" that will be completed by the utxo's bitcoin scriptPubKey, completing without error and leaving a True value on top of the stack. It's necessary to know about the scriptPubKey that is part of the address from the input utxo -- whose source was the input transaction and which is stored within each node's utxoset.  Below is our input (the seventh output and utxo of our alt-signet-faucet funding transaction):
```
    {
      "value": 0.05555555,
      "n": 6,
      "scriptPubKey": {
        "asm": "0 99e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb",
        "desc": "addr(tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn)#sy8k2amh",
        "hex": "002099e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb",
        "address": "tb1qn830wmratrdgzuspcr9sf7guvxkqtqjgjx6w482ey7ljyw38tt9svztqsn",
        "type": "witness_v0_scripthash"
      }
    },
```

Below, we will start a btcdeb session at the bash commandline using the rawtransaction hex of the above transaction, as well as the same for this transaction's only input.
```
btcdeb --quiet --tx=020000000001013ff2ebb0b4472443e37cbf531fbec795dde0f5db3fc1ba81ea2539fd50025f230600000000fdffffff0350c3000000000000220020b7730c1050780aaec1c94c86aefee9f547308d6f2262bbc1453b36c571271432307753000000000016001462e30668b7f815a2a03e8fac9a162aac6e6273d0bf81000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac9040047304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d0147304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a01695221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653aebb8f0300 --txin=03000000000101e90c472e3e99e7d04b6aa36639fc9ef70f04a1c147ba3c20de5434e7fbb729bf0000000000fdffffff0a6973916337000000225120aac35fe91f20d48816b3c83011d117efa35acd2414d36c1e02b0f29fc3106d9063c55400000000001600144d2e27755127cf47f9763bd4650e143caaeee2e463c5540000000000160014bd799512b2b9880f8b46883ffcf31e17b5632a3f63c5540000000000225120239531dbea02747d8936587d441afbc0c56e82760c9eb433a0297ca9980c3f4e63c5540000000000160014d900297c3cd936828c012dac3b994321aae5a6c863c5540000000000160014d3f617675900a78409d5d498748bf14f4d779e6c63c554000000000022002099e2f76c7d58da817201c0cb04f91c61ac05824891b4ea9d5927bf223a275acb63c5540000000000225120adffe9ff326cb67d61676e275a7847bb06e58e85d627b6d60d2c01cae37017f763c55400000000001600148b3ae50f61c86a71f92d22ce208ff59f3ae34f8363c55400000000002251207acb7cbb8251fccbbb7c9b0b2f2ea1a29ee536c02c8f9f37041ef6570368eab801404ac25fcb8ed1b9a0bd25ee86a3a01ba8e0cf2f303199149b17d39494e351761e02f6663b799e9f7cc3c0fda6b877c4b0eee40f33a7944dcc78903eb9eb122c6b578f0300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 6; value = 5555555
got witness stack of size 4
34 bytes (v0=P2WSH, v1=taproot/tapscript)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(235f0250fd, 6)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  | 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62d...
029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c | 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862...
033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b |                                                                 0x
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 | 
3                                                                  | 
OP_CHECKMULTISIG                                                   | 
#0000 2
```
Even though we used --quiet, we have plenty of output.

* we're signing segwit or taproot
* btcdeb will work one input at a time, and it knows the vout and the value in sats
* It knows that signing this transaction initialized the script with two values which it pushed onto the stack (last-in-first-out)
* It knows it's dealing with a p2wsh address and that the script seems valid
* It calcs the input's txid and acknowledges it w/ the vout index
* and then it shows us the script (one step per line) on the left with the initial stack from our txinwitness values.

Let's now walk thru, one step at a time (we'll just type: `step` and hit `<enter>` within the btcdeb session)

The first step will push `2` (as in `2`-of-3 multisig) onto the top of the stack
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c |                                                                 02
033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b | 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62d...
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 | 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862...
3                                                                  |                                                                 0x
OP_CHECKMULTISIG                                                   | 
#0001 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c
```
The next 3 steps will push pubkeys (note, in `sortedmulti` order), one at a time onto the top of the stack

The first pubkey is pushed, from Key C's first receive address -- which just-so-happens to sort as the first in this set
```
		<> PUSH stack 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b | 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 |                                                                 02
3                                                                  | 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62d...
OP_CHECKMULTISIG                                                   | 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862...
                                                                   |                                                                 0x
#0002 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
```

The second pubkey is pushed, from Key A's first receive address -- which just-so-happens to sort as the second this time
```
		<> PUSH stack 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
3                                                                  | 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c
OP_CHECKMULTISIG                                                   |                                                                 02
                                                                   | 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62d...
                                                                   | 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862...
                                                                   |                                                                 0x
#0003 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
```

The last pubkey is pushed, from Key B's first receive address -- which just-so-happens to sort last
```
		<> PUSH stack 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
3                                                                  | 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
OP_CHECKMULTISIG                                                   | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
                                                                   | 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c
                                                                   |                                                                 02
                                                                   | 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62d...
                                                                   | 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862...
                                                                   |                                                                 0x
#0004 3
```

Next, a `3` (as in 2-of-`3` sortedmulti) is pushed on top of the stack
```
		<> PUSH stack 03
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKMULTISIG                                                   |                                                                 03
                                                                   | 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
                                                                   | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
                                                                   | 029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c
                                                                   |                                                                 02
                                                                   | 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62d...
                                                                   | 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862...
                                                                   |                                                                 0x
#0005 OP_CHECKMULTISIG
```

finally, OP_CHECKMULTISIG will validate that 2 signatures for 3 of the pubkeys in the stack are valid, pushing `1` for success or `0` for failure.
```
stack has 8 entries [require 1]
stack has 8 entries [require 5]
stack has 8 entries [require 8]
scriptCode = 5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae
looping for multisig
loop: sigs = 2, keys = 3
- got sig 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a01
- got key 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a01
  pub key     = 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
  script code = 5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=5555555)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = 9be8948ee28dae005a5139fd112e603a20bbc3ab81414dbca3d497d81dfe2712
  pubkey.VerifyECDSASignature(sig=304402200dea8c93ed50b21b22d97a8c9aa73ce1df74d52cf8574e61c85e62dcce3c288b022002e227e21b4c89487e1ec650a102784faebb6a44ce0da2619017776c6a7fe37a, sighash=9be8948ee28dae005a5139fd112e603a20bbc3ab81414dbca3d497d81dfe2712):
  result: success
- sig check succeeded
loop: sigs = 1, keys = 2
- got sig 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d01
- got key 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d01
  pub key     = 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
  script code = 5221029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c21033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d653ae
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=5555555)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = 9be8948ee28dae005a5139fd112e603a20bbc3ab81414dbca3d497d81dfe2712
  pubkey.VerifyECDSASignature(sig=304402203e26abe9e8673e6431b0eb856a475577b3026a7d21222fb325c8862f3b27ce4a022018f3c7b1963a2221196f41660a20ad47ccc4a180594bcb7b22d8d7b1d17f897d, sighash=9be8948ee28dae005a5139fd112e603a20bbc3ab81414dbca3d497d81dfe2712):
  result: success
- sig check succeeded
loop ended in successful state
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 01
```
Since the script completed without failure and left a non-zero value on top of the stack... this transactioin is valid.
