# ARM64 Kbuild helper provenance

These ARM64 binaries were extracted from FriendlyElec's original
`linux-headers-6.1.141` package:

- `scripts/genksyms/genksyms`
- `scripts/mod/modpost`

They were validated against the FriendlyElec Linux 6.1.141 kernel
baseline used by this repository.

Corresponding kernel source:

- Repository: https://github.com/friendlyarm/kernel-rockchip
- Commit: `474962f3f7ff699a8e021b77183c667174088154`
- Branch at the time of testing: `nanopi6-v6.1.y`

Expected SHA-256 values are recorded in `SHA256SUMS`.