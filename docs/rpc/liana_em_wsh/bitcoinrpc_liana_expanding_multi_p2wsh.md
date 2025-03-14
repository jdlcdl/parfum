# Bitcoin Core Watch-Only Liana Expanding-Multi WSH

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

We'll be using multiple labels for our secrets and keys because we're aware that our Liana Miniscript descriptor will re-use our first two xpubs from the primary path as altered key-expressions in the recovery path. These labels will make more sense once we load the descriptor in Krux, which will label keys A-E in the order they are presented within the descriptor. Our primary path will use pubkeys from A, B while the recovery path will use pubkeys from C, D and E.

### A and C (primary + recovery)

**BIP-39 mnemonic**: `auction crucial trend safe faith barrel orbit roast source stereo discover cart`

**BIP-39 passphrase**: ""

### B and D (primary + recovery)

**BIP-39 mnemonic**: `orange enter age rug chef denial legend topic identify sign always mother`

**BIP-39 passphrase**: ""

### E (recovery only)

**BIP-39 mnemonic**: `news lecture under adapt inspire chunk tongue fun party build defense receive`

**BIP-39 passphrase**: ""

![secrets-exp-multi](https://gist.github.com/user-attachments/assets/7879acda-0c0f-4514-9705-b34fbb359f22)


---

## Extended Public Keys

### A and C (primary + recovery)

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

### B and D (primary + recovery)

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

### E (recovery only)

**BIP-32 master fingerprint**: `cdef7cd9`

**Derivation Path**: `m/48h/1h/0h/2h`

**Key-Origin**: `[cdef7cd9/48h/1h/0h/2h]`

**xpub**:
```
tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2
```

**xpub w/ key-origin**:
```
[cdef7cd9/48h/1h/0h/2h]tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2
```

---

## Output Descriptor:

We will use Liana to create a P2WSH based Expanding Multisig wallet such that the primary spending path will be a 2-of-2 multisig using keys A and B, and after some time it will expand to a 2-of-3 multisig -- reusing same keys relabeled as C and D, and including recovery key E. We'll use "Advanced" to select "P2WSH", then using each of the above XPUBs w/ key-origin, we'll cut-n-paste them into Liana, and we'll edit the recovery expiration so that each UTXO can be recovered after 6h (36 confirmations). Finally, Liana will ask us to backup "The descriptor"

```
wsh(or_d(multi(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),and_v(v:thresh(2,pkh([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*),a:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*),a:pkh([cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*)),older(36))))#wa74c6se
```

Let's take a moment to inspect this Liana descriptor.  We'll add some whitespace and indentation for readability below:
```
wsh(
  or_d(
    multi(
      2,
      [07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,
      [da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*
    ),
    and_v(
      v:thresh(
        2,
        pkh([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*),
        a:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*),
        a:pkh([cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*)
      ),
      older(36)
    )
  )
)#wa74c6se
```
* the Miniscript is wrapped in `wsh()` with a `#` and bech32 checksum appended
* our two paths, the primary 2-of-2 multisig, and time-delayed recovery 2-of-3 multisig are joined by `or()` (either would be "good enough")
* our primary path is a standard 2-of-2 multisig... NOT sorted, so the order of keys matter
* both keys in our primary will derive receive and change pubkeys via their bip32 child nodes `0` and `1` respectively
* our recovery path requires 2 terms, joined by `and()` (both must succeed)
* first recovery term: `thresh()` is used instead of standard multisig, and our recovery key has been added, order of keys matter
* since our xpubs from the primary path are being reused again, as C and D in the recovery path, they'll derive their receive and change pubkeys from bip32 child nodes `2` and `3` respectively
* recovery-only key `E` will derive receive and change pubkeys from its bip32 child nodes `0` and `1` respectively
* second recovery term: `older(36)` is our relatve time-delay so that spending each input requires committing to doing so only after 36 confirmations.


### Split the descriptor into receive and change `descriptors list` for Bitcoin Core

```python
>>> descriptor = "wsh(or_d(multi(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*),and_v(v:thresh(2,pkh([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*),a:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*),a:pkh([cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*)),older(36))))#wa74c6se"

>>> import bip380_checksum

>>> descriptors = [
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/0/*").replace("/<2;3>/*", "/2/*").split("#")[0]
    ),
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/1/*").replace("/<2;3>/*", "/3/*").split("#")[0]
    )]

>>> print(repr(descriptors))
["wsh(or_d(multi(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),and_v(v:thresh(2,pkh([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/2/*),a:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/2/*),a:pkh([cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*)),older(36))))#c3rx3uqz", "wsh(or_d(multi(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*),and_v(v:thresh(2,pkh([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/3/*),a:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/3/*),a:pkh([cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/1/*)),older(36))))#59gwpw8a"]
```

---

## Watch-Only Wallet

Instead of using `bitcoin-cli` at the command line, we'll be connecting to Bitcoin Core with python-bitcoinrpc as is done in this [BlockchainCommons tutorial](https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/master/18_4_Accessing_Bitcoind_with_Python.md).  If you're more familiar with `bitcoin-cli`, then know that all `rpc.method_name()` calls below can be made like `bitcoin-cli command` with similar paramaters.

In the examples below, `rpc.` is an authproxy object that is still connected to `bitcoind` -- thanks to an exaggerated configuration setting `rpcservertimeout=600` so that connections remain open for 10m, instead of closing once idle for 30s.

```python
# create the wallet
>>> rpc.createwallet(
    "Liana-EM-wsh",  # wallet_name
    True,  # disable_private_keys
    True,  # blank
    "",  # passphrase
    True,  # avoid_reuse
    True,  # descriptors
    False,  # load_on_startup
    False)  # external_signer
{'name': 'Liana-EM-wsh', 'warnings': ['Empty string given as passphrase, wallet will not be encrypted.']}

# import its descriptors
>>> rpc.importdescriptors([dict(desc=x, timestamp="now") for x in descriptors])
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]

# view the wallet
>>> pprint(rpc.getwalletinfo())
{'avoid_reuse': True,
 'balance': Decimal('0E-8'),
 'birthtime': 1738847959,
 'blank': True,
 'descriptors': True,
 'external_signer': False,
 'format': 'sqlite',
 'immature_balance': Decimal('0E-8'),
 'keypoolsize': 0,
 'keypoolsize_hd_internal': 0,
 'lastprocessedblock': {'hash': '00000094944c050cc40ade04a9c2ffd952b2dd2af0e933f37f4976bbb83cc835',
                        'height': 234207},
 'paytxfee': Decimal('0E-8'),
 'private_keys_enabled': False,
 'scanning': False,
 'txcount': 0,
 'unconfirmed_balance': Decimal('0E-8'),
 'walletname': 'Liana-EM-wsh',
 'walletversion': 169900}

# view first 10 receive addresses
>>> pprint(rpc.deriveaddresses(descriptors[0], 9))
['tb1q0rjexqk42w07f3cersg26fr0s4hht6q0q73s44qeny6hnspwgntsxkwcq0',
 'tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys',
 'tb1qf4ytttx7l026n3jqkxqhmqlu2qltpus854zvhrg9xqf407r72fgsqhuxx9',
 'tb1q5j45zvn8tljusk59mgtm9kqz6danjfahrwsstu5mmx626f7w3xhsucumdc',
 'tb1qmdnlxe547s365pqyg9lpeqk98lmhdu4szykyyve6zzl4n2cg5fmskglkjy',
 'tb1qa36hd7sefdznpa760k4rcq46hpw6wuxgg9x5k7kzgazyvua7k2usqkhjgn',
 'tb1qhdr0rrvqaryehgkt3j8va208pvscj00ha4s5k8w46cr3mmnsvjqsla33yh',
 'tb1qt6gtghzza56hp98uw3ypuvc89xxzu6v3hct9a7v7f82n0sf32eesscwes5',
 'tb1qelknmpwkervjax09x9ejl439l9v7jdt42zuh9nhhtckwmmpw96tqdp6dvp',
 'tb1qf38saxvpnrdjrdyv0y6xwafrxhpl99kwh4exwxm7wpje0tml4lcsvy8r2n']

# view first 10 change addresses
>>> pprint(rpc.deriveaddresses(descriptors[1], 9))
['tb1qt44rx0h7glkw6nxksqn79aw2y09m7ts9nh7gzutr3shk9833etzq794nyh',
 'tb1qrr827eu8j0nxht64v7yu4c5tntl3chrw4t2k7mteda8y9q3us4sqtq897g',
 'tb1qpayxf3ar9mszn80nmr5lykkc0czc2sln48e9ntsgwehdt7kxca5ssgf9cp',
 'tb1qlhs6qkpcnqukl37ejhydwrglfxcqjvhvymxs20ckrj2vasuw2n9s33vxha',
 'tb1qhnggfqy4tymqytagfteglp88wjcrfh7t6zzmeegutwm00z4jh45qnujqd4',
 'tb1qc5327dtszhcnd7uxu24e485y6zmtjlvh778nallamvrnkzsx0rfq77a73r',
 'tb1q7xaj39x3fcxkq032vcv9jvnx9u54rvpnk833yusf09e6svx69rxqp0ryr7',
 'tb1qqtwctz2xlaj5vun32z758xkqmxccj4kxynpnxgeqtqy6axuujaeq3ft57y',
 'tb1qp6k9esqprpszwh3yg7hda5haq4rvy3s70j7yzctg492yhxslagyq2y7yxj',
 'tb1qf25dk0af2rg9txstfknnwc65s8rh48tza43zplqf6wusl4y5hvssxye7z3']

#
# funded first two receive addresses from alt.signetfaucet.com
# recycle-to: tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn
#

# view UTXOs
>>> pprint(rpc.listunspent())
[{'address': 'tb1q0rjexqk42w07f3cersg26fr0s4hht6q0q73s44qeny6hnspwgntsxkwcq0',
 'amount': Decimal('0.00200000'),
 'confirmations': 2,
 'desc': 'wsh(or_d(multi(2,[07fd816d/48h/1h/0h/2h/0/0]033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b,[da855a1f/48h/1h/0h/2h/0/0]0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6),and_v(v:thresh(2,pkh([07fd816d/48h/1h/0h/2h/2/0]02c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852),a:pkh([da855a1f/48h/1h/0h/2h/2/0]034cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9),a:pkh([cdef7cd9/48h/1h/0h/2h/0/0]029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c)),older(36))))#v8zcz5yk',
 'parent_descs': ["wsh(or_d(multi(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*),and_v(v:thresh(2,pkh([07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/2/*),a:pkh([da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/2/*),a:pkh([cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*)),older(36))))#c3rx3uqz"],
 'reused': False,
 'safe': True,
 'scriptPubKey': '002078e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d7',
 'solvable': True,
 'spendable': True,
 'txid': 'cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619',
 'vout': 1,
 'witnessScript': '5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268'}]
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

# we'll send 70,000 sats to this wallet
>>> my_addr = rpc.deriveaddresses(rcv_desc['desc'], [rcv_desc['next'], rcv_desc['next']])[0]
>>> my_amt = Decimal("0.0007")
>>> txout.append({my_addr: my_amt})
>>> spendable -= my_amt

# we'll also send 75,000 sats to an external wallet
>>> send_addr = "tb1qs4n7wdnw8hdwvku0v90a4p9a00age3a430c5kn"
>>> send_amt = Decimal("0.00075")
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
[{'txid': 'cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619',
  'vout': 1}]

>>> pprint(txout)
[{'tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys': Decimal('0.0007')},
 {'tb1qs4n7wdnw8hdwvku0v90a4p9a00age3a430c5kn': Decimal('0.00075')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00053440')}]

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAJwCAAAAARn2BebNN91deA4Vxx1X1OfCPepWtkuTeLMRrdxUC8LPAQAAAAD9////A3ARAQAAAAAAIgAgVb8VH8FlJkxsbi5Spn7qhqwnUHWI/UDp4f7voZbYEO34JAEAAAAAABYAFIVn5zZuPdrmW49hX9qEvXv6jMe1wNAAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yeKSAwAAAQErQA0DAAAAAAAiACB45ZMC1VOf5McZHBCtJG+Fb3XoDwejCtQZmTV5wC5E1wEFoFIhAzw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobIQNGAjt4cuACdXfYpFE4m8WkMOcp8tOJoj7MoSN9w7f31lKuc2R2qRS1Ld+PYgMQ+w05oTyArXFPN0sCRoisa3apFOW3+jsaTj2O7Qo84ZGJltHt5GAKiKxsk2t2qRRmpZGyV9G+7UQqgaJwSLhY6WGd4IisbJNSiAEksmgiBgKd0EX4aPz4z6DRlPn+yZDX0L3wHSpC5JToduy23UgvXBzN73zZMAAAgAEAAIAAAACAAgAAgAAAAAAAAAAAIgYCx8NKDhs8Rdkj66bm1/qQdrqQDSoppLub4QBbW+Xv+FIcB/2BbTAAAIABAACAAAAAgAIAAIACAAAAAAAAACIGAzw0PgY4G355Jv2+VMY6sAQMXCyEkeLUvXI4y5BmBAobHAf9gW0wAACAAQAAgAAAAIACAACAAAAAAAAAAAAiBgNGAjt4cuACdXfYpFE4m8WkMOcp8tOJoj7MoSN9w7f31hzahVofMAAAgAEAAIAAAACAAgAAgAAAAAAAAAAAIgYDTPYYWbOGwvBDbIWs86yFr9EQPc5EbuH1i2P9dWGjG/kc2oVaHzAAAIABAACAAAAAgAIAAIACAAAAAAAAAAABAaBSIQIkp+SAiZoU6gFAZn7uHDlcXSEFG1R6E42vq1TMN/a00CECytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq1SrnNkdqkUODnhlmSH8T8DsUlgBMreFBoi6ViIrGt2qRTW6CdbyCJSPFxYOGNGAwcnoF67SYisbJNrdqkUQ8/qLu/TcjO9zl2PHX6jzEkm6xWIrGyTUogBJLJoIgICJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNAcB/2BbTAAAIABAACAAAAAgAIAAIAAAAAAAQAAACICAsrWE3zEgozquhSCJ7EtOq6bRAp+Epy3maezJ4jz2S6tHNqFWh8wAACAAQAAgAAAAIACAACAAAAAAAEAAAAiAgLbh16dBUcAywMG+OazeZZdKHngWnnRPiCbOVdxHF94VRwH/YFtMAAAgAEAAIAAAACAAgAAgAIAAAABAAAAIgIDh2IV9FSkPBH0LVP0dGcv+UNkr4u+UBpc0LUTx2p2K/Mc2oVaHzAAAIABAACAAAAAgAIAAIACAAAAAQAAACICA/jJrzFOdG6Zs5e67n/W7j1ZBBM7WgeIO4qfNucl70w8HM3vfNkwAACAAQAAgAAAAIACAACAAAAAAAEAAAAAAAA='

# result of signing by external signer (in this case, krux)
>>> psbtA = "cHNidP8BAJwCAAAAARn2BebNN91deA4Vxx1X1OfCPepWtkuTeLMRrdxUC8LPAQAAAAD9////A3ARAQAAAAAAIgAgVb8VH8FlJkxsbi5Spn7qhqwnUHWI/UDp4f7voZbYEO34JAEAAAAAABYAFIVn5zZuPdrmW49hX9qEvXv6jMe1wNAAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yeKSAwAAAQErQA0DAAAAAAAiACB45ZMC1VOf5McZHBCtJG+Fb3XoDwejCtQZmTV5wC5E1yICAsfDSg4bPEXZI+um5tf6kHa6kA0qKaS7m+EAW1vl7/hSRzBEAiAnotXLISCOWpOece+gIgvSy0PHZdOlUv0FTdPt4kyNkQIgKEUfmPSxUVEYyqZU+OzLEfj0mahgcGZZSE/LF1c6eP4BIgIDPDQ+Bjgbfnkm/b5UxjqwBAxcLISR4tS9cjjLkGYEChtHMEQCIAlpcrZsUWufme07kuHVQFMTdx6L96w9WLwS2KmEs5hiAiAcfeho8+Fd7+u4Uzj9JMcwPBCd8NqT3gDiK77+5O3TqgEBBaBSIQM8ND4GOBt+eSb9vlTGOrAEDFwshJHi1L1yOMuQZgQKGyEDRgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399ZSrnNkdqkUtS3fj2IDEPsNOaE8gK1xTzdLAkaIrGt2qRTlt/o7Gk49ju0KPOGRiZbR7eRgCoisbJNrdqkUZqWRslfRvu1EKoGicEi4WOlhneCIrGyTUogBJLJoAAAAAA=="

>>> psbtB = "cHNidP8BAJwCAAAAARn2BebNN91deA4Vxx1X1OfCPepWtkuTeLMRrdxUC8LPAQAAAAD9////A3ARAQAAAAAAIgAgVb8VH8FlJkxsbi5Spn7qhqwnUHWI/UDp4f7voZbYEO34JAEAAAAAABYAFIVn5zZuPdrmW49hX9qEvXv6jMe1wNAAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yeKSAwAAAQErQA0DAAAAAAAiACB45ZMC1VOf5McZHBCtJG+Fb3XoDwejCtQZmTV5wC5E1yICA0YCO3hy4AJ1d9ikUTibxaQw5yny04miPsyhI33Dt/fWRzBEAiAm+KpMBUQwGZoI5Q6ix8TnlV4t6qpEj2s5gZuBz+elLQIgf7XLali57ScWsHdtwqk44dCZvCvHUJb1Qpu1usTkWXABIgIDTPYYWbOGwvBDbIWs86yFr9EQPc5EbuH1i2P9dWGjG/lHMEQCIH9oWZeXdgL5QikGQcMXUD74OKCFxY9umr81F8d/c/wbAiA2Ap4bweRxmMTJ5eSKB8sRo2czP4/klCwamhaM0+bHhAEBBaBSIQM8ND4GOBt+eSb9vlTGOrAEDFwshJHi1L1yOMuQZgQKGyEDRgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399ZSrnNkdqkUtS3fj2IDEPsNOaE8gK1xTzdLAkaIrGt2qRTlt/o7Gk49ju0KPOGRiZbR7eRgCoisbJNrdqkUZqWRslfRvu1EKoGicEi4WOlhneCIrGyTUogBJLJoAAAAAA=="

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, psbtA, psbtB])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953)

---

## Primary: Evolution of a PSBT

In the above session, the PSBT evolves with each step. Below these changes are explained.

### `psbt = rpc.createpsbt(...`
Bitcoin Core creates the initial psbt template, knowing only the input's `txid` and `vout`, but including no information about what that input is worth or what the spending conditions might be.  Note: because this is segwit, the final pre-segwit `txid` is already known, but not the `hash`.
```json
{
  "tx": {
    "txid": "3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953",
    "hash": "3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953",
    "version": 2,
    "size": 156,
    "vsize": 156,
    "weight": 624,
    "locktime": 234210,
    "vin": [
      {
        "txid": "cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619",
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
        "value": 0.00070000,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 55bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
          "desc": "addr(tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys)#rhw06cd4",
          "hex": "002055bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
          "address": "tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys",
          "type": "witness_v0_scripthash"
        }
      },
      {
        "value": 0.00075000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 8567e7366e3ddae65b8f615fda84bd7bfa8cc7b5",
          "desc": "addr(tb1qs4n7wdnw8hdwvku0v90a4p9a00age3a430c5kn)#zz4l68lv",
          "hex": "00148567e7366e3ddae65b8f615fda84bd7bfa8cc7b5",
          "address": "tb1qs4n7wdnw8hdwvku0v90a4p9a00age3a430c5kn",
          "type": "witness_v0_keyhash"
        }
      },
      {
        "value": 0.00053440,
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
          "asm": "0 78e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d7",
          "desc": "addr(tb1q0rjexqk42w07f3cersg26fr0s4hht6q0q73s44qeny6hnspwgntsxkwcq0)#9fqhs7ek",
          "hex": "002078e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d7",
          "address": "tb1q0rjexqk42w07f3cersg26fr0s4hht6q0q73s44qeny6hnspwgntsxkwcq0",
          "type": "witness_v0_scripthash"
        }
      },
      "witness_script": {
        "asm": "2 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 2 OP_CHECKMULTISIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 b52ddf8f620310fb0d39a13c80ad714f374b0246 OP_EQUALVERIFY OP_CHECKSIG OP_TOALTSTACK OP_DUP OP_HASH160 e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD OP_TOALTSTACK OP_DUP OP_HASH160 66a591b257d1beed442a81a27048b858e9619de0 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD 2 OP_EQUALVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0"
        },
        {
          "pubkey": "02c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/0"
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
        },
        {
          "pubkey": "034cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/0"
        }
      ]
```
It also adds to `outputs` for any that it knows about
```json
      "witness_script": {
        "asm": "2 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead 2 OP_CHECKMULTISIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 3839e1966487f13f03b1496004cade141a22e958 OP_EQUALVERIFY OP_CHECKSIG OP_TOALTSTACK OP_DUP OP_HASH160 d6e8275bc822523c5c58386346030727a05ebb49 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD OP_TOALTSTACK OP_DUP OP_HASH160 43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD 2 OP_EQUALVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268",
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
        },
        {
          "pubkey": "02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```
Lastly, since it knows about the input value, it can calculate and add the `fee`
```json
   ,
  "fee": 0.00001560
```

### `signed_psbt = ...`
It is the job of the Signer to display pertinent information to the user so that they understand what is being signed, and to add signatures to the PSBT.  Signing with secret A, krux adds a key with two signatures (for each path, because it doesn't know which will be finalizable) to existing dictionary in `inputs`
```json
      "partial_signatures": {
        "02c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852": "3044022027a2d5cb21208e5a939e71efa0220bd2cb43c765d3a552fd054dd3ede24c8d91022028451f98f4b1515118caa654f8eccb11f8f499a860706659484fcb17573a78fe01",
        "033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b": "30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4edd3aa01"
      },
```
and when signing with secret B, it similary adds a signature for both paths
```json
      "partial_signatures": {
        "0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6": "3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001",
        "034cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9": "304402207f685997977602f942290641c317503ef838a085c58f6e9abf3517c77f73fc1b022036029e1bc1e47198c4c9e5e48a07cb11a367333f8fe4942c1a9a168cd3e6c78401"
      },
```
The Signer's role in the PSBT is simply to "add" signatures, and done.  In fact,
[BIP-174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki#signer)
clearly states: "The Signer must only add data to a PSBT.", but some rules were meant to be broken?!?!

Because an air-gapped Signer which transmits data via QR-Code is constrained by this low-bandwidth medium, some have "optimized" to transfer less -- by stripping unnecessary fields from `inputs` and `outputs`.  Krux strips`bip32_derivs`from `inputs` and also strips `bip32_derivs` and `witness_script` from `outputs`.

This is a non-standard `feature` of some air-gapped-via-qrcode Signers which is NOT practiced when a signed PSBT can be written efficiently to an sdcard or transfered electronically.  As expected, this non-standard feature must also be tolerated by whichever software is acting in the role of "Combiner", as we'll see below.

### `combined_psbt = rpc.combinepsbt(...`
Had the air-gapped signer not stripped any fields in its quest for an optimized transmit-via-qrocde user-experience, the resulting `signed_psbt` would have been the same as the `combined_psbt`.  Still, because other types of transactions may require multiple signatures from multiple Signers, it is important for the Combiner to consider combining all signed PSBTs with the original PSBT distributed to Signers.  In this case, Bitcoin Core combines the signatures then simply replaces the dictionary keys that had been stripped out from `inputs`

```json
      "bip32_derivs": [
        {
          "pubkey": "029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0"
        },
        {
          "pubkey": "02c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/0"
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
        },
        {
          "pubkey": "034cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/0"
        }
      ]
```
and also restores what was stripped from `outputs`
```json
      "witness_script": {
        "asm": "2 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead 2 OP_CHECKMULTISIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 3839e1966487f13f03b1496004cade141a22e958 OP_EQUALVERIFY OP_CHECKSIG OP_TOALTSTACK OP_DUP OP_HASH160 d6e8275bc822523c5c58386346030727a05ebb49 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD OP_TOALTSTACK OP_DUP OP_HASH160 43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD 2 OP_EQUALVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268",
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
        },
        {
          "pubkey": "02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1"
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
        "30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4edd3aa01",
        "3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001",
        "5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268"
```

By default, when the PSBT is complete and final, Bitcoin Core's `finalizepsbt` will also extract the final rawtransaction hex so that it's ready to be broadcasted into the mempool.  Once extracted, the transaction's `hash` is finally known, while the pre-segwit `txid` had never changed.

```json
{
  "txid": "3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953",
  "hash": "526ab777508711783151a605a9ef538c05349e7cd1c95bb52aca5ec422134813",
  "version": 2,
  "size": 465,
  "vsize": 234,
  "weight": 933,
  "locktime": 234210,
  "vin": [
    {
      "txid": "cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619",
      "vout": 1,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "",
        "30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4edd3aa01",
        "3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001",
        "5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268"
      ],
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00070000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 55bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
        "desc": "addr(tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys)#rhw06cd4",
        "hex": "002055bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
        "address": "tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys",
        "type": "witness_v0_scripthash"
      }
    },
    {
      "value": 0.00075000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 8567e7366e3ddae65b8f615fda84bd7bfa8cc7b5",
        "desc": "addr(tb1qs4n7wdnw8hdwvku0v90a4p9a00age3a430c5kn)#zz4l68lv",
        "hex": "00148567e7366e3ddae65b8f615fda84bd7bfa8cc7b5",
        "address": "tb1qs4n7wdnw8hdwvku0v90a4p9a00age3a430c5kn",
        "type": "witness_v0_keyhash"
      }
    },
    {
      "value": 0.00053440,
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

In a nutshell, spending programmable money is about the owner providing `bitcoin script` "inputs" that will be completed by the utxo's `bitcoin scriptPubKey`, completing without error and leaving a `True` value on top of the stack.  It's necessary to know about the `scriptPubKey` that is part of the address from the input utxo -- whose source was the input transaction and which is stored within each node's `utxoset`.  Below is our input (the output and utxo of our funding transaction):
```json
    {
      "value": 0.00200000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 78e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d7",
        "desc": "addr(tb1q0rjexqk42w07f3cersg26fr0s4hht6q0q73s44qeny6hnspwgntsxkwcq0)#9fqhs7ek",
        "hex": "002078e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d7",
        "address": "tb1q0rjexqk42w07f3cersg26fr0s4hht6q0q73s44qeny6hnspwgntsxkwcq0",
        "type": "witness_v0_scripthash"
      }
    },
```

Below, we will start a `btcdeb` session at the bash commandline using the rawtransaction hex of the above transaction, as well as the same for this transaction's only input.
```
btcdeb --quiet --tx=0200000000010119f605e6cd37dd5d780e15c71d57d4e7c23dea56b64b9378b311addc540bc2cf0100
000000fdffffff03701101000000000022002055bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810edf8240100000000
001600148567e7366e3ddae65b8f615fda84bd7bfa8cc7b5c0d0000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac9040047
30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0
da93de00e22bbefee4edd3aa01473044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9
ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001a05221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb
9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c
80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858
e9619de088ac6c9352880124b268e2920300 --txin=0200000000010103dc016b3823bb1d01244ffce1a1117704183ced100b51601b70d69421
3c88c90000000000fdffffff03400d03000000000022512035d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37400d
03000000000022002078e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d773424a0000000000160014b7e7f48e5c3b
d445be101eb5835021219ced250c0247304402207768183ffd882c935d403f822b88c625d36687005d36707acd9a6e157f5a0b1c022056ab7497
2e275df0ddcc6c1bec45eda7a3ad8a73db7a92835a5cf493b31e91730121028ef9b939e3eba3b7ae30c01e1e5b3633195448b264510cccac9510
55d6910ccbe0920300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 1; value = 200000
got witness stack of size 4
34 bytes (v0=P2WSH, v1=taproot/tapscript)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(cfc20b54dc, 1)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  | 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b8...
033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b | 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a...
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 |                                                                 0x
2                                                                  | 
OP_CHECKMULTISIG                                                   | 
OP_IFDUP                                                           | 
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0000 2
```

Even though we used `--quiet`, we have plenty of output.
* we're signing segwit or taproot
* btcdeb will work one input at a time, and it knows the vout and the value in sats
* It knows that signing this transaction initialized the `script` with two values which it pushed onto the stack (last-in-first-out)
* It knows it's dealing with a wsh or taproot address and that the script seems valid
* It calcs the input's txid and acknowledges it w/ the vout index
* and then it shows us the script (one step per line) on the left with the initial stack from our two signatures.

Let's now walk thru, one step at a time (we'll just type: `step` and hit `<enter>` within the btcdeb session)

Stack was initilalized with:
* an unused but to-be-consumed null byte
* signature for Key A in primary 2-of-2 path
* signature for Key B in primary 2-of-2 path

Next, `2` will be pushed (as in 2-of-n signatures)
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b |                                                                 02
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 | 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b8...
2                                                                  | 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a...
OP_CHECKMULTISIG                                                   |                                                                 0x
OP_IFDUP                                                           | 
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0001 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
```

The pubkey for Key A will be pushed
```
		<> PUSH stack 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6 | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
2                                                                  |                                                                 02
OP_CHECKMULTISIG                                                   | 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b8...
OP_IFDUP                                                           | 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a...
OP_NOTIF                                                           |                                                                 0x
OP_DUP                                                             | 
OP_HASH160                                                         | 
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0002 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
```

The pubkey for Key B will be pushed
```
		<> PUSH stack 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  | 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
OP_CHECKMULTISIG                                                   | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
OP_IFDUP                                                           |                                                                 02
OP_NOTIF                                                           | 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b8...
OP_DUP                                                             | 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a...
OP_HASH160                                                         |                                                                 0x
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0003 2
```

The number of pubkeys, `2` will be pushed (as in m-of-2 pubkeys)
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKMULTISIG                                                   |                                                                 02
OP_IFDUP                                                           | 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
OP_NOTIF                                                           | 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
OP_DUP                                                             |                                                                 02
OP_HASH160                                                         | 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b8...
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a...
OP_EQUALVERIFY                                                     |                                                                 0x
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0004 OP_CHECKMULTISIG
```

The multisig is checked for validity (7 items consumed, `1` pushed since valid)
```
stack has 7 entries [require 1]
stack has 7 entries [require 4]
stack has 7 entries [require 7]
scriptCode = 5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268
looping for multisig
loop: sigs = 2, keys = 2
- got sig 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001
- got key 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001
  pub key     = 0346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
  script code = 5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=200000)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = c1caed578ed9e3f49f5d99e5283024ca17b199f04ee9be21e090976d48279c32
  pubkey.VerifyECDSASignature(sig=3044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e45970, sighash=c1caed578ed9e3f49f5d99e5283024ca17b199f04ee9be21e090976d48279c32):
  result: success
- sig check succeeded
loop: sigs = 1, keys = 1
- got sig 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4edd3aa01
- got key 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4edd3aa01
  pub key     = 033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
  script code = 5221033c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=200000)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = c1caed578ed9e3f49f5d99e5283024ca17b199f04ee9be21e090976d48279c32
  pubkey.VerifyECDSASignature(sig=30440220096972b66c516b9f99ed3b92e1d5405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4edd3aa, sighash=c1caed578ed9e3f49f5d99e5283024ca17b199f04ee9be21e090976d48279c32):
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
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_IFDUP                                                           |                                                                 01
OP_NOTIF                                                           | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0005 OP_IFDUP
```

The top item, since true, will be duplicated
```
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NOTIF                                                           |                                                                 01
OP_DUP                                                             |                                                                 01
OP_HASH160                                                         | 
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0006 OP_NOTIF
```

Since the top of stack is true, OP_NOTIF will fail, skipping all the way to OP_ENDIF
```
 stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NOTIF                                                           |                                                                 01
OP_DUP                                                             |                                                                 01
OP_HASH160                                                         | 
b52ddf8f620310fb0d39a13c80ad714f374b0246                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
e5b7fa3b1a4e3d8eed0a3ce1918996d1ede4600a                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
66a591b257d1beed442a81a27048b858e9619de0                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0006 OP_NOTIF
```
...
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_ENDIF                                                           |                                                                 01
#0032 OP_ENDIF

script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 01
```

Since the script completed without failure and left a non-zero value on top of the stack... this transactioin is valid.

---

## Recovery: Spend Funds

This time, we'll spend our only utxo to 2 outputs and we'll sign with the Recovery Keys D and E. Because this path has an OP_CSV, (See [BIP-68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki) and [BIP-112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) for how this works), we must be careful setting the input's nSequence value to be greater-than-or-equal-to 36 (recall older(36) in our Miniscript descriptor). When it's time to spend -- at least 36 blocks after our utxo's first confirmation, we'll also set our nLockTime to the current blockheight (or if pre-signing in advance, we'll extend the nLockTime appropriately).

```python
# assemble the inputs from our only utxo, note how much is spendable
# explicitely set nSequence
>>> unspent = rpc.listunspent()[0]
>>> spendable = unspent['amount']
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
  'txid': '3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953',
  'vout': 0}]

>>> pprint(txout)
[{'tb1qf4ytttx7l026n3jqkxqhmqlu2qltpus854zvhrg9xqf407r72fgsqhuxx9': Decimal('0.0005')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00018750')}]

# our utxo is already expired according to "older(36)"
>>> unspent['confirmations']
62

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'cHNidP8BAH0CAAAAAVNpEpN5jnMD1cOLT6kUy1mJCWus5Uq1HithI/e/7IU4AAAAAAAkAAAAAlDDAAAAAAAAIgAgTUi1rN771anGQLGBfYP8UD6w8gelRMuNBTATV/h+UlE+SQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJJJMDAAABAStwEQEAAAAAACIAIFW/FR/BZSZMbG4uUqZ+6oasJ1B1iP1A6eH+76GW2BDtAQWgUiECJKfkgImaFOoBQGZ+7hw5XF0hBRtUehONr6tUzDf2tNAhAsrWE3zEgozquhSCJ7EtOq6bRAp+Epy3maezJ4jz2S6tUq5zZHapFDg54ZZkh/E/A7FJYATK3hQaIulYiKxrdqkU1ugnW8giUjxcWDhjRgMHJ6Beu0mIrGyTa3apFEPP6i7v03Izvc5djx1+o8xJJusViKxsk1KIASSyaCIGAiSn5ICJmhTqAUBmfu4cOVxdIQUbVHoTja+rVMw39rTQHAf9gW0wAACAAQAAgAAAAIACAACAAAAAAAEAAAAiBgLK1hN8xIKM6roUgiexLTqum0QKfhKct5mnsyeI89kurRzahVofMAAAgAEAAIAAAACAAgAAgAAAAAABAAAAIgYC24denQVHAMsDBvjms3mWXSh54Fp50T4gmzlXcRxfeFUcB/2BbTAAAIABAACAAAAAgAIAAIACAAAAAQAAACIGA4diFfRUpDwR9C1T9HRnL/lDZK+LvlAaXNC1E8dqdivzHNqFWh8wAACAAQAAgAAAAIACAACAAgAAAAEAAAAiBgP4ya8xTnRumbOXuu5/1u49WQQTO1oHiDuKnzbnJe9MPBzN73zZMAAAgAEAAIAAAACAAgAAgAAAAAABAAAAAAEBoFIhA7I42JkNxOtzKV1/B5AJGtNxWz6aDm7fDaGYCgIiNDu4IQLMd6JmM12Yaox4Je1kbx8mrhtmq0mtFeont2scmLYoMlKuc2R2qRROv/bPjReGuXeGZyiPrqsdqT3N5Iisa3apFBhqfDfL6iBCZkXhlc8TVQX2Nbz0iKxsk2t2qRTIHp/Cp4498u0A9cIlY+Y9DdXT0YisbJNSiAEksmgiAgLKCKgsVsc5kEntj1eV+TmuqZ4HgX5MFD2oK/RwGpcrGhzahVofMAAAgAEAAIAAAACAAgAAgAIAAAACAAAAIgICzHeiZjNdmGqMeCXtZG8fJq4bZqtJrRXqJ7drHJi2KDIc2oVaHzAAAIABAACAAAAAgAIAAIAAAAAAAgAAACICAzBwapZhp0NcsgzdcqtlVodxm4IuGJRhqHlHZ1zgBRSVHM3vfNkwAACAAQAAgAAAAIACAACAAAAAAAIAAAAiAgOiG9L2lCrAQ0ixOPuI5AmvPlNdqjwlPxU+CMkMeTHN5hwH/YFtMAAAgAEAAIAAAACAAgAAgAIAAAACAAAAIgIDsjjYmQ3E63MpXX8HkAka03FbPpoObt8NoZgKAiI0O7gcB/2BbTAAAIABAACAAAAAgAIAAIAAAAAAAgAAAAAA'

# result of signing by external signer (in this case, krux)
>>> psbtD = "cHNidP8BAH0CAAAAAVNpEpN5jnMD1cOLT6kUy1mJCWus5Uq1HithI/e/7IU4AAAAAAAkAAAAAlDDAAAAAAAAIgAgTUi1rN771anGQLGBfYP8UD6w8gelRMuNBTATV/h+UlE+SQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJJJMDAAABAStwEQEAAAAAACIAIFW/FR/BZSZMbG4uUqZ+6oasJ1B1iP1A6eH+76GW2BDtIgICytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq1HMEQCIHu+KQEWZNUjksiMh09CqzU5q6TJKC1PAUzA/ZK+vpJ+AiA9tEOKlDwcncxKOflWFU56V5SMZCq+5N8Cv2eq5mDJUQEiAgOHYhX0VKQ8EfQtU/R0Zy/5Q2Svi75QGlzQtRPHanYr80cwRAIgM3NVz50A7p20203L5SX8OPK3hSq5k0auUAX37WNG9Z0CIG2dAn4ubIZev/LwefUKrlRz2Oekn1/p8M66RlQXIHRBAQEFoFIhAiSn5ICJmhTqAUBmfu4cOVxdIQUbVHoTja+rVMw39rTQIQLK1hN8xIKM6roUgiexLTqum0QKfhKct5mnsyeI89kurVKuc2R2qRQ4OeGWZIfxPwOxSWAEyt4UGiLpWIisa3apFNboJ1vIIlI8XFg4Y0YDByegXrtJiKxsk2t2qRRDz+ou79NyM73OXY8dfqPMSSbrFYisbJNSiAEksmgAAAA="

>>> psbtE = "cHNidP8BAH0CAAAAAVNpEpN5jnMD1cOLT6kUy1mJCWus5Uq1HithI/e/7IU4AAAAAAAkAAAAAlDDAAAAAAAAIgAgTUi1rN771anGQLGBfYP8UD6w8gelRMuNBTATV/h+UlE+SQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJJJMDAAABAStwEQEAAAAAACIAIFW/FR/BZSZMbG4uUqZ+6oasJ1B1iP1A6eH+76GW2BDtIgID+MmvMU50bpmzl7ruf9buPVkEEztaB4g7ip825yXvTDxHMEQCIAk75LNQafl6pwN5WGRP9MYcF+84AjPRXEnc0Ggb4HsxAiB8WAOQkHhVxTQ+v/OOY3S86Lu8PVAZT5V/DtIkVyRAFwEBBaBSIQIkp+SAiZoU6gFAZn7uHDlcXSEFG1R6E42vq1TMN/a00CECytYTfMSCjOq6FIInsS06rptECn4SnLeZp7MniPPZLq1SrnNkdqkUODnhlmSH8T8DsUlgBMreFBoi6ViIrGt2qRTW6CdbyCJSPFxYOGNGAwcnoF67SYisbJNrdqkUQ8/qLu/TcjO9zl2PHX6jzEkm6xWIrGyTUogBJLJoAAAA"

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, psbtD, psbtE])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'd47addc57753961c5fb0f174bccdf8af0f0fb3d5ab89ca7650a81dca62f6c3e1'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/d47addc57753961c5fb0f174bccdf8af0f0fb3d5ab89ca7650a81dca62f6c3e1)

---

## Recovery: Evolution of a PSBT

### psbt = rpc.createpsbt(...
```json
{
  "tx": {
    "txid": "d47addc57753961c5fb0f174bccdf8af0f0fb3d5ab89ca7650a81dca62f6c3e1",
    "hash": "d47addc57753961c5fb0f174bccdf8af0f0fb3d5ab89ca7650a81dca62f6c3e1",
    "version": 2,
    "size": 125,
    "vsize": 125,
    "weight": 500,
    "locktime": 234276,
    "vin": [
      {
        "txid": "3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953",
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
          "asm": "0 4d48b5acdefbd5a9c640b1817d83fc503eb0f207a544cb8d05301357f87e5251",
          "desc": "addr(tb1qf4ytttx7l026n3jqkxqhmqlu2qltpus854zvhrg9xqf407r72fgsqhuxx9)#n58swvaj",
          "hex": "00204d48b5acdefbd5a9c640b1817d83fc503eb0f207a544cb8d05301357f87e5251",
          "address": "tb1qf4ytttx7l026n3jqkxqhmqlu2qltpus854zvhrg9xqf407r72fgsqhuxx9",
          "type": "witness_v0_scripthash"
        }
      },
      {
        "value": 0.00018750,
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

### unsigned_psbt = descriptorprocesspsbt(...

Fills `inputs`
```json
      "witness_utxo": {
        "amount": 0.00070000,
        "scriptPubKey": {
          "asm": "0 55bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
          "desc": "addr(tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys)#rhw06cd4",
          "hex": "002055bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
          "address": "tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys",
          "type": "witness_v0_scripthash"
        }
      },
      "witness_script": {
        "asm": "2 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead 2 OP_CHECKMULTISIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 3839e1966487f13f03b1496004cade141a22e958 OP_EQUALVERIFY OP_CHECKSIG OP_TOALTSTACK OP_DUP OP_HASH160 d6e8275bc822523c5c58386346030727a05ebb49 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD OP_TOALTSTACK OP_DUP OP_HASH160 43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD 2 OP_EQUALVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268",
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
        },
        {
          "pubkey": "02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```

and `outputs`
```json
      "witness_script": {
        "asm": "2 03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8 02cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832 2 OP_CHECKMULTISIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 4ebff6cf8d1786b9778667288faeab1da93dcde4 OP_EQUALVERIFY OP_CHECKSIG OP_TOALTSTACK OP_DUP OP_HASH160 186a7c37cbea20426645e195cf135505f635bcf4 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD OP_TOALTSTACK OP_DUP OP_HASH160 c81e9fc2a78e3df2ed00f5c22563e63d0dd5d3d1 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD 2 OP_EQUALVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "522103b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb82102cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b6283252ae736476a9144ebff6cf8d1786b9778667288faeab1da93dcde488ac6b76a914186a7c37cbea20426645e195cf135505f635bcf488ac6c936b76a914c81e9fc2a78e3df2ed00f5c22563e63d0dd5d3d188ac6c9352880124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "02ca08a82c56c7399049ed8f5795f939aea99e07817e4c143da82bf4701a972b1a",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/2"
        },
        {
          "pubkey": "02cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2"
        },
        {
          "pubkey": "0330706a9661a7435cb20cdd72ab655687719b822e189461a87947675ce0051495",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/2"
        },
        {
          "pubkey": "03a21bd2f6942ac04348b138fb88e409af3e535daa3c253f153e08c90c7931cde6",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/2"
        },
        {
          "pubkey": "03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2"
        }
      ]
```

and `fee`
```json
   ,
  "fee": 0.00001250
```

### signed_psbt = ...

strips from `inputs`: `bip32_derivs`

adds to `inputs`
```json
      "partial_signatures": {
        "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c": "30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd0681be07b3102207c580390907855c5343ebff38e6374bce8bbbc3d50194f957f0ed2245724401701",
        "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3": "30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7ed6346f59d02206d9d027e2e6c865ebff2f079f50aae5473d8e7a49f5fe9f0ceba46541720744101",
        "02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead": "304402207bbe29011664d52392c88c874f42ab3539aba4c9282d4f014cc0fd92bebe927e02203db4438a943c1c9dcc4a39f956154e7a57948c642abee4df02bf67aae660c95101"
      },

```

strips from `outputs`: `witness_script` and `bip32_derivs`

### combined_psbt = rpc.combinepsbt(...

replaces stripped fields from `inputs`
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
        },
        {
          "pubkey": "02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1"
        },
        {
          "pubkey": "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1"
        }
      ]
```

...and `outputs`
```json
      "witness_script": {
        "asm": "2 03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8 02cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832 2 OP_CHECKMULTISIG OP_IFDUP OP_NOTIF OP_DUP OP_HASH160 4ebff6cf8d1786b9778667288faeab1da93dcde4 OP_EQUALVERIFY OP_CHECKSIG OP_TOALTSTACK OP_DUP OP_HASH160 186a7c37cbea20426645e195cf135505f635bcf4 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD OP_TOALTSTACK OP_DUP OP_HASH160 c81e9fc2a78e3df2ed00f5c22563e63d0dd5d3d1 OP_EQUALVERIFY OP_CHECKSIG OP_FROMALTSTACK OP_ADD 2 OP_EQUALVERIFY 36 OP_CHECKSEQUENCEVERIFY OP_ENDIF",
        "hex": "522103b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb82102cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b6283252ae736476a9144ebff6cf8d1786b9778667288faeab1da93dcde488ac6b76a914186a7c37cbea20426645e195cf135505f635bcf488ac6c936b76a914c81e9fc2a78e3df2ed00f5c22563e63d0dd5d3d188ac6c9352880124b268",
        "type": "nonstandard"
      },
      "bip32_derivs": [
        {
          "pubkey": "02ca08a82c56c7399049ed8f5795f939aea99e07817e4c143da82bf4701a972b1a",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/2"
        },
        {
          "pubkey": "02cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2"
        },
        {
          "pubkey": "0330706a9661a7435cb20cdd72ab655687719b822e189461a87947675ce0051495",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/2"
        },
        {
          "pubkey": "03a21bd2f6942ac04348b138fb88e409af3e535daa3c253f153e08c90c7931cde6",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/2"
        },
        {
          "pubkey": "03b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2"
        }
      ]
```

### finalized_psbt = rpc.finalizepsbt(...

strips from `inputs`: `partial_signatures` and `bip32_derivs`

adds to `inputs`
```json
      "final_scriptwitness": [
        "30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd0681be07b3102207c580390907855c5343ebff38e6374bce8bbbc3d50194f957f0ed2245724401701",
        "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
        "30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7ed6346f59d02206d9d027e2e6c865ebff2f079f50aae5473d8e7a49f5fe9f0ceba46541720744101",
        "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
        "",
        "02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
        "",
        "",
        "",
        "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268"
```

Final transaction
```json
{
  "txid": "d47addc57753961c5fb0f174bccdf8af0f0fb3d5ab89ca7650a81dca62f6c3e1",
  "hash": "7afbe6b9a43c1cb9f99c8c4d71625119a4c6338fd4e35946c07bd2b39c08caff",
  "version": 2,
  "size": 539,
  "vsize": 229,
  "weight": 914,
  "locktime": 234276,
  "vin": [
    {
      "txid": "3885ecbff723612b1eb54ae5ac6b098959cb14a94f8bc3d503738e7993126953",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd0681be07b3102207c580390907855c5343ebff38e6374bce8bbbc3d50194f957f0ed2245724401701",
        "03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
        "30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7ed6346f59d02206d9d027e2e6c865ebff2f079f50aae5473d8e7a49f5fe9f0ceba46541720744101",
        "03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
        "",
        "02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
        "",
        "",
        "",
        "52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268"
      ],
      "sequence": 36
    }
  ],
  "vout": [
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 4d48b5acdefbd5a9c640b1817d83fc503eb0f207a544cb8d05301357f87e5251",
        "desc": "addr(tb1qf4ytttx7l026n3jqkxqhmqlu2qltpus854zvhrg9xqf407r72fgsqhuxx9)#n58swvaj",
        "hex": "00204d48b5acdefbd5a9c640b1817d83fc503eb0f207a544cb8d05301357f87e5251",
        "address": "tb1qf4ytttx7l026n3jqkxqhmqlu2qltpus854zvhrg9xqf407r72fgsqhuxx9",
        "type": "witness_v0_scripthash"
      }
    },
    {
      "value": 0.00018750,
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
      "value": 0.00070000,
      "n": 0,
      "scriptPubKey": {
        "asm": "0 55bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
        "desc": "addr(tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys)#rhw06cd4",
        "hex": "002055bf151fc165264c6c6e2e52a67eea86ac27507588fd40e9e1feefa196d810ed",
        "address": "tb1q2kl3287pv5nycmrw9ef2vlh2s6kzw5r43r75p60plmh6r9kczrksvv2yys",
        "type": "witness_v0_scripthash"
      }
    },
```

Starting btcdeb

```
/tmp/btcdeb # btcdeb --quiet --tx=0200000000010153691293798e7303d5c38b4fa914cb5989096bace54ab51e2b
6123f7bfec85380000000000240000000250c30000000000002200204d48b5acdefbd5a9c640b1817d83fc503eb0f207a5
44cb8d05301357f87e52513e49000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90a473044022009
3be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd0681be07b3102207c580390907855c5343ebff38e6374bc
e8bbbc3d50194f957f0ed22457244017012103f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef
4c3c4730440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7ed6346f59d02206d9d027e2e6c865e
bff2f079f50aae5473d8e7a49f5fe9f0ceba465417207441012103876215f454a43c11f42d53f474672ff94364af8bbe50
1a5cd0b513c76a762bf3002102db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855000000a0
52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227
b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac
6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc49
26eb1588ac6c9352880124b26824930300 --txin=0200000000010119f605e6cd37dd5d780e15c71d57d4e7c23dea56b6
4b9378b311addc540bc2cf0100000000fdffffff03701101000000000022002055bf151fc165264c6c6e2e52a67eea86ac
27507588fd40e9e1feefa196d810edf8240100000000001600148567e7366e3ddae65b8f615fda84bd7bfa8cc7b5c0d000
0000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac904004730440220096972b66c516b9f99ed3b92e1d5
405313771e8bf7ac3d58bc12d8a984b3986202201c7de868f3e15defebb85338fd24c7303c109df0da93de00e22bbefee4
edd3aa01473044022026f8aa4c054430199a08e50ea2c7c4e7955e2deaaa448f6b39819b81cfe7a52d02207fb5cb6a58b9
ed2716b0776dc2a938e1d099bc2bc75096f5429bb5bac4e4597001a05221033c343e06381b7e7926fdbe54c63ab0040c5c
2c8491e2d4bd7238cb9066040a1b210346023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d652
ae736476a914b52ddf8f620310fb0d39a13c80ad714f374b024688ac6b76a914e5b7fa3b1a4e3d8eed0a3ce1918996d1ed
e4600a88ac6c936b76a91466a591b257d1beed442a81a27048b858e9619de088ac6c9352880124b268e2920300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 0; value = 70000
got witness stack of size 10
34 bytes (v0=P2WSH, v1=taproot/tapscript)
valid script
- generating prevout hash from 1 ins
[+] COutPoint(3885ecbff7, 0)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  |                                                                 0x
0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 |                                                                 0x
02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead |                                                                 0x
2                                                                  | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_CHECKMULTISIG                                                   |                                                                 0x
OP_IFDUP                                                           | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_NOTIF                                                           | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_DUP                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_HASH160                                                         | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
3839e1966487f13f03b1496004cade141a22e958                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0000 2
```

Stack was initilalized with:
* signature for Key E in recovery 2-of-3 path
* pubkey for Key E in recovery path
* signature for Key D in recovery 2-of-3 path
* pubkey for Key D in recovery path
* an empty signature for Key C in recovery 2-of-3 path
* pubkey for Key C in recovery 2-of-3 path
* an unused but to-be-consumed null byte for primary path
* an empty signature for Key A in primary path
* an empty signature for Key B in primary path

Next, `2` will be pushed (as in 2-of-n signatures)

```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0 |                                                                 02
02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead |                                                                 0x
2                                                                  |                                                                 0x
OP_CHECKMULTISIG                                                   |                                                                 0x
OP_IFDUP                                                           | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_NOTIF                                                           |                                                                 0x
OP_DUP                                                             | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_HASH160                                                         | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
3839e1966487f13f03b1496004cade141a22e958                           | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_EQUALVERIFY                                                     | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_CHECKSIG                                                        | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0001 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
```

The pubkey for Key A will be pushed
```
		<> PUSH stack 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead | 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
2                                                                  |                                                                 02
OP_CHECKMULTISIG                                                   |                                                                 0x
OP_IFDUP                                                           |                                                                 0x
OP_NOTIF                                                           |                                                                 0x
OP_DUP                                                             | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_HASH160                                                         |                                                                 0x
3839e1966487f13f03b1496004cade141a22e958                           | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_EQUALVERIFY                                                     | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_CHECKSIG                                                        | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_TOALTSTACK                                                      | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_DUP                                                             | 
OP_HASH160                                                         | 
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0002 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
```

The pubkey for Key B will be pushed
```
		<> PUSH stack 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_CHECKMULTISIG                                                   | 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
OP_IFDUP                                                           |                                                                 02
OP_NOTIF                                                           |                                                                 0x
OP_DUP                                                             |                                                                 0x
OP_HASH160                                                         |                                                                 0x
3839e1966487f13f03b1496004cade141a22e958                           | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_EQUALVERIFY                                                     |                                                                 0x
OP_CHECKSIG                                                        | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_TOALTSTACK                                                      | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_DUP                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_HASH160                                                         | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0003 2
```

The number of pubkeys, `2` will be pushed (as in m-of-2 pubkeys)
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKMULTISIG                                                   |                                                                 02
OP_IFDUP                                                           | 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
OP_NOTIF                                                           | 0224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0
OP_DUP                                                             |                                                                 02
OP_HASH160                                                         |                                                                 0x
3839e1966487f13f03b1496004cade141a22e958                           |                                                                 0x
OP_EQUALVERIFY                                                     |                                                                 0x
OP_CHECKSIG                                                        | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_TOALTSTACK                                                      |                                                                 0x
OP_DUP                                                             | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_HASH160                                                         | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
d6e8275bc822523c5c58386346030727a05ebb49                           | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_EQUALVERIFY                                                     | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0004 OP_CHECKMULTISIG
```

The multisig is checked for validity (7 items consumed, `0` pushed since invalid)
```
stack has 13 entries [require 1]
stack has 13 entries [require 4]
stack has 13 entries [require 7]
scriptCode = 52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268
looping for multisig
loop: sigs = 2, keys = 2
- got sig 
- got key 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
GenericTransactionSignatureChecker::CheckECDSASignature(0 len sig, 33 len pubkey, sigversion=1)
  sig         = 
  pub key     = 02cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead
  script code = 52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268
- failed: signature is empty
- sig check failed
loop ended in failure state
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> PUSH stack 
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_IFDUP                                                           |                                                                 0x
OP_NOTIF                                                           | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_DUP                                                             |                                                                 0x
OP_HASH160                                                         | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
3839e1966487f13f03b1496004cade141a22e958                           | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_EQUALVERIFY                                                     | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_CHECKSIG                                                        | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0005 OP_IFDUP
```

since false, top stack item is NOT duplicated
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NOTIF                                                           |                                                                 0x
OP_DUP                                                             | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_HASH160                                                         |                                                                 0x
3839e1966487f13f03b1496004cade141a22e958                           | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_EQUALVERIFY                                                     | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_CHECKSIG                                                        | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_TOALTSTACK                                                      | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_DUP                                                             | 
OP_HASH160                                                         | 
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0006 OP_NOTIF
```

The top item is consumed by OP_NOTIF which since false allows following branch to execute
```
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_DUP                                                             | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_HASH160                                                         |                                                                 0x
3839e1966487f13f03b1496004cade141a22e958                           | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_EQUALVERIFY                                                     | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_CHECKSIG                                                        | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_TOALTSTACK                                                      | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_DUP                                                             | 
OP_HASH160                                                         | 
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0007 OP_DUP
```

Top of stack's pubkey is duplicated
```
		<> PUSH stack 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_HASH160                                                         | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
3839e1966487f13f03b1496004cade141a22e958                           | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_EQUALVERIFY                                                     |                                                                 0x
OP_CHECKSIG                                                        | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_TOALTSTACK                                                      | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_DUP                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_HASH160                                                         | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
d6e8275bc822523c5c58386346030727a05ebb49                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0008 OP_HASH160
```

Replaced with hash160 of the pubkey
```
		<> POP  stack
		<> PUSH stack 3839e1966487f13f03b1496004cade141a22e958
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
3839e1966487f13f03b1496004cade141a22e958                           |                           3839e1966487f13f03b1496004cade141a22e958
OP_EQUALVERIFY                                                     | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_CHECKSIG                                                        |                                                                 0x
OP_TOALTSTACK                                                      | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_DUP                                                             | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_HASH160                                                         | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
d6e8275bc822523c5c58386346030727a05ebb49                           | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0009 3839e1966487f13f03b1496004cade141a22e958
```

hash160 from witness_script is pushed
```
		<> PUSH stack 3839e1966487f13f03b1496004cade141a22e958
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                                                     |                           3839e1966487f13f03b1496004cade141a22e958
OP_CHECKSIG                                                        |                           3839e1966487f13f03b1496004cade141a22e958
OP_TOALTSTACK                                                      | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_DUP                                                             |                                                                 0x
OP_HASH160                                                         | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
d6e8275bc822523c5c58386346030727a05ebb49                           | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_EQUALVERIFY                                                     | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_CHECKSIG                                                        | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0010 OP_EQUALVERIFY
```

...and compared without failing (leaving stack unchanged), continuing
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        | 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
OP_TOALTSTACK                                                      |                                                                 0x
OP_DUP                                                             | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_HASH160                                                         | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
d6e8275bc822523c5c58386346030727a05ebb49                           | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_EQUALVERIFY                                                     | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0011 OP_CHECKSIG
```

Signature check is performed, failing, pushing `0`
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(0 len sig, 33 len pubkey, sigversion=1)
  sig         = 
  pub key     = 02db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
  script code = 52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268
- failed: signature is empty
		<> POP  stack
		<> POP  stack
		<> PUSH stack 
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_TOALTSTACK                                                      |                                                                 0x
OP_DUP                                                             | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_HASH160                                                         | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
d6e8275bc822523c5c58386346030727a05ebb49                           | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_EQUALVERIFY                                                     | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0012 OP_TOALTSTACK
```

Top `0` saved to the altstack
```
		<> PUSH altstack 
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_DUP                                                             | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_HASH160                                                         | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
d6e8275bc822523c5c58386346030727a05ebb49                           | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_EQUALVERIFY                                                     | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0013 OP_DUP
```

Top pubkey is duplicated
```
		<> PUSH stack 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_HASH160                                                         | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
d6e8275bc822523c5c58386346030727a05ebb49                           | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_EQUALVERIFY                                                     | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_CHECKSIG                                                        | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_FROMALTSTACK                                                    | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_ADD                                                             | 
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0014 OP_HASH160
```

Replaced with hash160 of the pubkey
```
		<> POP  stack
		<> PUSH stack d6e8275bc822523c5c58386346030727a05ebb49
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
d6e8275bc822523c5c58386346030727a05ebb49                           |                           d6e8275bc822523c5c58386346030727a05ebb49
OP_EQUALVERIFY                                                     | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_CHECKSIG                                                        | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_FROMALTSTACK                                                    | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_ADD                                                             | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_TOALTSTACK                                                      | 
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0015 d6e8275bc822523c5c58386346030727a05ebb49
```

hash160 from witness_script is pushed
```
		<> PUSH stack d6e8275bc822523c5c58386346030727a05ebb49
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                                                     |                           d6e8275bc822523c5c58386346030727a05ebb49
OP_CHECKSIG                                                        |                           d6e8275bc822523c5c58386346030727a05ebb49
OP_FROMALTSTACK                                                    | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_ADD                                                             | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_TOALTSTACK                                                      | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_DUP                                                             | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0016 OP_EQUALVERIFY
```

...and compared without failing (leaving stack unchanged), continuing
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        | 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
OP_FROMALTSTACK                                                    | 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7e...
OP_ADD                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_TOALTSTACK                                                      | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0017 OP_CHECKSIG
```

Signature check is performed, pushing `1`
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7ed6346f59d02206d9d027e2e6c865ebff2f079f50aae5473d8e7a49f5fe9f0ceba46541720744101
  pub key     = 03876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
  script code = 52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=70000)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = b6265c67751b701c1fa3d0caf2f7bfad929040df244590997a6c2e50f36d6268
  pubkey.VerifyECDSASignature(sig=30440220337355cf9d00ee9db4db4dcbe525fc38f2b7852ab99346ae5005f7ed6346f59d02206d9d027e2e6c865ebff2f079f50aae5473d8e7a49f5fe9f0ceba465417207441, sighash=b6265c67751b701c1fa3d0caf2f7bfad929040df244590997a6c2e50f36d6268):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_FROMALTSTACK                                                    |                                                                 01
OP_ADD                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_TOALTSTACK                                                      | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_DUP                                                             | 
OP_HASH160                                                         | 
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0018 OP_FROMALTSTACK
```

Previous failed signature check copied back from altstack
```
		<> PUSH stack 
		<> POP  altstack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_ADD                                                             |                                                                 0x
OP_TOALTSTACK                                                      |                                                                 01
OP_DUP                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_HASH160                                                         | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0019 OP_ADD
```

Top 2 items are added to keep track of verified signatures
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_TOALTSTACK                                                      |                                                                 01
OP_DUP                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_HASH160                                                         | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0020 OP_TOALTSTACK
```

...and saved on the altstack
```
		<> PUSH altstack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_DUP                                                             | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_HASH160                                                         | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 
OP_EQUALVERIFY                                                     | 
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0021 OP_DUP
```

Next pubkey is duplicated
```
		<> PUSH stack 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_HASH160                                                         | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_EQUALVERIFY                                                     | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_CHECKSIG                                                        | 
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0022 OP_HASH160
```

and replaced with hash160
```
		<> POP  stack
		<> PUSH stack 43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15                           |                           43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15
OP_EQUALVERIFY                                                     | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_CHECKSIG                                                        | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_FROMALTSTACK                                                    | 
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0023 43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15
```

last hash160 from witness_script is pushed
```
		<> PUSH stack 43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                                                     |                           43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15
OP_CHECKSIG                                                        |                           43cfea2eefd37233bdce5d8f1d7ea3cc4926eb15
OP_FROMALTSTACK                                                    | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_ADD                                                             | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0024 OP_EQUALVERIFY
```

...and compared without failing (leaving stack unchanged), continuing
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        | 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
OP_FROMALTSTACK                                                    | 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd06...
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0025 OP_CHECKSIG
```

Last signtaure check is performed, pushing `1`
```
EvalChecksig() sigversion=1
Eval Checksig Pre-Tapscript
GenericTransactionSignatureChecker::CheckECDSASignature(71 len sig, 33 len pubkey, sigversion=1)
  sig         = 30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd0681be07b3102207c580390907855c5343ebff38e6374bce8bbbc3d50194f957f0ed2245724401701
  pub key     = 03f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
  script code = 52210224a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d02102cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead52ae736476a9143839e1966487f13f03b1496004cade141a22e95888ac6b76a914d6e8275bc822523c5c58386346030727a05ebb4988ac6c936b76a91443cfea2eefd37233bdce5d8f1d7ea3cc4926eb1588ac6c9352880124b268
  hash type   = 01 (SIGHASH_ALL)
SignatureHash(nIn=0, nHashType=01, amount=70000)
- sigversion == SIGVERSION_WITNESS_V0
  sighash     = b6265c67751b701c1fa3d0caf2f7bfad929040df244590997a6c2e50f36d6268
  pubkey.VerifyECDSASignature(sig=30440220093be4b35069f97aa7037958644ff4c61c17ef380233d15c49dcd0681be07b3102207c580390907855c5343ebff38e6374bce8bbbc3d50194f957f0ed22457244017, sighash=b6265c67751b701c1fa3d0caf2f7bfad929040df244590997a6c2e50f36d6268):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_FROMALTSTACK                                                    |                                                                 01
OP_ADD                                                             | 
2                                                                  | 
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0026 OP_FROMALTSTACK
```

Previous sum of 1 copied from altstack
```
		<> PUSH stack 01
		<> POP  altstack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_ADD                                                             |                                                                 01
2                                                                  |                                                                 01
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0027 OP_ADD
```

Top 2 items are added to keep track of 2 verified signatures
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  |                                                                 02
OP_EQUALVERIFY                                                     | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0028 2
```

`2` is pushed, as in 2-of-m signatures
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_EQUALVERIFY                                                     |                                                                 02
24                                                                 |                                                                 02
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0029 OP_EQUALVERIFY
```

Top 2 items are compared equal, continuing
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
OP_ENDIF                                                           | 
#0030 24
```

0x24 (as in 36 block confirmations) is pushed
```
		<> PUSH stack 24
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSEQUENCEVERIFY                                             |                                                                 24
OP_ENDIF                                                           | 
#0031 OP_CHECKSEQUENCEVERIFY
```

OP_CSV rules are consulted, which in this case do not fail, continuing
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_ENDIF                                                           |                                                                 24
#0032 OP_ENDIF
```

OP_ENDIF is reached and we are done
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 24
```


Since the script completed without failure and left a non-zero value on top of the stack... this transaction is valid.
