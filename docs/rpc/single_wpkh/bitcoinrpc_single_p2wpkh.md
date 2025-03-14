# Bitcoin Core Watch-Only Single-sig Native Segwit

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
* [Extended Public Key](#extended-public-key): The `privacy-sensitive` public portion of keys.
* [Output Descriptor](#output-descriptor): Used to create and restore a watch-only wallet.
* [Watch-Only Wallet](#watch-only-wallet): Used to get new addresses, watch for UTXOs, propose and coordinate PSBTs.
* [Fun-Fact](#fun-fact): Bitcoin Core validated the descriptors, then altered them.
* [Spend Funds](#spend-funds): Create, update, sign, combine, finalize, and extract a PSBT, then broadcast it.
* [Evolution of a PSBT](#evolution-of-a-psbt): How a PSBT evolves - from creation to broadcast.
* [Understanding Programmable Money](#understanding-programmable-money): stepping thru validation of this transaction in `btcdeb`.

---

## Secrets

**BIP-39 mnemonic**: `auction crucial trend safe faith barrel orbit roast source stereo discover cart`
![07fd816d-mnemonic](07fd816d-mnemonic.png)


**BIP-39 passphrase**: ""

---

## Extended Public Key

**BIP-32 master fingerprint**: `07fd816d`

**Native Segwit Derivation Path**: `m/84'/1'/0'`

**Key-Origin**: `[07fd816d/84'/1'/0']`

**xpub**:
```
tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G`
```

**xpub w/ key-origin**:
```
[07fd816d/84'/1'/0']tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G
```
![07fd816d-natsw-xpub](07fd816d-natsw-xpub.png)

---

## Output Descriptor:

Using the xpub as a basis to the descriptor,
* prefix the xpub with key-origin info, wrapped in square brackets
* append `/<0;1>/*` to the end of the xpub (as the BIP-389 receive and change address path)
* wrap above with a prefix of `wpkh(` and a suffix of `)` (as the BIP-84 native segwit script wrapper)
* append the BIP-380 descriptor checksum prefixed by `#`

```python
>>> descriptor = "".join([
    "wpkh(",
    "[07fd816d/84'/1'/0']",
    "tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G",
    "/<0;1>/*",
    ")"])

>>> import bip380_checksum
>>> descriptor = bip380_checksum.descsum_create(descriptor)

>>> print(repr(descriptor))
"wpkh([07fd816d/84'/1'/0']tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G/<0;1>/*)#n2zsj5gl"
```

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
["wpkh([07fd816d/84'/1'/0']tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G/0/*)#pq9f8vwq", "wpkh([07fd816d/84'/1'/0']tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G/1/*)#s5qg6e7c"]
```

---

## Watch-Only Wallet

Instead of using `bitcoin-cli` at the command line, we'll be connecting to Bitcoin Core with 
python-bitcoinrpc as is done in this [BlockchainCommons tutorial](https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/master/18_4_Accessing_Bitcoind_with_Python.md).  If you're more familiar with `bitcoin-cli`, then know that all `rpc.method_name()` calls below can be made like `bitcoin-cli command` with similar paramaters.

In the examples below, `rpc.` is an authproxy object that is still connected to `bitcoind` -- thanks to an exaggerated configuration setting `rpcservertimeout=600` so that connections remain open for 10m, instead of closing once idle for 30s.

```python
# create the wallet
>>> rpc.createwallet(
    "single-p2wpkh",  # wallet_name
    True,  # disable_private_keys
    True,  # blank
    "",  # passphrase
    True,  # avoid_reuse
    True,  # descriptors
    False,  # load_on_startup
    False)  # external_signer
{'name': 'single-p2wpkh', 'warnings': ['Empty string given as passphrase, wallet will not be encrypted.']}

# import its descriptors
>>> rpc.importdescriptors([dict(desc=x, timestamp="now") for x in descriptors])
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]

# view the wallet
>>> pprint(rpc.getwalletinfo())
{'avoid_reuse': True,
 'balance': Decimal('0E-8'),
 'birthtime': 1737635294,
 'blank': True,
 'descriptors': True,
 'external_signer': False,
 'format': 'sqlite',
 'immature_balance': Decimal('0E-8'),
 'keypoolsize': 0,
 'keypoolsize_hd_internal': 0,
 'lastprocessedblock': {'hash': '0000006b23cabd40e6bba957bf445004aaf25f29db67699ddd06f965972b2314',
                        'height': 232247},
 'paytxfee': Decimal('0E-8'),
 'private_keys_enabled': False,
 'scanning': False,
 'txcount': 0,
 'unconfirmed_balance': Decimal('0E-8'),
 'walletname': 'single-p2wpkh',
 'walletversion': 169900}

# view first 10 receive addresses
>>> pprint(rpc.deriveaddresses(descriptors[0], 9))
['tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2',
 'tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8',
 'tb1qnd9x7q8ncvm2gyn8kl038qdtywgflus7ukmfeg',
 'tb1q3emex5n7pece35ytek3ec9gen6qz6hg83fy5f5',
 'tb1qz09lne6hmcuw7h5paef6wgs6xpd00rdlg3sy90',
 'tb1qj9739amd4mwpw9e4ehvlm8rzatlhazn0tvg3te',
 'tb1qr7egelt8wzxwz5c9z9n00037sacu4n9m8cg80v',
 'tb1qwds6h4n6tm97gshe2skaqwzdn2e34xcnh5ty8f',
 'tb1q5x54tdjkpq06p5dgnn24r59vy6vv8w7jagt30r',
 'tb1qr0pjqkqu24hgx74jfr5p6zvkfyyh6xqvx8lwgu']

# view first 10 change addresses
>>> pprint(rpc.deriveaddresses(descriptors[1], 9))
['tb1q89rwtw5x0xu65zy9rwvqcqzc36meagw6p86ux5',
 'tb1qpvyq4yp0frytyd0cxhae44g8a22gz9mtulxawe',
 'tb1qeqs6a6q3yscwczm44e5vflmsvkv752pgsz4cys',
 'tb1qz2grf42pztnyxj4l2s0ya0rppkcuqgwv7w7j8n',
 'tb1qjxhqhwhxp7etckl6rj63pt5p7qzu8m3ugsjanu',
 'tb1q7r7kzhhd5g28pvmvngvf9y799ggck5jg748mr6',
 'tb1q2c4ft88rwvsue7yll254uql2kp8w5vfmpj89cj',
 'tb1qajxv4dj8sfh6pf2caa45t4uph86dtrlzf0960p',
 'tb1q924pwhlnlm3pgmnppfsfw67jkj6tjdcl8n274e',
 'tb1q3ysq4kjm6xv9ju0prhcffqxfp3tuytccrpntrx']

#
# funded first receive address from alt.signetfaucet.com
# recycle-to: tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn
#

# view UTXOs
>>> pprint(rpc.listunspent())
[{'address': 'tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2',
  'amount': Decimal('0.00721248'),
  'confirmations': 66,
  'desc': 'wpkh([07fd816d/84h/1h/0h/0/0]03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a)#8wa4ketp',
  'parent_descs': ["wpkh([07fd816d/84'/1'/0']tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G/0/*)#pq9f8vwq"],
  'reused': False,
  'safe': True,
  'scriptPubKey': '00143fea67521d4b891c6231e54be2641de00e8ca5dd',
  'solvable': True,
  'spendable': True,
  'txid': 'b018c2a1a731bd35469cf652dd62ec9a52a3f7f23703d9c681b75376d6b42c5f',
  'vout': 1}]
```

---

## Fun Fact

Bitcoin Core validated the checksum on the descriptors imported above, then changed the key-origin's hardened-derivation path notation from `'` to `h`, and re-computed new checksums.  Still, all of the addresses remain the same.

Sparrow does this too.

[BIP-380](https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki#key-expressions)
initially specifies `h` as the indicator for `hardened` derivations, then later notes that `h`may be replaced by `H` and `'` as alternative hardened indicators.  While this works functionally as the "same" wallet output descriptor, this non-strict tolerance implies that the "same" functional descriptors may have different checksums, thus descriptor checksums would be malleable (not ideal for identifying a wallet) w/o a strict hardened indicator.

```python
# view the imported-validated-and-altered wallet descriptors
>>> pprint(rpc.listdescriptors())
{'descriptors': [{'active': False,
                  'desc': 'wpkh([07fd816d/84h/1h/0h]tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G/0/*)#j36kaker',
                  'next': 1,
                  'next_index': 1,
                  'range': [0, 1000],
                  'timestamp': 1737668554},
                 {'active': False,
                  'desc': 'wpkh([07fd816d/84h/1h/0h]tpubDDTaJCQYtwr9j9oFyNgnvy85MbkqxtdELw5YWLnfzm1EDc5Y1mDKLbrthDJjpdTowjL8K74Kno5sFiyu1nJHgZymBGWrQRXnfaw76vKYt7G/1/*)#r9lhqrfm',
                  'next': 0,
                  'next_index': 0,
                  'range': [0, 999],
                  'timestamp': 1737668554}],
 'wallet_name': 'single-p2wpkh'}

# view the first 10 receive addresses of the altered descriptors
>>> pprint(rpc.deriveaddresses(rpc.listdescriptors()['descriptors'][0]['desc'], 9))
['tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2',
 'tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8',
 'tb1qnd9x7q8ncvm2gyn8kl038qdtywgflus7ukmfeg',
 'tb1q3emex5n7pece35ytek3ec9gen6qz6hg83fy5f5',
 'tb1qz09lne6hmcuw7h5paef6wgs6xpd00rdlg3sy90',
 'tb1qj9739amd4mwpw9e4ehvlm8rzatlhazn0tvg3te',
 'tb1qr7egelt8wzxwz5c9z9n00037sacu4n9m8cg80v',
 'tb1qwds6h4n6tm97gshe2skaqwzdn2e34xcnh5ty8f',
 'tb1q5x54tdjkpq06p5dgnn24r59vy6vv8w7jagt30r',
 'tb1qr0pjqkqu24hgx74jfr5p6zvkfyyh6xqvx8lwgu']

# view the first 10 change addresses of the altered descriptors
>>> pprint(rpc.deriveaddresses(rpc.listdescriptors()['descriptors'][1]['desc'], 9))
['tb1q89rwtw5x0xu65zy9rwvqcqzc36meagw6p86ux5',
 'tb1qpvyq4yp0frytyd0cxhae44g8a22gz9mtulxawe',
 'tb1qeqs6a6q3yscwczm44e5vflmsvkv752pgsz4cys',
 'tb1qz2grf42pztnyxj4l2s0ya0rppkcuqgwv7w7j8n',
 'tb1qjxhqhwhxp7etckl6rj63pt5p7qzu8m3ugsjanu',
 'tb1q7r7kzhhd5g28pvmvngvf9y799ggck5jg748mr6',
 'tb1q2c4ft88rwvsue7yll254uql2kp8w5vfmpj89cj',
 'tb1qajxv4dj8sfh6pf2caa45t4uph86dtrlzf0960p',
 'tb1q924pwhlnlm3pgmnppfsfw67jkj6tjdcl8n274e',
 'tb1q3ysq4kjm6xv9ju0prhcffqxfp3tuytccrpntrx']
```

---

## Spend Funds

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

# we'll also send 650,000 sats to an external wallet
>>> send_addr = "tb1qlvrmer6xydhfkkqtag5mcaxxq6tdh3xm59k3n4"
>>> send_amt = Decimal("0.0065")
>>> txout.append({send_addr: send_amt})
>>> spendable -= send_amt

# we'll send the rest back to the faucet recycle address, assuming no fees yet
>>> faucet_addr = "tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn"
>>> faucet_amt = spendable
>>> txout.append({faucet_addr: spendable})
>>> spendable -= faucet_amt

# we'll pay 10 sats per vbyte
>>> fee_rate = Decimal("0.00000010")
>>> vsize = rpc.decodepsbt(rpc.createpsbt(txin, txout))['tx']['vsize']
>>> fees = vsize * fee_rate
>>> faucet_amt -= fees
>>> spendable += fees
>>> assert faucet_amt > 0 and spendable == fees
>>> txout[-1] = {faucet_addr: faucet_amt}

>>> pprint(txin)
[{'txid': 'b018c2a1a731bd35469cf652dd62ec9a52a3f7f23703d9c681b75376d6b42c5f',
  'vout': 1}]

>>> pprint(txout)
[{'tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8': Decimal('0.0005')},
 {'tb1qlvrmer6xydhfkkqtag5mcaxxq6tdh3xm59k3n4': Decimal('0.0065')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00019808')}]

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAJACAAAAAV8stNZ2U7eBxtkDN/L3o1Ka7GLdUvacRjW9MaehwhiwAQAAAAD9////A1DDAAAAAAAAFgAU8LRBGJw7hwiLgxsRJAx5GCv4jAUQ6wkAAAAAABYAFPsHvI9GI26bWAvqKbx0xgaW28TbYE0AAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yX6LAwAAAQEfYAELAAAAAAAWABQ/6mdSHUuJHGIx5UviZB3gDoyl3SIGA3aPv3CK3r+RnIpmW0JJNCBCbdbrmgQ/qVIirDSHm2tKGAf9gW1UAACAAQAAgAAAAIAAAAAAAAAAAAAiAgLEdiNTcTd8ZySk2S47anW27fR8U0TerVHwY7yJYEOURxgH/YFtVAAAgAEAAIAAAACAAAAAAAEAAAAAAAA='

# result of signing by external signer (in this case, krux)
>>> signed_psbt = "cHNidP8BAJACAAAAAV8stNZ2U7eBxtkDN/L3o1Ka7GLdUvacRjW9MaehwhiwAQAAAAD9////A1DDAAAAAAAAFgAU8LRBGJw7hwiLgxsRJAx5GCv4jAUQ6wkAAAAAABYAFPsHvI9GI26bWAvqKbx0xgaW28TbYE0AAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yX6LAwAAAQEfYAELAAAAAAAWABQ/6mdSHUuJHGIx5UviZB3gDoyl3SICA3aPv3CK3r+RnIpmW0JJNCBCbdbrmgQ/qVIirDSHm2tKRzBEAiBIcuRC6gXSABbcy3ltxVNWX57qGotb9SFtI1bvhM6FYgIgBAzjmc7fn7lbCU4wM8Q3xZnkCleoZmfDK21SuuIouo8BAAAAAA=="

# seedsigner would have added the same exact signature, but also trims inputs:witness_utxo
# >>> signed_psbt = "cHNidP8BAJACAAAAAV8stNZ2U7eBxtkDN/L3o1Ka7GLdUvacRjW9MaehwhiwAQAAAAD9////A1DDAAAAAAAAFgAU8LRBGJw7hwiLgxsRJAx5GCv4jAUQ6wkAAAAAABYAFPsHvI9GI26bWAvqKbx0xgaW28TbYE0AAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yX6LAwAAIgIDdo+/cIrev5GcimZbQkk0IEJt1uuaBD+pUiKsNIeba0pHMEQCIEhy5ELqBdIAFtzLeW3FU1Zfnuoai1v1IW0jVu+EzoViAiAEDOOZzt+fuVsJTjAzxDfFmeQKV6hmZ8MrbVK64ii6jwEAAAAA"

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, signed_psbt])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'acaa76bb5b4316865145fb4b60a411be69a8d1732f4e21ab1cff18ffc132347a'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/acaa76bb5b4316865145fb4b60a411be69a8d1732f4e21ab1cff18ffc132347a)

---

## Evolution of a PSBT

In the above session, the PSBT evolves with each step. Below these changes are explained.

### `psbt = rpc.createpsbt(...`
Bitcoin Core creates the initial psbt template, knowing only the input's `txid` and `vout`, but including no information about what that input is worth or what the spending conditions might be.  Note: because this is segwit, the final pre-segwit `txid` is already known, but not the `hash`.
```json
{
  "tx": {
    "txid": "acaa76bb5b4316865145fb4b60a411be69a8d1732f4e21ab1cff18ffc132347a",
    "hash": "acaa76bb5b4316865145fb4b60a411be69a8d1732f4e21ab1cff18ffc132347a",
    "version": 2,
    "size": 144,
    "vsize": 144,
    "weight": 576,
    "locktime": 232318,
    "vin": [
      {
        "txid": "b018c2a1a731bd35469cf652dd62ec9a52a3f7f23703d9c681b75376d6b42c5f",
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
          "asm": "0 f0b441189c3b87088b831b11240c79182bf88c05",
          "desc": "addr(tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8)#kxe8r56f",
          "hex": "0014f0b441189c3b87088b831b11240c79182bf88c05",
          "address": "tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00650000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 fb07bc8f46236e9b580bea29bc74c60696dbc4db",
          "desc": "addr(tb1qlvrmer6xydhfkkqtag5mcaxxq6tdh3xm59k3n4)#aht47r2m",
          "hex": "0014fb07bc8f46236e9b580bea29bc74c60696dbc4db",
          "address": "tb1qlvrmer6xydhfkkqtag5mcaxxq6tdh3xm59k3n4",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00019808,
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
        "amount": 0.00721248,
        "scriptPubKey": {
          "asm": "0 3fea67521d4b891c6231e54be2641de00e8ca5dd",
          "desc": "addr(tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2)#87efd9q4",
          "hex": "00143fea67521d4b891c6231e54be2641de00e8ca5dd",
          "address": "tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2",
          "type": "witness_v0_keyhash"
        }
      },
      "bip32_derivs": [
        {
          "pubkey": "03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a",
          "master_fingerprint": "07fd816d",
          "path": "m/84h/1h/0h/0/0"
        }
      ]
```
It also adds information pertaining to any of the outputs which are known to belong to the descriptors.  In this case, it fills the first of three empty dictionaries with the following content -- within the `outputs` list.
```json
      "bip32_derivs": [
        {
          "pubkey": "02c476235371377c6724a4d92e3b6a75b6edf47c5344dead51f063bc8960439447",
          "master_fingerprint": "07fd816d",
          "path": "m/84h/1h/0h/0/1"
        }
      ]
```
Last of all, since the value of the inputs is now known, it can finally calculate and append the `fee`.
```json
   ,
  "fee": 0.00001440
```

### `signed_psbt = ...`
It is the job of the Signer to display pertinent information to the user so that they understand what is being signed, and to add `partial_signatures` to the PSBT.  In this case, the Signer adds a key to the existing dictionary in `inputs`
```json
"partial_signatures": {
        "03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a": "304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356ef84ce85620220040ce399cedf9fb95b094e3033c437c599e40a57a86667c32b6d52bae228ba8f01"
      }
```
The Signer's role in the PSBT is simply to "add" signatures, and done.  In fact,
[BIP-174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki#signer)
clearly states: "The Signer must only add data to a PSBT.", but some rules were meant to be broken?!?!

Because an air-gapped Signer which transmits data via QR-Code is constrained by this low-bandwidth medium, some have "optimized" to transfer less -- by stripping unnecessary fields from `inputs` and `outputs`.  Krux strips`bip32_derivs`from both `inputs` and `outputs`, while SeedSigner does the same and also strips `witness_utxo` from `inputs`.

This is a non-standard `feature` of some air-gapped-via-qrcode Signers which is NOT practiced when a signed PSBT can be written efficiently to an sdcard or transfered electronically.  As expected, this non-standard feature must also be tolerated by whichever software is acting in the role of "Combiner", as we'll see below.

### `combined_psbt = rpc.combinepsbt(...`
Had the air-gapped signer not stripped any fields in its quest for an optimized transmit-via-qrocde user-experience, the resulting `signed_psbt` would have been the same as the `combined_psbt`.  Still, because other types of transactions may require multiple signatures from multiple Signers, it is important for the Combiner to consider combining all signed PSBTs with the original PSBT distributed to Signers.  In this case, Bitcoin Core simply replaces the dictionary keys that had been stripped out.

Recall that SeedSigner had stripped `witness_utxo` from `inputs`, therefore Bitcoin Core restores it,
```json
      "witness_utxo": {
        "amount": 0.00721248,
        "scriptPubKey": {
          "asm": "0 3fea67521d4b891c6231e54be2641de00e8ca5dd",
          "desc": "addr(tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2)#87efd9q4",
          "hex": "00143fea67521d4b891c6231e54be2641de00e8ca5dd",
          "address": "tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2",
          "type": "witness_v0_keyhash"
        }
      }
```
...and both Krux and SeedSigner had also stripped `bip32_derivs` from `inputs`, which gets restored.
```json
       ,
      "bip32_derivs": [
        {
          "pubkey": "03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a",
          "master_fingerprint": "07fd816d",
          "path": "m/84h/1h/0h/0/0"
        }
      ]
```
Lastly, Bitcoin Core restores content in the first dictionary that had been stripped from the `outputs` field.
```json
      "bip32_derivs": [
        {
          "pubkey": "02c476235371377c6724a4d92e3b6a75b6edf47c5344dead51f063bc8960439447",
          "master_fingerprint": "07fd816d",
          "path": "m/84h/1h/0h/0/1"
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
As the Finalizer role, Bitcoin Core is tasked to determine whether or not the PSBT is complete and final, by verifying that spending conditions for all inputs have been met.  In this case, finalizing the complete PSBT strips-out previously added `bip32_derivs` and replaces `partial_signatures` with `final_scriptwitness` within the `inputs` field.
```json
      "final_scriptwitness": [
        "304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356ef84ce85620220040ce399cedf9fb95b094e3033c437c599e40a57a86667c32b6d52bae228ba8f01",
        "03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a"
      ]
```

By default, when the PSBT is complete and final, Bitcoin Core's `finalizepsbt` will also extract the final rawtransaction hex so that it's ready to be broadcasted into the mempool.  Once extracted, the transaction's `hash` is finally known, while the pre-segwit `txid` had never changed.

```json
{
  "txid": "acaa76bb5b4316865145fb4b60a411be69a8d1732f4e21ab1cff18ffc132347a",
  "hash": "0dc00c2056e775971240a5c85e3e0578da8053ba98ccd65dea38e603f4ecf973",
  "version": 2,
  "size": 253,
  "vsize": 172,
  "weight": 685,
  "locktime": 232318,
  "vin": [
    {
      "txid": "b018c2a1a731bd35469cf652dd62ec9a52a3f7f23703d9c681b75376d6b42c5f",
      "vout": 1,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356ef84ce85620220040ce399cedf9fb95b094e3033c437c599e40a57a86667c32b6d52bae228ba8f01",
        "03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a"
      ],
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 f0b441189c3b87088b831b11240c79182bf88c05",
        "desc": "addr(tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8)#kxe8r56f",
        "hex": "0014f0b441189c3b87088b831b11240c79182bf88c05",
        "address": "tb1q7z6yzxyu8wrs3zurrvgjgrrerq4l3rq9fu32p8",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 0.00650000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 fb07bc8f46236e9b580bea29bc74c60696dbc4db",
        "desc": "addr(tb1qlvrmer6xydhfkkqtag5mcaxxq6tdh3xm59k3n4)#aht47r2m",
        "hex": "0014fb07bc8f46236e9b580bea29bc74c60696dbc4db",
        "address": "tb1qlvrmer6xydhfkkqtag5mcaxxq6tdh3xm59k3n4",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 0.00019808,
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

In a nutshell, spending programmable money is about the owner providing `bitcoin script` "inputs" that will be completed by the utxo's `bitcoin scriptPubKey`, completing without error and leaving a `True` value on top of the stack.  It's necessary to know about the `scriptPubKey` that is part of the address from the input utxo -- whose source was the input transaction and which is stored within each node's `utxoset`.  Below is our input (the second output and utxo of our alt-signet-faucet funding transaction):
```
    {
      "value": 0.00721248,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 3fea67521d4b891c6231e54be2641de00e8ca5dd",
        "desc": "addr(tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2)#87efd9q4",
        "hex": "00143fea67521d4b891c6231e54be2641de00e8ca5dd",
        "address": "tb1q8l4xw5safwy3cc33u497yeqauq8gefwaza50x2",
        "type": "witness_v0_keyhash"
      }
    }
```

Below, we will start a `btcdeb` session at the bash commandline using the rawtransaction hex of the above transaction, as well as the same for this transaction's only input.
```bash
btcdeb --quiet --tx=020000000001015f2cb4d67653b781c6d90337f2f7a3529aec62dd52f69c4635bd31a7a1c218b00100000000fdff
ffff0350c3000000000000160014f0b441189c3b87088b831b11240c79182bf88c0510eb090000000000160014fb07bc8f46236e9b580bea29bc74
c60696dbc4db604d000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90247304402204872e442ea05d20016dccb796dc55356
5f9eea1a8b5bf5216d2356ef84ce85620220040ce399cedf9fb95b094e3033c437c599e40a57a86667c32b6d52bae228ba8f012103768fbf708ade
bf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a7e8b0300 --txin=02000000000101db196168d74dacd940bedb9f8828261f50c7
ec5e3167b2c8ad9cd73b651bb6a30000000000fdffffff0294ceef4107000000225120aac35fe91f20d48816b3c83011d117efa35acd2414d36c1e
02b0f29fc3106d9060010b00000000001600143fea67521d4b891c6231e54be2641de00e8ca5dd014085140756d4ecb2b3e06c3c24672b9aa0e306
556fd92969af2e658f74e2129532c4262345eb90e576b4f42b57fd8827800ab48a06ef30f753e1e5875fc1a7c0813c8b0300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 1; value = 721248
got witness stack of size 2
22 bytes (P2WPKH)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(b018c2a1a7, 1)
script                                   |                                                             stack 
-----------------------------------------+-------------------------------------------------------------------
OP_DUP                                   | 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
OP_HASH160                               | 304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356e...
3fea67521d4b891c6231e54be2641de00e8ca5dd | 
OP_EQUALVERIFY                           | 
OP_CHECKSIG                              | 
#0000 OP_DUP
```

Even though we used `--quiet`, we have plenty of output.
* we're signing segwit or taproot
* btcdeb will work one input at a time, and it knows the vout and the value in sats
* It knows that signing this transaction initialized the `script` with two values which it pushed onto the stack (last-in-first-out)
* It knows it's dealing with a p2wpkh address and that the script seems valid
* It calcs the input's txid and acknowledges it w/ the vout index
* and then it shows us the script (one step per line) on the left with the initial stack from our txinwitness values.

Let's now walk thru, one step at a time (we'll just type: `step` and hit `<enter>` within the btcdeb session)

OP_DUP will put an extra copy of the top-stack-item on top of the stack
```
		<> PUSH stack 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
script                                   |                                                             stack 
-----------------------------------------+-------------------------------------------------------------------
OP_HASH160                               | 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
3fea67521d4b891c6231e54be2641de00e8ca5dd | 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
OP_EQUALVERIFY                           | 304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356e...
OP_CHECKSIG                              | 
#0001 OP_HASH160
```

OP_HASH160 will pop an item off the stack and push the digest of `ripemd160(sha256(popped-item))` onto the stack
```
		<> POP  stack
		<> PUSH stack 3fea67521d4b891c6231e54be2641de00e8ca5dd
script                                   |                                                             stack 
-----------------------------------------+-------------------------------------------------------------------
3fea67521d4b891c6231e54be2641de00e8ca5dd |                           3fea67521d4b891c6231e54be2641de00e8ca5dd
OP_EQUALVERIFY                           | 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
OP_CHECKSIG                              | 304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356e...
#0002 3fea67521d4b891c6231e54be2641de00e8ca5dd
```

The next step will push a hash that came from the address of our input, onto the stack
```
		<> PUSH stack 3fea67521d4b891c6231e54be2641de00e8ca5dd
script                                   |                                                             stack 
-----------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                           |                           3fea67521d4b891c6231e54be2641de00e8ca5dd
OP_CHECKSIG                              |                           3fea67521d4b891c6231e54be2641de00e8ca5dd
                                         | 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
                                         | 304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356e...
#0003 OP_EQUALVERIFY
```

The next step will compare the first two items on top of the stack, and fail if they are not the same
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                   |                                                             stack 
-----------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                              | 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
                                         | 304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356e...
#0004 OP_CHECKSIG
```

The next step will validate the pubkey+signature in the top two stack items, and push a `1` if valid, or `0` if invalid
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356ef84ce85620220040ce399cedf9fb95b094e3033c437c599e40a57a86667c32b6d52bae228ba8f01
  pub key     = 03768fbf708adebf919c8a665b42493420426dd6eb9a043fa95222ac34879b6b4a
  script code = 76a9143fea67521d4b891c6231e54be2641de00e8ca5dd88ac
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=721248)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = 0f0f650933c365e8d424f0ab37739990e5c8d425a4b29658a0ea7e1e8a84bdea
  pubkey.VerifyECDSASignature(sig=304402204872e442ea05d20016dccb796dc553565f9eea1a8b5bf5216d2356ef84ce85620220040ce399cedf9fb95b094e3033c437c599e40a57a86667c32b6d52bae228ba8f, sighash=0f0f650933c365e8d424f0ab37739990e5c8d425a4b29658a0ea7e1e8a84bdea):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                   |                                                             stack 
-----------------------------------------+-------------------------------------------------------------------
                                         |                                                                 01
```

Since the script completed without failure and left a non-zero value on top of the stack... this transactioin is valid.
