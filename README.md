# sc_fw_decrypt

Decrypt and verify `sc_fw_update` component images (**BLNK** / **BASE** / **SYST**).

Each component file is a nested container:

```
whole file  = AES-128-CBC( key, IV = 0 )           # encrypted
plaintext   = [ 16-byte AES-128-CMAC ][ payload ]  # after CBC decrypt
CMAC        = AES-128-CMAC( cmac_key ) over payload # == plaintext[0:16]
payload     = real image: 4-byte magic + version, 0xFF-padded
```

| File       | Magic  |
|------------|--------|
| `40000001` | `BLNK` |
| `40000002` | `BASE` |
| `40000003` | `SYST` |

## Keys

Both keys are the published defaults and are baked into the tool
(override with `--key` / `--cmac-key`):

```
AES-128-CBC key : 5301C28824B57137A819C042FC119E3F   (IV = all zeros)
AES-128-CMAC key: 8F215691AC7EF6510239DD32CC6A2394
```

## Requirements

- Python 3.6+
- [pycryptodome](https://www.pycryptodome.org/)

```sh
python3 -m pip install --user pycryptodome
# or, on Debian/Ubuntu:
sudo apt install python3-pycryptodome
```

## Install

```sh
# per-user (no root; ~/.local/bin is usually on PATH)
install -m 0755 sc_fw_decrypt ~/.local/bin/sc_fw_decrypt

# system-wide
sudo install -m 0755 sc_fw_decrypt /usr/local/bin/sc_fw_decrypt
```

## Usage

```sh
# Decrypt all three; writes 40000001.dec etc. beside the inputs, CMAC-verified
sc_fw_decrypt 40000001 40000002 40000003

# Write outputs into a chosen directory
sc_fw_decrypt -C ./out 40000001 40000002 40000003

# Strip the 16-byte CMAC header, keep only the raw payload
sc_fw_decrypt --strip -o blnk.bin 40000001

# Pipe decrypted bytes to another tool
sc_fw_decrypt --stdout 40000003 | xxd | less
```

The `.dec` output keeps the 16-byte CMAC header; add `--strip` for payload only.
By default the tool refuses to write a file whose CMAC check fails
(use `--force` to write anyway, or `--no-verify` to skip the check).

### Options

| Option | Meaning |
|--------|---------|
| `-o, --output PATH` | output file path (single input only) |
| `-C, --outdir DIR`  | write outputs into `DIR` instead of beside each input |
| `-s, --suffix SUF`  | suffix for default output names (default `.dec`) |
| `--strip`           | drop the 16-byte CMAC header, write payload only |
| `--stdout`          | write decrypted bytes to stdout (single input) |
| `--no-verify`       | skip the AES-CMAC check |
| `--force`           | write output even if the CMAC check fails |
| `--key HEX`         | override the AES-128-CBC key (32 hex chars) |
| `--cmac-key HEX`    | override the AES-128-CMAC key (32 hex chars) |

## License

MIT — see [LICENSE](LICENSE).
