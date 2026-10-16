# sc_fw_tools

Tools for PS4 **syscon** (Renesas RL78) `sc_fw_update` firmware:

- **`sc_fw_decrypt`** — decrypt and CMAC-verify the components (BLNK / BASE / SYST).
- **`sc_fw_merge`** — merge the three components into a single flash image.

## Container format

Each component file is a nested container:

```
whole file  = AES-128-CBC( key, IV = 0 )           # encrypted
plaintext   = [ 16-byte AES-128-CMAC ][ payload ]  # after CBC decrypt
CMAC        = AES-128-CMAC( cmac_key ) over payload # == plaintext[0:16]
payload     = [ magic(4) ][ ver ][ nblk ][ pver ] ... block table @0x20 ...
```

| File       | Magic  | Role                                |
|------------|--------|-------------------------------------|
| `40000001` | `BLNK` | blank/skeleton image                |
| `40000002` | `BASE` | base firmware region                |
| `40000003` | `SYST` | system firmware region              |
| (patch)    | `PTCH` | patch, applied last                 |

Block table entry (at `0x20`, 8 bytes each): `u16 file_off, u16 size, u16 flash_addr`,
each shifted left by 8; `file_off` also gets `+0x110` (`+0x400` for `PTCH`).

### Keys

```
AES-128-CBC key : 5301C28824B57137A819C042FC119E3F   (IV = all zeros)
AES-128-CMAC key: 8F215691AC7EF6510239DD32CC6A2394
```

## Requirements

- Python 3.6+
- [pycryptodome](https://www.pycryptodome.org/) (only needed for encrypted input)

```sh
python3 -m pip install --user pycryptodome
# or: sudo apt install python3-pycryptodome
```

## Install

```sh
install -m 0755 sc_fw_decrypt sc_fw_merge ~/.local/bin/     # per-user
sudo install -m 0755 sc_fw_decrypt sc_fw_merge /usr/local/bin/   # system-wide
```

## sc_fw_decrypt

```sh
# decrypt all three; writes 40000001.dec etc. beside the inputs, CMAC-verified
sc_fw_decrypt 40000001 40000002 40000003

sc_fw_decrypt -C ./out 40000001 40000002 40000003   # into a directory
sc_fw_decrypt --strip -o blnk.bin 40000001           # payload only (drop CMAC hdr)
sc_fw_decrypt --stdout 40000003 | xxd | less
```

The `.dec` output keeps the 16-byte CMAC header; `--strip` writes payload only.
By default it refuses to write a file whose CMAC fails (`--force` / `--no-verify`).

## sc_fw_merge

Reproduces the chip's flashing sequence: start from an erased (`0xFF`) image and
lay each component's blocks at their flash addresses in order **BLNK → BASE → SYST
→ PTCH**, so later writes override earlier ones.

```sh
# inputs in any order, encrypted or .dec
sc_fw_merge 40000001 40000002 40000003 -o syscon_fw.bin
```

Options: `--size 0x60000`, `--fill 0xFF`, `--order BLNK,BASE,SYST`, `--no-verify`.
It prints the component list, the full block-placement map (flagging overrides and
clipping) and any uncovered regions. Default image size is the largest block end,
at least `0x60000` (384 KiB).

## License

MIT — see [LICENSE](LICENSE).
