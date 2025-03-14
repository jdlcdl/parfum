# Bitcoin Core Watch-Only Liana Expanding-Multi TR

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

We'll be using "less than obvious" labels for our secrets and keys because we're aware that our Liana Miniscript descriptor will be ordering our xpubs so that recovery keys are presented first and primary keys are presented last.  Using an altered order here will be more useful once we load the descriptor in Krux, which will label keys A-F in the order they are presented within the descriptor.  Note: We won't be using A, because it will be defined by Liana as the internal NUMS taproot key. Our recovery path will use pubkeys from B, C and D while the primary path will use pubkeys from E and F.

### B and E (recovery + primary)

**BIP-39 mnemonic**: `auction crucial trend safe faith barrel orbit roast source stereo discover cart`

**BIP-39 passphrase**: ""

### C and F (recovery + primary)

**BIP-39 mnemonic**: `orange enter age rug chef denial legend topic identify sign always mother`

**BIP-39 passphrase**: ""

### D (recovery only)

**BIP-39 mnemonic**: `news lecture under adapt inspire chunk tongue fun party build defense receive`

**BIP-39 passphrase**: ""

![secrets-exp-multi-tr](secrets-exp-multi-tr.png)

---

## Extended Public Keys

### B and E (recovery + primary)

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

### C and F (recovery + primary)

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

### D (recovery only)

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

We will use Liana to create a Taproot based Expanding Multisig wallet such that the primary spending path will be a 2-of-2 multisig using keys E and F, and after some time it will expand to a 2-of-3 multisig -- reusing the same keys relabeled as B and C, and including recovery key D.  We'll use "Advanced" to select "Taproot", then using each of the above XPUBs w/ key-origin, we'll cut-n-paste them into Liana, and we'll edit the recovery expiration so that each UTXO can be recovered after 6h (36 confirmations).  Finally, Liana will ask us to backup "The descriptor"

```
tr(tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/<0;1>/*,{and_v(v:multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*),older(36)),multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*)})#tvh3u2lu
```
Let's take a moment to inspect this Liana descriptor.  We'll add whitespace and indentation for readability.
```
tr(
  tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/<0;1>/*,
  {
    and_v(
      v:multi_a(
        2,
        [07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*,
        [da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*,
        [cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*
      ),
      older(36)
    ),
    multi_a(
      2,
      [07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,
      [da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*
    )
  }
)#tvh3u2lu
```
* the Miniscript is wrapped in `tr()` with a `#` and bech32 checksum appended
* the mysterious first key-expression is the internal taproot key, aka: NUMS (nothing up my sleeve), it is provably unspendable
* our two tapscript paths are in reverse order: first the time-delayed recovery 2-of-3 multisig, then the primary 2-of-2 multisig
* we find no `or()` joining any conditions, taproot implies that the taproot key OR any of the tapscript paths are "good enough"
* our recovery path requires 2 terms, joined by `and()` (both must succeed)
* first recovery term: `multi()` is NOT sorted and our recovery key has been added, order of keys matter
* since our xpubs from the primary path are both used twice, as `B` and `C` in the recovery path, they'll derive their receive and change pubkeys from bip32 child nodes `2` and `3` respectively
* recovery-only key `D` will derive receive and change pubkeys from its bip32 child nodes `0` and `1` respectively
* second recovery term: `older(36)` is our relatve time-delay so that spending each input requires committing to doing so only after 36 confirmations.
* our primary path is a standard 2-of-2 `multi()`. It is NOT sorted, so the order of keys matter
* both keys in our primary path will derive receive and change pubkeys via their bip32 child nodes `0` and `1` respectively

### Provably-unspendable NUMS tapkey
Above we mentioned a provably unspendable NUMS internal tapkey.  We can verify using a simple script, thanks to existing Krux code (or krux will check and inform the user when loading this descriptor).
```bash
# set some env variables to work with below
~/bclipy$ descriptor="tr(tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/<0;1>/*,{and_v(v:multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*),older(36)),multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*)})#tvh3u2lu"

~/bclipy$ B=tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5
~/bclipy$ E=$B

~/bclipy$ C=tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5
~/bclipy$ F=$C

~/bclipy$ D=tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2

# does this descriptor have a provably-unspendable NUMS tapkey?
~/bclipy$ ./bip341_nums_tapkey.py $descriptor
Descriptor has provably-unspendable NUMS tapkey? True

# using OUR xpubs in the order they are presented in the descriptor, calculate the provably-unspendable tapkey?
~/bclipy$ ./bip341_nums_tapkey.py $B $C $D $E $F
Provably-unspendable NUMS tapkey: tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN
# The first XPUB in our descriptor should match the provably-unspendable xpub calculated just above
```

### Split the descriptor into receive and change `descriptors list` for Bitcoin Core

```python
>>> descriptor = "tr(tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/<0;1>/*,{and_v(v:multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<2;3>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<2;3>/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/<0;1>/*),older(36)),multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/<0;1>/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/<0;1>/*)})#tvh3u2lu"

>>> import bip380_checksum

>>> descriptors = [
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/0/*").replace("/<2;3>/*", "/2/*").split("#")[0]
    ),
    bip380_checksum.descsum_create(
        descriptor.replace("/<0;1>/*", "/1/*").replace("/<2;3>/*", "/3/*").split("#")[0]
    )]

>>> print(repr(descriptors))
["tr(tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/0/*,{and_v(v:multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/2/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/2/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*),older(36)),multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*)})#cnkkfylh", "tr(tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/1/*,{and_v(v:multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/3/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/3/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/1/*),older(36)),multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/1/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/1/*)})#8rajr06n"]
```

---

## Watch-Only Wallet

Instead of using `bitcoin-cli` at the command line, we'll be connecting to Bitcoin Core with python-bitcoinrpc as is done in this [BlockchainCommons tutorial](https://github.com/BlockchainCommons/Learning-Bitcoin-from-the-Command-Line/blob/master/18_4_Accessing_Bitcoind_with_Python.md).  If you're more familiar with `bitcoin-cli`, then know that all `rpc.method_name()` calls below can be made like `bitcoin-cli command` with similar paramaters.

In the examples below, `rpc.` is an authproxy object that is still connected to `bitcoind` -- thanks to an exaggerated configuration setting `rpcservertimeout=600` so that connections remain open for 10m, instead of closing once idle for 30s.

