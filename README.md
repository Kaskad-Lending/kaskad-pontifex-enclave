# Kaskad Pontifex: bridge enclave

Source of the AWS Nitro Enclave images behind the KSKD bridge from Igra (chain 38833) to Robinhood Chain (chain 4663): the bridge enclave that signs claims, and the Kaskad oracle enclave that derives the bridge's signing key.

Contracts: [Kaskad-Lending/kaskad-pontifex](https://github.com/Kaskad-Lending/kaskad-pontifex)

## How a claim works

1. A user burns KSKD through `KskdExit` on Igra, which adds the amount to the cumulative `burned[recipient]`.
2. The bridge enclave reads `burned[recipient]` from a finalized Igra block at least 800 blocks behind the head. It checks the recipient and the amount, then signs an EIP-712 claim over (recipient, cumulative burn, deadline) for `KskdEntry` on Robinhood Chain. Within one enclave run, the total it signs for a recipient never decreases.
3. `KskdEntry` mints the difference between the signed cumulative burn and the amount already bridged to the recipient, once the claim carries `threshold` signatures of registered enclave signers. A daily mint limit caps the amount.

## Where the signing key comes from

- The bridge enclave never generates a key. At boot it fetches its key over RA-TLS and accepts it only from an oracle image whose PCR0 is in the baked `PONTIFEX_ANCESTOR_PCRS` list.
- The oracle derives the bridge key from its root key: `HKDF-SHA256(ikm = K_root, salt = "kaskad-keyex-v1", info = "kaskad/pontifex/" || label)`. The label comes from the approval; the bridge's label is `keccak256("kaskad/pontifex/v1")`.
- The oracle hands a key to an image with a different PCR0 only under an owner approval for that PCR0: CHILD for a derived key, CARRY with a strictly higher version for the root key itself. An oracle enclave running the same image receives the root key without an approval.
- An approval counts only when at least 3 of the 5 Safe owners baked into the oracle image (`KEYEX_ORACLE_OWNERS`, `KEYEX_ORACLE_THRESHOLD`) sign it. Approvals are EIP-712 typed data under the domain `Kaskad Keyex`, version 1, chain 4663.
- The oracle also reads the governing Safe `0x313807B857222278FbddF36541DE2Af5a50BdCB5` on Robinhood Chain. That read can only remove owners from the baked set or raise the threshold.
- `KskdEntry.registerEnclave`, callable by its owner (the same Safe), admits a signer only from an AWS Nitro attestation whose PCR0 matches `KskdEntry.expectedPcr0()`. The pin holds the first 32 of the 48 PCR0 bytes.

## Layout

| Path | Content |
|---|---|
| `crates/pontifex-bridge` | bridge enclave binary |
| `crates/keyex` | key handover shared by both images: EIP-712 approvals, RA-TLS, boot rules, key derivation |
| `crates/nitro-common` | NSM access, attestation parsing and verification |
| `src/`, `proto/`, `config/`, `build.rs` | Kaskad price oracle, the bridge's key parent |
| `Dockerfile.pontifex`, `Dockerfile.oracle` | the two enclave images |
| `host/eif-config/robinhood-mainnet.env` | build arguments baked into both images: contract addresses, RPC URLs, Safe owners |
| `testdata/`, `tests/` | attestation fixtures and tests |

All crates share one Cargo workspace and one `Cargo.lock`.

## Live images on Robinhood Chain (4663)

| Image | Dockerfile | Contract | Registered signer |
|---|---|---|---|
| Bridge | `Dockerfile.pontifex` | `KskdEntry` `0x7b1D080d2485E63791468d73418e2B262c25C402` | `0x7d3B23a631501a35cafe4F00133653A1b90f2CC3` |
| Oracle | `Dockerfile.oracle` | `KaskadPriceOracleV2` `0xB4922b744cc395E05Cc962d55FA63FBab0599ec2` | `0xe4d88be9756eb3fc5bd28b81642605fbf238d2c9` |

```text
bridge  PCR0 9edc0e6d71d3e797f407eb141532d0973f34ec566bf49bae0a61b975fa4b1fb7041b7e84fdb995d441703c2f70f2bc67
        PCR1 4b4d5b3661b3efc12920900c80e126e4ce783c522de6c02a2a5bf7af3a2b9327b86776f188e4be1c1c404a129dbda493
        PCR2 7d317a409c3c5333326bef388eab2fdc3b4d9bb1e908182891c52a8230565a12fcc9e41bab63d167694fea6143447679
oracle  PCR0 d7b46b6b59833df30f2e72fc9bbb465dcb3f9d6365eccbafe0c5557975f4b8e0533f46abe9bd6809b3c7995965044d7d
        PCR1 4b4d5b3661b3efc12920900c80e126e4ce783c522de6c02a2a5bf7af3a2b9327b86776f188e4be1c1c404a129dbda493
        PCR2 0c6260d9e09e6252fd0ec33b83e32deda3f37926689622cb651bbf9138a199016b671f215aaee9056f2f6b6b0c858805
```

These are the PCRs measured by the release build of each image on 2026-11-05.

Checked on 2026-11-05 (block 107034121):

- Both signers are registered on chain: `KskdEntry.validSigner` and `KaskadPriceOracleV2.isValidSigner` return true.
- The PCR0 in the live bridge attestation documents equals the bridge PCR0 above.
- `KaskadPriceOracleV2.expectedPcr0()` holds the first 32 bytes of the oracle PCR0 above. `KskdEntry.expectedPcr0()` still holds the first 32 bytes of the image version 1 bridge PCR0.
- Image version 2 keeps the version 1 keys: the oracle receives its root key under a CARRY approval, and the bridge its derived key under a CHILD approval for its new PCR0. The signers stay the same.

The checks of image version 1 are in the README at tag `robinhood-mainnet-v1`.

## Build

Requirements:

- x86_64 Linux with Docker. No Nitro hardware and no AWS account: `nitro-cli build-enclave` only needs Docker.
- The default builder uses the `docker` driver: `docker buildx inspect default` shows `Driver: docker`.
- The classic image store: `docker info --format '{{.Driver}}'` prints `overlay2`.

The containerd image store and the BuildKit `docker-container` driver build the same binary but give a different PCR2, and therefore a different PCR0.

Build the bridge image:

```sh
( set -a; . host/eif-config/robinhood-mainnet.env; set +a
  docker build --builder default --no-cache -f Dockerfile.pontifex -t pontifex-bridge \
    --build-arg SOURCE_DATE_EPOCH=1700000000 \
    --build-arg PONTIFEX_EXIT --build-arg PONTIFEX_KSKD --build-arg PONTIFEX_ENTRY \
    --build-arg PONTIFEX_CHAIN_ID --build-arg PONTIFEX_IGRA_RPCS \
    --build-arg PONTIFEX_VERSION --build-arg PONTIFEX_ANCESTOR_PCRS . )
```

Measure it with nitro-cli 1.5.0 from the Amazon Linux 2023 repository. The printed JSON carries PCR0, PCR1 and PCR2:

```sh
docker run --rm --security-opt label=disable \
  -v /var/run/docker.sock:/var/run/docker.sock -v "$PWD/out:/out" \
  -e NITRO_CLI_ARTIFACTS=/tmp/nca public.ecr.aws/amazonlinux/amazonlinux:2023 bash -c '
    set -e
    dnf install -y aws-nitro-enclaves-cli-1.5.0-0.amzn2023 aws-nitro-enclaves-cli-devel-1.5.0-0.amzn2023 >/dev/null
    mkdir -p /var/log/nitro_enclaves /tmp/nca
    nitro-cli build-enclave --docker-uri pontifex-bridge --output-file /out/pontifex.eif'
```

- Compare PCRs, not the EIF file hash. The EIF records its build time outside the measured sections, so its file hash changes on every build.
- nitro-cli 1.3.4 through 1.5.0 ship identical kernel and init blobs and give the same PCRs.

The oracle image builds the same way; measure it with `--docker-uri oracle-enclave --output-file /out/oracle.eif`:

```sh
( set -a; . host/eif-config/robinhood-mainnet.env; set +a
  docker build --builder default --no-cache -f Dockerfile.oracle -t oracle-enclave \
    --build-arg SOURCE_DATE_EPOCH=1700000000 \
    --build-arg KEYEX_ORACLE_REGISTRY --build-arg KEYEX_ORACLE_RH_RPCS --build-arg KEYEX_ORACLE_SAFE \
    --build-arg KEYEX_ORACLE_OWNERS --build-arg KEYEX_ORACLE_THRESHOLD --build-arg KEYEX_ORACLE_CHAIN_ID \
    --build-arg KEYEX_ORACLE_VERSION --build-arg KEYEX_ORACLE_ANCESTORS --build-arg KEYEX_ORACLE_PEERS . )
```

## Source

This tree is an export of [Kaskad-Lending/kaskad-nuntius@005b71b](https://github.com/Kaskad-Lending/kaskad-nuntius/tree/005b71b6cd22c5b4163119d3128536502c6261f1). Every exported file has the same git object id as in that commit.

- The live bridge image was built from that commit.
- The live oracle image was built from its parent `7e6b832`. The two commits differ only in the `PONTIFEX_ANCESTOR_PCRS` line of `host/eif-config/robinhood-mainnet.env` and its comment. Every file `Dockerfile.oracle` reads, and every `KEYEX_ORACLE_*` value, is identical in both commits.

## License

Apache-2.0. See `LICENSE` and `NOTICE`.
