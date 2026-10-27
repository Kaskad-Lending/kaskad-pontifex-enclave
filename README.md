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

| Image | Dockerfile | Pinned in | Registered signer |
|---|---|---|---|
| Bridge | `Dockerfile.pontifex` | `KskdEntry` `0x7b1D080d2485E63791468d73418e2B262c25C402` | `0x7d3B23a631501a35cafe4F00133653A1b90f2CC3` |
| Oracle | `Dockerfile.oracle` | `KaskadPriceOracleV2` `0xB4922b744cc395E05Cc962d55FA63FBab0599ec2` | `0xe4d88be9756eb3fc5bd28b81642605fbf238d2c9` |

```text
bridge  PCR0 8761b33a338db4fab12c9201eef87e1c582310d59645848c2574061d1e070d71238976ea1af05ad4b4e645004621aac6
        PCR1 4b4d5b3661b3efc12920900c80e126e4ce783c522de6c02a2a5bf7af3a2b9327b86776f188e4be1c1c404a129dbda493
        PCR2 053d5d61b98e960c777ea280b7b9550951d3b105689f23bb63d25a2afaf0ff3823ad240aff39f34437289e2761a4a2c1
oracle  PCR0 5248a0d0ca1bea3044a270078286a7f788dfba98ada7069bdc4db81407e6a3a65c5be8f76e3cd299571621d71cc02296
        PCR1 4b4d5b3661b3efc12920900c80e126e4ce783c522de6c02a2a5bf7af3a2b9327b86776f188e4be1c1c404a129dbda493
        PCR2 cb935649ecdfcff7944cd891ae8de97b91c1fd252faa38547c68eb8a447a0acee3ef3244309d38673df49cefce5c4a46
```

Checked on 2026-10-15:

- Rebuilding both images from this source without Nitro hardware gives these PCR0, PCR1 and PCR2 values.
- They equal the PCRs in the live attestation documents of both enclaves. Each document's signature chains to the AWS Nitro root certificate.
- The first 32 bytes of each PCR0 equal `expectedPcr0()` of the contract that pins it.
- Both signers are registered on chain (block 89212749): `KskdEntry.validSigner` and `KaskadPriceOracleV2.isValidSigner` return true.

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

This tree is an export of [Kaskad-Lending/kaskad-nuntius@0f3865c](https://github.com/Kaskad-Lending/kaskad-nuntius/tree/0f3865c834495938b14bb7a951e5998331cac072). Every exported file has the same git object id as in that commit.

- The live bridge image was built from that commit.
- The live oracle image was built from its ancestor `6df59ad`. Every file `Dockerfile.oracle` reads, and every `KEYEX_ORACLE_*` value, is identical in both commits.

Two places use the earlier ids of these commits: the commit message of `79c81b8` names `0f3865c` as `3944e30`, and `host/eif-config/robinhood-mainnet.env` names `6df59ad` as `dcfe989`.

## License

Apache-2.0. See `LICENSE` and `NOTICE`.
