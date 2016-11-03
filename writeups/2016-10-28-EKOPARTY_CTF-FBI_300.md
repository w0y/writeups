Title: EKOPARTY CTF 2016 - FBI 300
Category: writeup
Tags: ctf, writeup
Authors: kernoelpanic
CTF-CTFTIME-EVENT: 
CTF: EKOPARTYCTF
CTF-YEAR: 2016
CTF-CATEGORY: FBI
CTF-POINTS: 300
CTF-CHAL-NAME: BTC 300
Summary: Bitcoin as OP_RETURN Dropbox  


The description of the challenge was as follows:
```
There has been some strange transactions on this blockchain! Let's do some research.
```

After downloading and extracting the data ([fbi300_64635d9aa64b20d0.7z]({filename}/data/ctf/ekoparty2016/fbi300_64635d9aa64b20d0.7z)) is was clear that we where looking at at
a `.bitcoin` folder of a *bitcoin-core* client hat was started in *regtest* mode. 

As a first guess we used **bitcoin-abe** to read and analyze the blockchain.
(https://github.com/bitcoin-abe/bitcoin-abe).
Since *bitcoin-abe* looks out-of-the-box in the default bitcoin directory (`$HOME/.bitcoin/blcoks/*`) the only thing we had to take care about was that the regtest files can be found there.
```
$ python -m Abe.abe --config abe-sqlite.conf --commit-bytes 100000 --no-serve 
... 
$ python -m Abe.abe --config abe-sqlite.conf
```

We quickly found that the only blocks in the blockchain that include transactions (despite coinbase transactions) contain batches of `25` tx per block.

By looking at those transactions we quickly saw that the last output always includes a 
`OP_RETURN` together with some bytes appended to it. 

After looking at the first transaction in block `102` where this whole pattern started
it was clear that there was some *gzip* file hidden after the `OP_RETURN` opcodes:
```
>>> import binascii
>>> binascii.unhexlify("1f8b080850cb10580400626974636f696e2e706466008c9a636c2e5012866bdbb66ddfdab6dd7ebdb5eddb766bdbb66ddbb66ddb5d66b3c9ee8ffd33994ce6cc9b777292933c39e48aa2e2f4cc0c6c30")
'\x1f\x8b\x08\x08P\xcb\x10X\x04\x00bitcoin.pdf\x00\x8c\x9acl.P\x12\x86k\xdb\xb6m\xdf\xda\xb6\xdd~\xbd\xb5\xed\xdbvk\xdb\xb6m\xdb\xb6m\xdb]f\xb3\xc9\xee\x8f\xfd3\x99L\xe6\xcc\x9bwr\x92\x93<9\xe4\x8a\xa2\xe2\xf4\xcc\x0cl0'
```

We used the *sqlite3* database of *bitcoin-abe* directly to read out all the transaction outputs containing the `OP_RETURN` (`0x6a`) values so that we have them also in the right order in which they appear in the blockchain. 
```
sqlite> select block_height,tx_pos,quote(txout_scriptPubKey) from txout_detail where quote(txout_scriptPubKey) like "X'6a%" ORDER BY block_height ASC, tx_pos ASC;
102|1|X'6A4C501F8B080850CB10580400626974636F696E2E706466008C9A636C2E5012866BDBB66DDFDAB6DD7EBDB5EDDB766BDBB66DDBB66DDB5D66B3C9EE8FFD33994CE6CC9B777292933C39E48AA2E2F4CC0C6C30'
102|2|X'6A4C50E40385039D03CD0339302C444C4476C696307C7C8CB200DBDFCEE644AC7FAB28338A5B583B031C19C5AD8D9C01A200133B53003F3F8C93B323C0C806C63DB34A572682AC0DB97BA6A6B4216F9D9A9A1C'
...
```

After concatenating all values (ignoring size parameters) we saw that the results *gzip* archive was corrupted.
It took as while to figure out that this was because there where some transactions missing, but those transaction have not been in the blockchain yet. 
The missing transactions where `0` confirmation transactions. 

We fired up *bitcoin-core* and extracted those. 
```
$ ./bitcoin-cli -regtest listtransactions | grep -REin -A 2 'confirmations\": 0' | grep -En "txid" | cut -d"\"" -f4 | uniq
59df67eeb96d74bb492a1ae510bf992b155993a4e1fba9b50be18584516c976c
d4d01e49f92af854067ae11f118f72db8e4d74baea3b11d6ba4c7179eab29625
e4f56398207e14bf13f764d56655e5063d2150fcf154bb9e841583f50d9bd471
c46a2e6ff9329bc3e61e6eaab5fff877c3dde4609826e1d03bd3110c777c543a
0c970f180f32959a1cad6ec5c5abb15a706cce5b13d134214781a28826947ca9
ccfff7a9c493878dd10bcb93a41623b58b1cc84a00c677a4201862d95cbc83bf
2b02bcb8623428c8fb8ca827a90759bbbbd207f4835a482ef74347d4cd9bdf4f
d070e9596f2e90f9de3fdea77176d057ea3a1ac5ba7701e5f9fa917985097fd2
47ae220f6ce6d56d4a2b818c4f2d77085f4c869bb8747bdf3abac01ad5c83e2f

$ ./bitcoin-cli -regtest getrawtransaction 47ae220f6ce6d56d4a2b818c4f2d77085f4c869bb8747bdf3abac01ad5c83e2f 1
{
  "hex": "0100000001d27f09857991faf9e50177bac51a3aea57d07671a7de3fdef9902e6f59e970d0010000006b483045022100bfd4a7d44b40aeba6fd50a436539fe8e2ca57a3af2035952c59c97b7693e41e5022025263cc5741093872cae55efa3f7565dfa45d091ea99074dc9b183d126ff4591012103bcc584628ff0eee4f3d2526abb809fc26f821c0c5a7330624e38c29fa57db3c4ffffffff0320a10700000000001976a9146dbd4f1fe23f2f4e971121b667396facc5948ac588ac009c5ee6000000001976a9146042bda27b7fe3b9e1d6471b0c3f4beaa251f34388ac00000000000000001b6a197effdba3318ef6d6060e18174bb4c79f01aa1123dbfecf020000000000",
  "txid": "47ae220f6ce6d56d4a2b818c4f2d77085f4c869bb8747bdf3abac01ad5c83e2f",
  "hash": "47ae220f6ce6d56d4a2b818c4f2d77085f4c869bb8747bdf3abac01ad5c83e2f",
  "size": 262,
  "vsize": 262,
  "version": 1,
  "locktime": 0,
  "vin": [
    {
      "txid": "d070e9596f2e90f9de3fdea77176d057ea3a1ac5ba7701e5f9fa917985097fd2",
      "vout": 1,
      "scriptSig": {
        "asm": "3045022100bfd4a7d44b40aeba6fd50a436539fe8e2ca57a3af2035952c59c97b7693e41e5022025263cc5741093872cae55efa3f7565dfa45d091ea99074dc9b183d126ff4591[ALL] 03bcc584628ff0eee4f3d2526abb809fc26f821c0c5a7330624e38c29fa57db3c4",
        "hex": "483045022100bfd4a7d44b40aeba6fd50a436539fe8e2ca57a3af2035952c59c97b7693e41e5022025263cc5741093872cae55efa3f7565dfa45d091ea99074dc9b183d126ff4591012103bcc584628ff0eee4f3d2526abb809fc26f821c0c5a7330624e38c29fa57db3c4"
      },
      "sequence": 4294967295
    }
  ],
  "vout": [
    {
      "value": 0.00500000,
      "n": 0,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 6dbd4f1fe23f2f4e971121b667396facc5948ac5 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a9146dbd4f1fe23f2f4e971121b667396facc5948ac588ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "mqXCjPWwiSyy4XYVzGQrKtLSPmKEYegJFT"
        ]
      }
    }, 
    {
      "value": 38.64960000,
      "n": 1,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 6042bda27b7fe3b9e1d6471b0c3f4beaa251f343 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a9146042bda27b7fe3b9e1d6471b0c3f4beaa251f34388ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "mpHw7hVxSDsWDtBcXHoBaK4f5cnqf95xN9"
        ]
      }
    }, 
    {
      "value": 0.00000000,
      "n": 2,
      "scriptPubKey": {
        "asm": "OP_RETURN 7effdba3318ef6d6060e18174bb4c79f01aa1123dbfecf0200",
        "hex": "6a197effdba3318ef6d6060e18174bb4c79f01aa1123dbfecf0200",
        "type": "nulldata"
      }
    }
  ]
}
```

After appending those transactions the *gzip* file decompressed successfully and showed the original 
Bitcoin paper. 

At the end of the `Bitcoin.pdf` there was the flag appended:
```
$ xxd Bitcoin.pdf | tail -n3 
0002cfd0: 7274 7872 6566 0a31 3832 3732 370a 2525  rtxref.182727.%%
0002cfe0: 454f 460a 454b 4f7b 4f50 5f4f 505f 4f50  EOF.EKO{OP_OP_OP
0002cff0: 5f72 6574 7572 6e5f 7374 796c 657d       _return_style}
```