```python
# create the wallet
>>> rpc.createwallet(
    "Liana-EM-tr",  # wallet_name
    True,  # disable_private_keys
    True,  # blank
    "",  # passphrase
    True,  # avoid_reuse
    True,  # descriptors
    False,  # load_on_startup
    False)  # external_signer
{'name': 'Liana-EM-tr', 'warnings': ['Empty string given as passphrase, wallet will not be encrypted.']}

# import its descriptors
>>> rpc.importdescriptors([dict(desc=x, timestamp="now") for x in descriptors])
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]

# view the wallet
>>> pprint(rpc.getwalletinfo())
[{'success': True, 'warnings': ['Range not given, using default keypool range']}, {'success': True, 'warnings': ['Range not given, using default keypool range']}]
>>> pprint(rpc.getwalletinfo())
{'avoid_reuse': True,
 'balance': Decimal('0E-8'),
 'birthtime': 1738845756,
 'blank': True,
 'descriptors': True,
 'external_signer': False,
 'format': 'sqlite',
 'immature_balance': Decimal('0E-8'),
 'keypoolsize': 0,
 'keypoolsize_hd_internal': 0,
 'lastprocessedblock': {'hash': '000000df96ac60e79ce20de3dbb351944850cabf38ea3d8f31a2b183c8167147',
                        'height': 234205},
 'paytxfee': Decimal('0E-8'),
 'private_keys_enabled': False,
 'scanning': False,
 'txcount': 0,
 'unconfirmed_balance': Decimal('0E-8'),
 'walletname': 'Liana-EM-tr',
 'walletversion': 169900}

# view first 10 receive addresses
>>> pprint(rpc.deriveaddresses(descriptors[0], 9))
['tb1pxhffvmstgs0yqcjg7nkc56dx9q6nd2z7cq7fw3589yv5sfjuacmshqgewc',
 'tb1pghk3qtaykyzhmd7gcu6kh7y954n7yh296wc3n5g3mjpf5dg3ctzq2dqe8y',
 'tb1pmkvw09wv8zaeskqf753lxftnp5rc3ezt7f97szccrw7wglcxlqgs9wha2u',
 'tb1pfdhz9wgctrgvadka0nf3secsrm5m34jwvg86n6vt3j476uzpqq4styadcp',
 'tb1p69kvf8ylkvj8we4ec7873nnhpl2fgqsa3cw8cmu0zz79w6yjg63q3alve7',
 'tb1psw0gn8cxmjgaktt76kr4wr9lsgxulu6cq7e9hhyggk7y7m0y9r2q4rzwq5',
 'tb1pzqq6e8xr44eue5k5llgpnxmtws7m7s7a2a0lull2kwpm4ztmsexqxu79y8',
 'tb1p6v78raeemwvw33e0s4hvlklu27h5ulf6yfksvp2v56wp6vxly5dsfpcrxv',
 'tb1p85p038mv2x9zls53z0jnurdk6fmsuypl5d0e807kke39w8ezpddsdpwv6d',
 'tb1pwq98hwy286s9ygkpkl7yeh9g8z9wlykavy53kvy3j4uq44cwf87qm3f7re']

# view first 10 change addresses
>>> pprint(rpc.deriveaddresses(descriptors[1], 9))
['tb1pm9nalx2emzp0ujlf7d8ktfrm5f7pgjlff8azyfe5x5xz6kevsj4qhltz9l',
 'tb1p3wa5jnvd5zq8d6xy2e3zndwv3658ll2d2lkhf8drcy33mr88ndvqyew4s3',
 'tb1pxpns5ehxck2jn8dn00v3ad5mm52au8hcevvdjntte0wn85uv3nsqkke5lf',
 'tb1pxyz3alx7438ev7rnqaumlgu483akwvz4h2aw2wgn8dt2tsfxy7nse8k78x',
 'tb1pxsj49z4wmwkkn5s05s53kv5mszx6q8wctjzqt3v508e00k567neqfw5thm',
 'tb1pd323clc3pnwc2v68f8r44jjxrmkse9q9t87l3zxtkr74c6097pgqqjw2dz',
 'tb1p67heg797unwy03x4k2nhklmdmcq9fmy3selzdar8fnrjztr452ms8qrjdv',
 'tb1ptah0h46n2zdlvdyjyc0p0q5rx7t08yp87crpgzuntnd6g37vq6asqtgpq3',
 'tb1pvz7vuq3gsefevhldv34a5y74dzxf8gezh9fumlcra3npvedk43cqeym2mv',
 'tb1pneycsvtfleqw8aqz7x652gwx9tlx3z7dwak34nr8ae8d2l4dsvaqzrjsrf']

#
# funded first receive address from alt.signetfaucet.com
# recycle-to: tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn
#

# view UTXOs
>>> pprint(rpc.listunspent())
[{'address': 'tb1pxhffvmstgs0yqcjg7nkc56dx9q6nd2z7cq7fw3589yv5sfjuacmshqgewc',
  'amount': Decimal('0.00200000'),
  'confirmations': 3,
  'desc': 'tr([7c461e5d/0/0]baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183,{multi_a(2,[07fd816d/48h/1h/0h/2h/0/0]3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b,[da855a1f/48h/1h/0h/2h/0/0]46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6),and_v(v:multi_a(2,[07fd816d/48h/1h/0h/2h/2/0]02c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852,024cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9,[cdef7cd9/48h/1h/0h/2h/0/0]029dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c),older(36))})#6dae2t7x',
  'parent_descs': ["tr(tpubD6NzVbkrYhZ4X6BRkDMxFyZxfUCQdjpK27dNgqwDqsQ2PUbMmjjPPFxfcTJiGEjeNz2zLbZ1PRmgCAzXn4pE6tEuQPScXyUbuAgdcec6pMN/0/*,{and_v(v:multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/2/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/2/*,[cdef7cd9/48'/1'/0'/2']tpubDEzdWp7365AFAExeUsHiwRmkZN5it3sSAZsd6GKUXvUiJBytXnZrRKMAt9UgCkWB2mP3K9WujLuTjrRLBn51Y18pMVyg2v18un4ivqWSAk2/0/*),older(36)),multi_a(2,[07fd816d/48'/1'/0'/2']tpubDDvFWduSiwhW7hUbL1oMyUfcNgeSyZgHbooe1WjHyRaXYH3uUjm1xdxWXAGbQFn8QGScDg4b4a6WMGNiEAq2uQdmPDhDKPE5Dr8DX24mwd5/0/*,[da855a1f/48'/1'/0'/2']tpubDEHRt73d4guqR5BLGQud4XMW8vDCGHUj54qDTFtsdFstF6PAYx1oAy3jfKg1PffqLUWuSsXmnetKeTJFKfKLXeJR97yUuqvvojnoBcUDHg5/0/*)})#cnkkfylh"],
  'reused': False,
  'safe': True,
  'scriptPubKey': '512035d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37',
  'solvable': True,
  'spendable': True,
  'txid': 'cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619',
  'vout': 0}]
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
>>> send_addr = "tb1qce67pd95y73nccs5wup0duc4f9an8ltvuyh3ns"
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
  'vout': 0}]

>>> pprint(txout)
[{'tb1pghk3qtaykyzhmd7gcu6kh7y954n7yh296wc3n5g3mjpf5dg3ctzq2dqe8y': Decimal('0.0007')},
 {'tb1qce67pd95y73nccs5wup0duc4f9an8ltvuyh3ns': Decimal('0.00075')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00053440')}]

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'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'

# result of signing by external signer (in this case, krux)
>>> psbtE = "cHNidP8BAJwCAAAAARn2BebNN91deA4Vxx1X1OfCPepWtkuTeLMRrdxUC8LPAAAAAAD9////A3ARAQAAAAAAIlEgRe0QL6SxBX23yMc1a/iFpWfiXUXTsRnREdyCmjURwsT4JAEAAAAAABYAFMZ14LS0J6M8YhR3AvbzFUl7M/1swNAAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yeOSAwAAAQErQA0DAAAAAAAiUSA10pZuC0QeQGJI9O2KaaYoNTaoXsA8l0aHKRlIJlzuN0EUPDQ+Bjgbfnkm/b5UxjqwBAxcLISR4tS9cjjLkGYEChuKcBo2SbPuTXQB978V/mYeKskLaKO7XOZzhsAljQ3R80AeEsA79j1B+Hov15K3f6+Ba2X3IEg9IniY+PQe/mlvIqOLr58k5LzyENu9rv1T2KKmO9pWADiFRRdzgN6Yi0KqQRTHw0oOGzxF2SPrpubX+pB2upANKimku5vhAFtb5e/4UtF0LmFheCeZ1ERnqw3DdLHLiLwvcZ5EvEn56lRcntJVQLmYpB9Vx+83mQtELs9qaQVtg8uUPYDKtILB0ONP1vPDe5C20teJY4uITMNh17RQE09mf+zNQ58jKsg2gb407CwAAAAA"

>>> psbtF = "cHNidP8BAJwCAAAAARn2BebNN91deA4Vxx1X1OfCPepWtkuTeLMRrdxUC8LPAAAAAAD9////A3ARAQAAAAAAIlEgRe0QL6SxBX23yMc1a/iFpWfiXUXTsRnREdyCmjURwsT4JAEAAAAAABYAFMZ14LS0J6M8YhR3AvbzFUl7M/1swNAAAAAAAAAWABREf94eN9lyVbWCHS3ugW6PGPa6yeOSAwAAAQErQA0DAAAAAAAiUSA10pZuC0QeQGJI9O2KaaYoNTaoXsA8l0aHKRlIJlzuN0EURgI7eHLgAnV32KRROJvFpDDnKfLTiaI+zKEjfcO399aKcBo2SbPuTXQB978V/mYeKskLaKO7XOZzhsAljQ3R80Dw7M2V763XQPvhj5OkRBty2RAAIuw2TheiOj7n/wo7bwyZVNpCpY0INIApdOKRz6/jGC8Y7CXK+sMH7IE9Ll7TQRRM9hhZs4bC8ENshazzrIWv0RA9zkRu4fWLY/11YaMb+dF0LmFheCeZ1ERnqw3DdLHLiLwvcZ5EvEn56lRcntJVQAExdOPgt6PrKHkSCalyKbatMRoSFRDWcifZo+MutnVNLuHGeFxWOlfqJl6S8BT4CGVXuBn8soCNqGx4+2lsxLYAAAAA"

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, psbtE, psbtF])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101)

---

## Primary: Evolution of a PSBT

In the above session, the PSBT evolves with each step. Below these changes are explained.

### `psbt = rpc.createpsbt(...`
Bitcoin Core creates the initial psbt template, knowing only the input's `txid` and `vout`, but including no information about what that input is worth or what the spending conditions might be.  Note: because this is segwit, the final pre-segwit `txid` is already known, but not the `hash`.
```json
{
  "tx": {
    "txid": "6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101",
    "hash": "6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101",
    "version": 2,
    "size": 156,
    "vsize": 156,
    "weight": 624,
    "locktime": 234211,
    "vin": [
      {
        "txid": "cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619",
        "vout": 0,
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
          "asm": "1 45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
          "desc": "rawtr(45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4)#tzvrwcm4",
          "hex": "512045ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
          "address": "tb1pghk3qtaykyzhmd7gcu6kh7y954n7yh296wc3n5g3mjpf5dg3ctzq2dqe8y",
          "type": "witness_v1_taproot"
        }
      },
      {
        "value": 0.00075000,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 c675e0b4b427a33c62147702f6f315497b33fd6c",
          "desc": "addr(tb1qce67pd95y73nccs5wup0duc4f9an8ltvuyh3ns)#qqpccvs2",
          "hex": "0014c675e0b4b427a33c62147702f6f315497b33fd6c",
          "address": "tb1qce67pd95y73nccs5wup0duc4f9an8ltvuyh3ns",
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
          "asm": "1 35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37",
          "desc": "rawtr(35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37)#68xmqjkh",
          "hex": "512035d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37",
          "address": "tb1pxhffvmstgs0yqcjg7nkc56dx9q6nd2z7cq7fw3589yv5sfjuacmshqgewc",
          "type": "witness_v1_taproot"
        }
      },
      "taproot_scripts": [
        {
          "script": "203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c",
          "leaf_ver": 192,
          "control_blocks": [
            "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "script": "20c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852ac204cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9ba209dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5cba529d0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd8121838a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        },
        {
          "pubkey": "46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        },
        {
          "pubkey": "4cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "pubkey": "9dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "pubkey": "baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/0",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        }
      ],
      "taproot_internal_key": "baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183",
      "taproot_merkle_root": "e5820285d82dbfaa2d201188445d1ac9f1e92768092eb72a96f83f7b5d660950"
```
Also, it fills `outputs` for those it knows about
```json
      "witness_utxo": {
        "amount": 0.00200000,
        "scriptPubKey": {
          "asm": "1 35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37",
          "desc": "rawtr(35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37)#68xmqjkh",
          "hex": "512035d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37",
          "address": "tb1pxhffvmstgs0yqcjg7nkc56dx9q6nd2z7cq7fw3589yv5sfjuacmshqgewc",
          "type": "witness_v1_taproot"
        }
      },
      "taproot_scripts": [
        {
          "script": "203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c",
          "leaf_ver": 192,
          "control_blocks": [
            "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "script": "20c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852ac204cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9ba209dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5cba529d0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd8121838a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        },
        {
          "pubkey": "46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        },
        {
          "pubkey": "4cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "pubkey": "9dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "pubkey": "baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/0",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        }
      ],
      "taproot_internal_key": "baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183",
      "taproot_merkle_root": "e5820285d82dbfaa2d201188445d1ac9f1e92768092eb72a96f83f7b5d660950"
```
Lastly, knowing the inputs, it calculates and adds the `fee`
```json
   ,
  "fee": 0.00001560
```

### `signed_psbt = ...`
It is the job of the Signer to display pertinent information to the user so that they understand what is being signed, and to add signatures to the PSBT.  As signed by secret E, krux adds two signatures to the existing dictionary in `inputs`.  The reason that  two signatures are added is because the same loaded secret is capable of signing both the primary and recovery path, but it is not aware of which path will complete the PSBT, so that will be done by the Finalizer.
```json
      "taproot_script_path_sigs": [
        {
          "pubkey": "3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "leaf_hash": "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3",
          "sig": "1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa"
        },
        {
          "pubkey": "c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852",
          "leaf_hash": "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255",
          "sig": "b998a41f55c7ef37990b442ecf6a69056d83cb943d80cab482c1d0e34fd6f3c37b90b6d2d789638b884cc361d7b450134f667feccd439f232ac83681be34ec2c"
        }
      ]
```
As signed by secret F, krux does similar
```json
      "taproot_script_path_sigs": [
        {
          "pubkey": "46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "leaf_hash": "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3",
          "sig": "f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3"
        },
        {
          "pubkey": "4cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9",
          "leaf_hash": "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255",
          "sig": "013174e3e0b7a3eb28791209a97229b6ad311a121510d67227d9a3e32eb6754d2ee1c6785c563a57ea265e92f014f8086557b819fcb2808da86c78fb696cc4b6"
        }
      ]
```

The Signer's role in the PSBT is simply to "add" signatures, and done.  In fact,
[BIP-174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki#signer)
clearly states: "The Signer must only add data to a PSBT.", but some rules were meant to be broken?!?!

Because an air-gapped Signer which transmits data via QR-Code is constrained by this low-bandwidth medium, some have "optimized" to transfer less -- by stripping unnecessary fields from `inputs` and `outputs`.  Krux strips `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root` from `inputs`.  It also strips `taproot_internal_key`, `taproot_tree` and `taproot_bip32_derivs` from `outputs`.

This is a non-standard `feature` of some air-gapped-via-qrcode Signers which is NOT practiced when a signed PSBT can be written efficiently to an sdcard or transfered electronically.  As expected, this non-standard feature must also be tolerated by whichever software is acting in the role of "Combiner", as we'll see below.

### `combined_psbt = rpc.combinepsbt(...`
Had the air-gapped signer not stripped any fields in its quest for an optimized transmit-via-qrocde user-experience, the resulting `signed_psbt` would have been the same as the `combined_psbt`.  Still, because other types of transactions may require multiple signatures from multiple Signers, it is important for the Combiner to consider combining all signed PSBTs with the original PSBT distributed to Signers.  In this case, Bitcoin Core simply replaces the dictionary keys that had been stripped out.

Bitcoin Core combines signatures and restores stripped fields from the signed PSBTs.  In `inputs` it restores
```json
      "taproot_scripts": [
        {
          "script": "203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c",
          "leaf_ver": 192,
          "control_blocks": [
            "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "script": "20c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852ac204cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9ba209dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5cba529d0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd8121838a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        },
        {
          "pubkey": "46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0dd1f3"
          ]
        },
        {
          "pubkey": "4cf61859b386c2f0436c85acf3ac85afd1103dce446ee1f58b63fd7561a31bf9",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "pubkey": "9dd045f868fcf8cfa0d194f9fec990d7d0bdf01d2a42e494e876ecb6dd482f5c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        },
        {
          "pubkey": "baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/0",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "c7c34a0e1b3c45d923eba6e6d7fa9076ba900d2a29a4bb9be1005b5be5eff852",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/0",
          "leaf_hashes": [
            "d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
          ]
        }
      ],
      "taproot_internal_key": "baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183",
      "taproot_merkle_root": "e5820285d82dbfaa2d201188445d1ac9f1e92768092eb72a96f83f7b5d660950"
```
and in `outputs` it restores
```json
      "taproot_internal_key": "15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e",
      "taproot_tree": [
        {
          "depth": 1,
          "leaf_ver": 192,
          "script": "2024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadba529c"
        },
        {
          "depth": 1,
          "leaf_ver": 192,
          "script": "20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        },
        {
          "pubkey": "876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        },
        {
          "pubkey": "db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "pubkey": "f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        }
      ]
```

### `finalized_psbt = rpc.finalizepsbt(...`
As the Finalizer role, Bitcoin Core is tasked to determine whether or not the PSBT is complete and final, by verifying that spending conditions for all inputs have been met.  In this case, finalizing the complete PSBT strips-out previously added `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root` replacing them with `final_scriptwitness` within the `inputs` field.
```json
      "final_scriptwitness": [
        "f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3",
        "1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa",
        "203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c",
        "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
      ]
```

By default, when the PSBT is complete and final, Bitcoin Core's `finalizepsbt` will also extract the final rawtransaction hex so that it's ready to be broadcasted into the mempool.  Once extracted, the transaction's `hash` is finally known, while the pre-segwit `txid` had never changed.

```json
{
  "txid": "6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101",
  "hash": "5f738275c196c08e0bedb29f458d97d884d57cc7116013c0068c8d9414d4c506",
  "version": 2,
  "size": 426,
  "vsize": 224,
  "weight": 894,
  "locktime": 234211,
  "vin": [
    {
      "txid": "cfc20b54dcad11b378934bb656ea3dc2e7d4571dc7150e785ddd37cde605f619",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3",
        "1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa",
        "203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c",
        "c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255"
      ],
      "sequence": 4294967293
    }
  ],
  "vout": [
    {
      "value": 0.00070000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
        "desc": "rawtr(45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4)#tzvrwcm4",
        "hex": "512045ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
        "address": "tb1pghk3qtaykyzhmd7gcu6kh7y954n7yh296wc3n5g3mjpf5dg3ctzq2dqe8y",
        "type": "witness_v1_taproot"
      }
    },
    {
      "value": 0.00075000,
      "n": 1,
      "scriptPubKey": {
        "asm": "0 c675e0b4b427a33c62147702f6f315497b33fd6c",
        "desc": "addr(tb1qce67pd95y73nccs5wup0duc4f9an8ltvuyh3ns)#qqpccvs2",
        "hex": "0014c675e0b4b427a33c62147702f6f315497b33fd6c",
        "address": "tb1qce67pd95y73nccs5wup0duc4f9an8ltvuyh3ns",
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
```
    {
      "value": 0.00200000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37",
        "desc": "rawtr(35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37)#68xmqjkh",
        "hex": "512035d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37",
        "address": "tb1pxhffvmstgs0yqcjg7nkc56dx9q6nd2z7cq7fw3589yv5sfjuacmshqgewc",
        "type": "witness_v1_taproot"
      }
    },
```

Below, we will start a `btcdeb` session at the bash commandline using the rawtransaction hex of the above transaction, as well as the same for this transaction's only input.
```
btcdeb --quiet --tx=0200000000010119f605e6cd37dd5d780e15c71d57d4e7c23dea56b64b9378b311addc540bc2cf0000000000fdffffff03701101000000
000022512045ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4f824010000000000160014c675e0b4b427a33c62147702f6f315497b33fd6c
c0d0000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90440f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954
da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3401e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4
bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa46203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0
027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c41c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161
782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255e3920300 --txin=0200000000010103dc016b3823bb1d01244ffce1a1117704183ced100b51601b70
d694213c88c90000000000fdffffff03400d03000000000022512035d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37400d03000000000022
002078e59302d5539fe4c7191c10ad246f856f75e80f07a30ad419993579c02e44d773424a0000000000160014b7e7f48e5c3bd445be101eb5835021219ced250c024730
4402207768183ffd882c935d403f822b88c625d36687005d36707acd9a6e157f5a0b1c022056ab74972e275df0ddcc6c1bec45eda7a3ad8a73db7a92835a5cf493b31e91
730121028ef9b939e3eba3b7ae30c01e1e5b3633195448b264510cccac951055d6910ccbe0920300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 0; value = 200000
got witness stack of size 4
34 bytes (v0=P2WSH, v1=taproot/tapscript)
Taproot commitment:
- control  = c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255
- program  = 35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37
- script   = 203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c
- path len = 1
- p        = baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183
- q        = 35d2966e0b441e406248f4ed8a69a6283536a85ec03c974687291948265cee37
- k        = f3d10d8d25c08673e65cbba3680bc92a1e66fe15bff701744deeb349361a708a          (tap leaf hash)
  (TapLeaf(0xc0 || 203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c))
valid script
- generating prevout hash from 1 ins
[+] COutPoint(cfc20b54dc, 0)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
<<< taproot commitment >>>                                         |                                                               i: 0
Branch: d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea5... | k: 8a701a3649b3ee4d7401f7bf15fe661e2ac90b68a3bb5ce67386c0258d0d...
Tweak: baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2... | 
CheckTapTweak                                                      | 
<<< committed script >>>                                           | 
3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b   | 
OP_CHECKSIG                                                        | 
46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6   | 
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUAL                                                        | 
#0000 Branch: d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255
```

TODO:explain
```
- looping over path (0..0)
  - 0: node = d1...; taproot control node match -> k first
  (TapBranch(TapLeaf(0xc0 || 203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c) || Span<33,32>=d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255))
  - 0: k -> 5009665d7b3ff8962ab72e096827e9f1c91a5d448811202daabf2dd8850282e5
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
<<< taproot commitment >>>                                         |                                                               i: 1
Branch: d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea5... | k: e5820285d82dbfaa2d201188445d1ac9f1e92768092eb72a96f83f7b5d66...
Tweak: baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2... | 
CheckTapTweak                                                      | 
<<< committed script >>>                                           | 
3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b   | 
OP_CHECKSIG                                                        | 
46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6   | 
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUAL                                                        | 
#0001 Tweak: baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183
```

TODO:explain
```
- looping over path (0..0)
- q.CheckTapTweak(p, k, 0) == success
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b   | 1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f2...
OP_CHECKSIG                                                        | f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6...
46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6   | 
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUAL                                                        | 
#0002 CheckTapTweak
```

TODO:explain
```
		<> PUSH stack 3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        |   3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6   | 1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f2...
OP_CHECKSIGADD                                                     | f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6...
2                                                                  | 
OP_NUMEQUAL                                                        | 
#0003 3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
```

TODO:explain
```
EvalChecksig() sigversion=3
Eval Checksig Tapscript
- sig must not be empty: ok
- validation weight - 50 -> 268
- 32 byte pubkey (new type); schnorr sig check
GenericTransactionSignatureChecker::CheckSchnorrSignature(64 len sig, 32 len pubkey, sigversion=3)
  sig         = 1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa
  pub key     = 3c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1b
SignatureHashSchnorr(in_pos=0, hash_type=00)
- tapscript sighash
- schnorr sighash = 37427dea3b788da55aa2ab6cc28242837b3926bfd2cc033205c92387198cc94e
  pubkey.VerifySchnorrSignature(sig=1e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa, sighash=37427dea3b788da55aa2ab6cc28242837b3926bfd2cc033205c92387198cc94e):
  result: success
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6   |                                                                 01
OP_CHECKSIGADD                                                     | f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6...
2                                                                  | 
OP_NUMEQUAL                                                        | 
#0004 OP_CHECKSIG
```

TODO:explain
```
		<> PUSH stack 46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIGADD                                                     |   46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
2                                                                  |                                                                 01
OP_NUMEQUAL                                                        | f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6...
#0005 46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
```

TODO:explain
```
EvalChecksig() sigversion=3
Eval Checksig Tapscript
- sig must not be empty: ok
- validation weight - 50 -> 218
- 32 byte pubkey (new type); schnorr sig check
GenericTransactionSignatureChecker::CheckSchnorrSignature(64 len sig, 32 len pubkey, sigversion=3)
  sig         = f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3
  pub key     = 46023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6
SignatureHashSchnorr(in_pos=0, hash_type=00)
- tapscript sighash
- schnorr sighash = 37427dea3b788da55aa2ab6cc28242837b3926bfd2cc033205c92387198cc94e
  pubkey.VerifySchnorrSignature(sig=f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3, sighash=37427dea3b788da55aa2ab6cc28242837b3926bfd2cc033205c92387198cc94e):
  result: success
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  |                                                                 02
OP_NUMEQUAL                                                        | 
#0006 OP_CHECKSIGADD
```

TODO:explain
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NUMEQUAL                                                        |                                                                 02
                                                                   |                                                                 02
#0007 2
```

TODO:explain
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 01
#0008 OP_NUMEQUAL
```

Since the script completed without failure and left a non-zero value on top of the stack... this transactioin is valid.

---

## Recovery: Spend Funds

This time, we'll spend our only utxo to 2 outputs and we'll sign with the Recovery Keys C and D. Because this path has an OP_CSV, (See [BIP-68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki) and [BIP-112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) for how this works), we must be careful setting the input's nSequence value to be greater-than-or-equal-to 36 (recall older(36) in our Miniscript descriptor). When it's time to spend -- at least 36 blocks after our utxo's first confirmation, we'll also set our nLockTime to the current blockheight (or if pre-signing in advance, we'll extend the nLockTime appropriately).

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
  'txid': '6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101',
  'vout': 0}]

>>> pprint(txout)
[{'tb1pmkvw09wv8zaeskqf753lxftnp5rc3ezt7f97szccrw7wglcxlqgs9wha2u': Decimal('0.0005')},
 {'tb1qg3lau83hm9e9tdvzr5k7aqtw3uv0dwkfct4xdn': Decimal('0.00018750')}]

# our utxo is already expired according to "older(36)"
>>> unspent['confirmations']
68

# create the initial PSBT
>>> psbt = rpc.createpsbt(txin, txout, rpc.getblockcount())

# process psbt with descriptors for external signing
>>> descriptors = [x['desc'] for x in rpc.listdescriptors()['descriptors']]
>>> unsigned_psbt = rpc.descriptorprocesspsbt(psbt, descriptors)['psbt']

# this base64 psbt needs to be signed by an external signer
>>> print(repr(unsigned_psbt))
'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'

# result of signing by external signer (in this case, krux)
>>> psbtC = "cHNidP8BAH0CAAAAAQFxnXWGnxSRU2EWVxu+Beu9AMbTTI/ZkIo+qiCEAERnAAAAAAAkAAAAAlDDAAAAAAAAIlEg3Zjnlcw4u5hYCfUj8yVzDQeI5EvyS+gLGBu85H8G+BE+SQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJJ5MDAAABAStwEQEAAAAAACJRIEXtEC+ksQV9t8jHNWv4haVn4l1F07EZ0RHcgpo1EcLEQRSHYhX0VKQ8EfQtU/R0Zy/5Q2Svi75QGlzQtRPHanYr8+7KFN2CUMTfS+dzBTwcWvecF+jgpHjVFEt1t/hgedW7QGGgZRfLo0IgJ5taBbs5MsISkCQWNOEpJr8XveKTYadGRDv7JNs9uChRhARyNfvx7QlL1AanWbaiTCg+o85bkVRBFMrWE3zEgozquhSCJ7EtOq6bRAp+Epy3maezJ4jz2S6tvP8HGfSal34kxixDHzV4km00u0FURx+Hj6TYJrJAehxAY6gNnThZc9P24kxwlVG4RpRdSN3M+GlaCePkADyjJHLQ/KXYWdK6jxYN14JL01u++JKxm04EkMtINFAFtX4DmAAAAA=="

>>> psbtD = "cHNidP8BAH0CAAAAAQFxnXWGnxSRU2EWVxu+Beu9AMbTTI/ZkIo+qiCEAERnAAAAAAAkAAAAAlDDAAAAAAAAIlEg3Zjnlcw4u5hYCfUj8yVzDQeI5EvyS+gLGBu85H8G+BE+SQAAAAAAABYAFER/3h432XJVtYIdLe6Bbo8Y9rrJJ5MDAAABAStwEQEAAAAAACJRIEXtEC+ksQV9t8jHNWv4haVn4l1F07EZ0RHcgpo1EcLEQRT4ya8xTnRumbOXuu5/1u49WQQTO1oHiDuKnzbnJe9MPO7KFN2CUMTfS+dzBTwcWvecF+jgpHjVFEt1t/hgedW7QKKKixSOGCqEzRU5zC68zoGKGALM68K9LQFVE7/x84PFyiI8YDz+007YLPWJNluXql6YrGrM5xWSAxcJFu8xobsAAAA="

# combine them
>>> combined_psbt = rpc.combinepsbt([unsigned_psbt, psbtC, psbtD])

# finalize
>>> finalized_psbt = rpc.finalizepsbt(combined_psbt)

# broadcast
>>> if finalized_psbt['complete']:
        rpc.sendrawtransaction(finalized_psbt['hex'])
'c792b4f84e0ee039c0130864b0b3c5aa1195f0b89fe8ddba2c7b529e43cb6d66'
```
[View this transaction on mempool.space](https://mempool.space/signet/tx/c792b4f84e0ee039c0130864b0b3c5aa1195f0b89fe8ddba2c7b529e43cb6d66)

---

## Recovery: Evolution of a PSBT

### psbt = rpc.createpsbt(...
```json
{
  "tx": {
    "txid": "c792b4f84e0ee039c0130864b0b3c5aa1195f0b89fe8ddba2c7b529e43cb6d66",
    "hash": "c792b4f84e0ee039c0130864b0b3c5aa1195f0b89fe8ddba2c7b529e43cb6d66",
    "version": 2,
    "size": 125,
    "vsize": 125,
    "weight": 500,
    "locktime": 234279,
    "vin": [
      {
        "txid": "6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101",
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
          "asm": "1 dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f811",
          "desc": "rawtr(dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f811)#5t6tl8wt",
          "hex": "5120dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f811",
          "address": "tb1pmkvw09wv8zaeskqf753lxftnp5rc3ezt7f97szccrw7wglcxlqgs9wha2u",
          "type": "witness_v1_taproot"
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
          "asm": "1 45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
          "desc": "rawtr(45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4)#tzvrwcm4",
          "hex": "512045ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
          "address": "tb1pghk3qtaykyzhmd7gcu6kh7y954n7yh296wc3n5g3mjpf5dg3ctzq2dqe8y",
          "type": "witness_v1_taproot"
        }
      },
      "taproot_scripts": [
        {
          "script": "2024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadba529c",
          "leaf_ver": 192,
          "control_blocks": [
            "c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49eeeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "script": "20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49ebcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        },
        {
          "pubkey": "876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        },
        {
          "pubkey": "db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "pubkey": "f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        }
      ],
      "taproot_internal_key": "15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e",
      "taproot_merkle_root": "0100b13a7e5544a4e7353dfd63940f64e561d5514ba9d734a101cbd0d6384e09"
```

and `outputs`
```json
      "taproot_internal_key": "b774b571b44645a58bec285c35cc98f84e1aaf5d1a5c6f4603877b49933edb89",
      "taproot_tree": [
        {
          "depth": 1,
          "leaf_ver": 192,
          "script": "20b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8ac20cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832ba529c"
        },
        {
          "depth": 1,
          "leaf_ver": 192,
          "script": "20a21bd2f6942ac04348b138fb88e409af3e535daa3c253f153e08c90c7931cde6ac20ca08a82c56c7399049ed8f5795f939aea99e07817e4c143da82bf4701a972b1aba2030706a9661a7435cb20cdd72ab655687719b822e189461a87947675ce0051495ba529d0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "30706a9661a7435cb20cdd72ab655687719b822e189461a87947675ce0051495",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "cc4af29e74fd0a7694ca18f79b7066bc0fe03f41efe46204235cccad04686eb1"
          ]
        },
        {
          "pubkey": "a21bd2f6942ac04348b138fb88e409af3e535daa3c253f153e08c90c7931cde6",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/2",
          "leaf_hashes": [
            "cc4af29e74fd0a7694ca18f79b7066bc0fe03f41efe46204235cccad04686eb1"
          ]
        },
        {
          "pubkey": "b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "fde5c90b7fed6dec08bcc3c689fb659fa41150833f03b845c9b596d7c3d7543f"
          ]
        },
        {
          "pubkey": "b774b571b44645a58bec285c35cc98f84e1aaf5d1a5c6f4603877b49933edb89",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/2",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "ca08a82c56c7399049ed8f5795f939aea99e07817e4c143da82bf4701a972b1a",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/2",
          "leaf_hashes": [
            "cc4af29e74fd0a7694ca18f79b7066bc0fe03f41efe46204235cccad04686eb1"
          ]
        },
        {
          "pubkey": "cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "fde5c90b7fed6dec08bcc3c689fb659fa41150833f03b845c9b596d7c3d7543f"
          ]
        }
      ]
```

and `fee`
```json
   ,
  "fee": 0.00001250
```

### signed_psbt = ...

strips from `inputs`: `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root`

adds to `inputs`
```json
      "taproot_script_path_sigs": [
        {
          "pubkey": "876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "leaf_hash": "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb",
          "sig": "61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a746443bfb24db3db8285184047235fbf1ed094bd406a759b6a24c283ea3ce5b9154"
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "leaf_hash": "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c",
          "sig": "63a80d9d385973d3f6e24c709551b846945d48ddccf8695a09e3e4003ca32472d0fca5d859d2ba8f160dd7824bd35bbef892b19b4e0490cb48345005b57e0398"
        },
        {
          "pubkey": "f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "leaf_hash": "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb",
          "sig": "a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c5ca223c603cfed34ed82cf589365b97aa5e98ac6acce7159203170916ef31a1bb"
        }
      ]
```

strips from `outputs`: `taproot_internal_key`, `taproot_tree` and `taproot_bip32_derivs`

### combined_psbt = rpc.combinepsbt(...

replaces stripped fields from `inputs`
```json
      ],
      "taproot_scripts": [
        {
          "script": "2024a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0ac20cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92eadba529c",
          "leaf_ver": 192,
          "control_blocks": [
            "c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49eeeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "script": "20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2",
          "leaf_ver": 192,
          "control_blocks": [
            "c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49ebcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/1",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "24a7e480899a14ea0140667eee1c395c5d21051b547a138dafab54cc37f6b4d0",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        },
        {
          "pubkey": "876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "pubkey": "cad6137cc4828ceaba148227b12d3aae9b440a7e129cb799a7b32788f3d92ead",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
          ]
        },
        {
          "pubkey": "db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        },
        {
          "pubkey": "f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/1",
          "leaf_hashes": [
            "eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079d5bb"
          ]
        }
      ],
      "taproot_internal_key": "15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e",
      "taproot_merkle_root": "0100b13a7e5544a4e7353dfd63940f64e561d5514ba9d734a101cbd0d6384e09"
```

...and `outputs`

```json
      "taproot_internal_key": "b774b571b44645a58bec285c35cc98f84e1aaf5d1a5c6f4603877b49933edb89",
      "taproot_tree": [
        {
          "depth": 1,
          "leaf_ver": 192,
          "script": "20b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8ac20cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832ba529c"
        },
        {
          "depth": 1,
          "leaf_ver": 192,
          "script": "20a21bd2f6942ac04348b138fb88e409af3e535daa3c253f153e08c90c7931cde6ac20ca08a82c56c7399049ed8f5795f939aea99e07817e4c143da82bf4701a972b1aba2030706a9661a7435cb20cdd72ab655687719b822e189461a87947675ce0051495ba529d0124b2"
        }
      ],
      "taproot_bip32_derivs": [
        {
          "pubkey": "30706a9661a7435cb20cdd72ab655687719b822e189461a87947675ce0051495",
          "master_fingerprint": "cdef7cd9",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "cc4af29e74fd0a7694ca18f79b7066bc0fe03f41efe46204235cccad04686eb1"
          ]
        },
        {
          "pubkey": "a21bd2f6942ac04348b138fb88e409af3e535daa3c253f153e08c90c7931cde6",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/2/2",
          "leaf_hashes": [
            "cc4af29e74fd0a7694ca18f79b7066bc0fe03f41efe46204235cccad04686eb1"
          ]
        },
        {
          "pubkey": "b238d8990dc4eb73295d7f0790091ad3715b3e9a0e6edf0da1980a0222343bb8",
          "master_fingerprint": "07fd816d",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "fde5c90b7fed6dec08bcc3c689fb659fa41150833f03b845c9b596d7c3d7543f"
          ]
        },
        {
          "pubkey": "b774b571b44645a58bec285c35cc98f84e1aaf5d1a5c6f4603877b49933edb89",
          "master_fingerprint": "7c461e5d",
          "path": "m/0/2",
          "leaf_hashes": [
          ]
        },
        {
          "pubkey": "ca08a82c56c7399049ed8f5795f939aea99e07817e4c143da82bf4701a972b1a",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/2/2",
          "leaf_hashes": [
            "cc4af29e74fd0a7694ca18f79b7066bc0fe03f41efe46204235cccad04686eb1"
          ]
        },
        {
          "pubkey": "cc77a266335d986a8c7825ed646f1f26ae1b66ab49ad15ea27b76b1c98b62832",
          "master_fingerprint": "da855a1f",
          "path": "m/48h/1h/0h/2h/0/2",
          "leaf_hashes": [
            "fde5c90b7fed6dec08bcc3c689fb659fa41150833f03b845c9b596d7c3d7543f"
          ]
        }
      ]
```

### finalized_psbt = rpc.finalizepsbt(...

strips from `inputs`: `taproot_script_path_sigs`, `taproot_scripts`, `taproot_bip32_derivs`, `taproot_internal_key` and `taproot_merkle_root`

adds to `inputs`
```json
      "final_scriptwitness": [
        "a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c5ca223c603cfed34ed82cf589365b97aa5e98ac6acce7159203170916ef31a1bb",
        "61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a746443bfb24db3db8285184047235fbf1ed094bd406a759b6a24c283ea3ce5b9154",
        "",
        "20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2",
        "c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49ebcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
      ]
```

Final transaction

```json
{
  "txid": "c792b4f84e0ee039c0130864b0b3c5aa1195f0b89fe8ddba2c7b529e43cb6d66",
  "hash": "0d83d6e2bf30c0dce9cb5106dc0434c42fb20f1131b5f1c402828667c3c04ca5",
  "version": 2,
  "size": 433,
  "vsize": 202,
  "weight": 808,
  "locktime": 234279,
  "vin": [
    {
      "txid": "6744008420aa3e8a90d98f4cd3c600bdeb05be1b5716615391149f86759d7101",
      "vout": 0,
      "scriptSig": {
        "asm": "",
        "hex": ""
      },
      "txinwitness": [
        "a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c5ca223c603cfed34ed82cf589365b97aa5e98ac6acce7159203170916ef31a1bb",
        "61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a746443bfb24db3db8285184047235fbf1ed094bd406a759b6a24c283ea3ce5b9154",
        "",
        "20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2",
        "c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49ebcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c"
      ],
      "sequence": 36
    }
  ],
  "vout": [
    {
      "value": 0.00050000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f811",
        "desc": "rawtr(dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f811)#5t6tl8wt",
        "hex": "5120dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f811",
        "address": "tb1pmkvw09wv8zaeskqf753lxftnp5rc3ezt7f97szccrw7wglcxlqgs9wha2u",
        "type": "witness_v1_taproot"
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
```
    {
      "value": 0.00070000,
      "n": 0,
      "scriptPubKey": {
        "asm": "1 45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
        "desc": "rawtr(45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4)#tzvrwcm4",
        "hex": "512045ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4",
        "address": "tb1pghk3qtaykyzhmd7gcu6kh7y954n7yh296wc3n5g3mjpf5dg3ctzq2dqe8y",
        "type": "witness_v1_taproot"
      }
    },
```

Starting btcdeb
```
btcdeb --quiet --tx=0200000000010101719d75869f1491536116571bbe05ebbd00c6d34c8fd9908a3eaa20840044670000000000240000000250c3000000000000225120dd98e795cc38bb985809f523f325730d0788e44bf24be80b181bbce47f06f8113e49000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90540a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c5ca223c603cfed34ed82cf589365b97aa5e98ac6acce7159203170916ef31a1bb4061a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a746443bfb24db3db8285184047235fbf1ed094bd406a759b6a24c283ea3ce5b9154006b20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b241c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49ebcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c27930300 --txin=0200000000010119f605e6cd37dd5d780e15c71d57d4e7c23dea56b64b9378b311addc540bc2cf0000000000fdffffff03701101000000000022512045ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4f824010000000000160014c675e0b4b427a33c62147702f6f315497b33fd6cc0d0000000000000160014447fde1e37d97255b5821d2dee816e8f18f6bac90440f0eccd95efadd740fbe18f93a4441b72d9100022ec364e17a23a3ee7ff0a3b6f0c9954da42a58d0834802974e291cfafe3182f18ec25cafac307ec813d2e5ed3401e12c03bf63d41f87a2fd792b77faf816b65f720483d227898f8f41efe696f22a38baf9f24e4bcf210dbbdaefd53d8a2a63bda5600388545177380de988b42aa46203c343e06381b7e7926fdbe54c63ab0040c5c2c8491e2d4bd7238cb9066040a1bac2046023b7872e0027577d8a451389bc5a430e729f2d389a23ecca1237dc3b7f7d6ba529c41c0baf872c3fb5bdb356615e7045ed8306d56e0df10c31fd2f0143192d2dd812183d1742e6161782799d44467ab0dc374b1cb88bc2f719e44bc49f9ea545c9ed255e3920300

LOG: signing segwit taproot
notice: btcdeb has gotten quieter; use --verbose if necessary (this message is temporary)
input tx index = 0; tx input vout = 0; value = 70000
got witness stack of size 5
34 bytes (v0=P2WSH, v1=taproot/tapscript)
Taproot commitment:
- control  = c015649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49ebcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c
- program  = 45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4
- script   = 20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2
- path len = 1
- p        = 15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e
- q        = 45ed102fa4b1057db7c8c7356bf885a567e25d45d3b119d111dc829a3511c2c4
- k        = bbd57960f8b7754b14d578a4e0e8179cf75a1c3c0573e74bdfc45082dd14caee          (tap leaf hash)
  (TapLeaf(0xc0 || 20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2))
valid script
- generating prevout hash from 1 ins
[+] COutPoint(6744008420, 0)
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
<<< taproot commitment >>>                                         |                                                               i: 0
Branch: bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d82... | k: eeca14dd8250c4df4be773053c1c5af79c17e8e0a478d5144b75b7f86079...
Tweak: 15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5a... | 
CheckTapTweak                                                      | 
<<< committed script >>>                                           | 
db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855   | 
OP_CHECKSIG                                                        | 
876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3   | 
OP_CHECKSIGADD                                                     | 
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   | 
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0000 Branch: bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c
```

TODO:explain
```
- looping over path (0..0)
  - 0: node = bc...; taproot control node mismatch -> k second
  (TapBranch(Span<33,32>=bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d826b2407a1c || TapLeaf(0xc0 || 20db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855ac20876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3ba20f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3cba529d0124b2)))
  - 0: k -> 094e38d6d0cb01a134d7a94b51d561e5640f9463fd3d35e7a444557e3ab10001
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
<<< taproot commitment >>>                                         |                                                               i: 1
Branch: bcff0719f49a977e24c62c431f3578926d34bb4154471f878fa4d82... | k: 0100b13a7e5544a4e7353dfd63940f64e561d5514ba9d734a101cbd0d638...
Tweak: 15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5a... | 
CheckTapTweak                                                      | 
<<< committed script >>>                                           | 
db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855   | 
OP_CHECKSIG                                                        | 
876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3   | 
OP_CHECKSIGADD                                                     | 
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   | 
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0001 Tweak: 15649776626f279b6a3604a438ec29d8a7935ee49516ddf536313f5acb98a49e
```

TODO:explain
```
- looping over path (0..0)
- q.CheckTapTweak(p, k, 0) == success
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855   |                                                                 0x
OP_CHECKSIG                                                        | 61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a74...
876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3   | a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c...
OP_CHECKSIGADD                                                     | 
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   | 
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0002 CheckTapTweak
```

TODO:explain
```
		<> PUSH stack db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIG                                                        |   db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3   |                                                                 0x
OP_CHECKSIGADD                                                     | 61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a74...
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   | a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c...
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0003 db875e9d054700cb0306f8e6b379965d2879e05a79d13e209b3957711c5f7855
```

TODO:explain
```
EvalChecksig() sigversion=3
Eval Checksig Tapscript
- sig must not be empty: it is empty
- 32 byte pubkey (new type); schnorr sig check
		<> POP  stack
		<> POP  stack
		<> PUSH stack 
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3   |                                                                 0x
OP_CHECKSIGADD                                                     | 61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a74...
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   | a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c...
OP_CHECKSIGADD                                                     | 
2                                                                  | 
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0004 OP_CHECKSIG
```

TODO:explain
```
		<> PUSH stack 876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIGADD                                                     |   876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   |                                                                 0x
OP_CHECKSIGADD                                                     | 61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a74...
2                                                                  | a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c...
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0005 876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
```

TODO:explain
```
EvalChecksig() sigversion=3
Eval Checksig Tapscript
- sig must not be empty: ok
- validation weight - 50 -> 306
- 32 byte pubkey (new type); schnorr sig check
GenericTransactionSignatureChecker::CheckSchnorrSignature(64 len sig, 32 len pubkey, sigversion=3)
  sig         = 61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a746443bfb24db3db8285184047235fbf1ed094bd406a759b6a24c283ea3ce5b9154
  pub key     = 876215f454a43c11f42d53f474672ff94364af8bbe501a5cd0b513c76a762bf3
SignatureHashSchnorr(in_pos=0, hash_type=00)
- tapscript sighash
- schnorr sighash = 0de5dd80832fed58c639fcacc59de08e6642d3f77b5146450f9d5a68476c91bd
  pubkey.VerifySchnorrSignature(sig=61a06517cba34220279b5a05bb3932c21290241634e12926bf17bde29361a746443bfb24db3db8285184047235fbf1ed094bd406a759b6a24c283ea3ce5b9154, sighash=0de5dd80832fed58c639fcacc59de08e6642d3f77b5146450f9d5a68476c91bd):
  result: success
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c   |                                                                 01
OP_CHECKSIGADD                                                     | a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c...
2                                                                  | 
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0006 OP_CHECKSIGADD
```

TODO:explain
```
		<> PUSH stack f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSIGADD                                                     |   f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
2                                                                  |                                                                 01
OP_NUMEQUALVERIFY                                                  | a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c...
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0007 f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
```

TODO:explain
```
EvalChecksig() sigversion=3
Eval Checksig Tapscript
- sig must not be empty: ok
- validation weight - 50 -> 256
- 32 byte pubkey (new type); schnorr sig check
GenericTransactionSignatureChecker::CheckSchnorrSignature(64 len sig, 32 len pubkey, sigversion=3)
  sig         = a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c5ca223c603cfed34ed82cf589365b97aa5e98ac6acce7159203170916ef31a1bb
  pub key     = f8c9af314e746e99b397baee7fd6ee3d5904133b5a07883b8a9f36e725ef4c3c
SignatureHashSchnorr(in_pos=0, hash_type=00)
- tapscript sighash
- schnorr sighash = 0de5dd80832fed58c639fcacc59de08e6642d3f77b5146450f9d5a68476c91bd
  pubkey.VerifySchnorrSignature(sig=a28a8b148e182a84cd1539cc2ebcce818a1802ccebc2bd2d015513bff1f383c5ca223c603cfed34ed82cf589365b97aa5e98ac6acce7159203170916ef31a1bb, sighash=0de5dd80832fed58c639fcacc59de08e6642d3f77b5146450f9d5a68476c91bd):
  result: success
		<> POP  stack
		<> POP  stack
		<> POP  stack
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
2                                                                  |                                                                 02
OP_NUMEQUALVERIFY                                                  | 
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0008 OP_CHECKSIGADD
```

TODO:explain
```
		<> PUSH stack 02
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_NUMEQUALVERIFY                                                  |                                                                 02
24                                                                 |                                                                 02
OP_CHECKSEQUENCEVERIFY                                             | 
#0009 2
```

TODO:explain
```
		<> POP  stack
		<> POP  stack
		<> PUSH stack 01
		<> POP  stack
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
24                                                                 | 
OP_CHECKSEQUENCEVERIFY                                             | 
#0010 OP_NUMEQUALVERIFY
```

TODO:explain
```
		<> PUSH stack 24
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
OP_CHECKSEQUENCEVERIFY                                             |                                                                 24
#0011 24
```

TODO:explain
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 24
#0012 OP_CHECKSEQUENCEVERIFY
```

TODO:explain
```
script                                                             |                                                             stack 
-------------------------------------------------------------------+-------------------------------------------------------------------
                                                                   |                                                                 24
```

Since the script completed without failure and left a non-zero value on top of the stack... this transaction is valid.
