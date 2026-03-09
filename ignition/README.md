# Ignition Configuration

This directory contains [Butane](https://coreos.github.io/butane/) source files used to provision servers running the `ucore-by-agent` OCI image.

Butane (`.butane`) is the human-readable YAML format. It must be compiled to Ignition JSON (`.ign`) before use. Compiled `.ign` files are excluded from version control via `.gitignore` — they contain secrets.

## Files

| File | Purpose |
|---|---|
| `ucore-local.butane` | Local bare-metal installation and VM testing (x86_64) |

## Prerequisites

`butane` must be available on your local machine. The easiest way if you are on Aurora or another Universal Blue desktop is via the `coreos-tools` Distrobox container:

```bash
distrobox enter coreos-tools
```

Alternatively, install `butane` directly:

```bash
# Fedora / RHEL
sudo dnf install butane

# Or download the binary directly
curl -Lo butane https://github.com/coreos/butane/releases/latest/download/butane-x86_64-unknown-linux-gnu
chmod +x butane && sudo mv butane /usr/local/bin/
```

## Usage

### 1. Copy the template

```bash
cp ucore-local.butane ucore-local.local.butane
```

The `.local.butane` suffix is also gitignored — work from this copy.

### 2. Fill in the placeholders

Open `ucore-local.local.butane` and replace:

| Placeholder | How to get the value |
|---|---|
| `YOUR_SSH_PUB_KEY_HERE` | Contents of `~/.ssh/id_ed25519.pub` (or your preferred key) |
| `YOUR_GOOD_PASSWORD_HASH_HERE` | See password hash section below |

**Generate a password hash:**

```bash
# Option A — openssl (available everywhere)
openssl passwd -6

# Option B — Python (available everywhere)
python3 -c 'import crypt, getpass; print(crypt.crypt(getpass.getpass()))'
```

Uncomment the `password_hash` line in the butane file after filling it in. A password hash is only needed if you require console login (e.g. no network on first boot). SSH key alone is sufficient for normal use.

### 3. Compile to Ignition JSON

```bash
butane --pretty --strict ucore-local.local.butane > ucore-local.ign
```

Verify the output looks sane:

```bash
cat ucore-local.ign
```

### 4. Use the Ignition file

**Fedora CoreOS live ISO (bare metal or VM):**

Pass the Ignition file at boot via the kernel argument:

```
coreos.inst.ignition_url=file:///path/to/ucore-local.ign
```

Or if serving over HTTP during install:

```
coreos.inst.ignition_url=http://192.168.x.x:8080/ucore-local.ign
```

**VM (quick local test with `coreos-installer`):**

```bash
coreos-installer install /dev/sdX --ignition-file ucore-local.ign
```

## How the autorebase works

Stock Fedora CoreOS does not know about your custom OCI image. The Ignition config plants two one-shot systemd services that run on first boot:

1. **`ucore-unsigned-autorebase`** — rebases to your image without signature verification, then reboots.
2. **`ucore-signed-autorebase`** — rebases again with signature verification (cosign), then reboots.

After the second reboot the system is running `ucore-by-agent` and both services are disabled. Subsequent updates happen automatically via `rpm-ostree` whenever a new image is published to the registry.

## Adding new targets

When adding a new deployment target (e.g. Oracle OCI ARM64), create a new butane file in this directory:

```
ignition/
├── ucore-local.butane       # local / bare metal (x86_64)
├── ucore-oci.butane         # Oracle Cloud ARM64 (future)
└── README.md
```

Keep one butane file per deployment target. Placeholders stay in version control; compiled `.ign` files do not.