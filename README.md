# linc

Work with Linc data from the command line: the catalogue, measurement series, exports delivered as Parquet, webhooks and the access audit.

```bash
docker run --rm ghcr.io/lincworld/linc --help
```

## Images

| Tag | Holds |
|---|---|
| `latest`, `<version>` | The static binary and CA certificates. Nothing else, no shell |
| `alpine`, `<version>-alpine` | The same on Alpine, for CI jobs that run a shell |

`linux/amd64` and `linux/arm64`. Runs as user `65532`, not root.

| Path or variable | Use |
|---|---|
| `/data` | Working directory. Mount what `--out` and `convert` write to |
| `/config` | `config.toml` and profiles, secrets included: a container has no keychain |
| Entrypoint | `linc`. `docker run ghcr.io/lincworld/linc elements` is `linc elements` |

## Credentials

An access key and the organisation it acts in, from the environment:

```bash
export LINC_ORGANISATION=<organisation id>
export LINC_ACCESS_KEY_ID=<access key id>
export LINC_ACCESS_KEY_SECRET=<secret>

docker run --rm -e LINC_ORGANISATION -e LINC_ACCESS_KEY_ID -e LINC_ACCESS_KEY_SECRET \
	ghcr.io/lincworld/linc elements > elements.json
```

`-e NAME` without a value passes the variable on without putting the secret in the command line.

The secret from a file, as Docker and Kubernetes secrets arrive:

```bash
docker run --rm -v "$PWD/secret":/run/secrets/linc:ro -e LINC_ACCESS_KEY_SECRET_FILE=/run/secrets/linc \
	-e LINC_ORGANISATION -e LINC_ACCESS_KEY_ID ghcr.io/lincworld/linc elements
```

## As if installed

Profiles kept on the host, files written to the current directory, both owned by you, nothing else reachable:

```bash
mkdir -p ~/.linc
alias linc='docker run --rm -i $([ -t 1 ] && echo -t) --user "$(id -u):$(id -g)" \
	--read-only --cap-drop ALL --security-opt no-new-privileges \
	-v "$HOME/.linc":/config -v "$PWD":/data ghcr.io/lincworld/linc'

linc profiles add acme --claim <transfer id>
linc fields panel --match voltage
linc exports create panel:ABCDEFGH --fields 'telemetry.voltage_rms.*' --last P7D --out ./data
linc convert ./data --to csv
```

- `--claim` takes the transfer id from the access key's issue dialog: in your organisation's settings, issue a key, press **Add to the Linc CLI**, and run the command it shows. The organisation, key and secret come from it, so none is copied or typed. It works once, within 10 minutes.
- `linc profiles add` without `--claim` asks for the organisation, the key and its secret instead.
- `--claim` refuses to run beside `LINC_ORGANISATION`, `LINC_ACCESS_KEY_ID` or `LINC_ACCESS_KEY_SECRET`: unset them first.
- `--user` makes the files yours; without it they belong to `65532`.
- Create `~/.linc` first, or Docker creates it owned by root.
- `-t` only at a terminal: a table for a person, the API's JSON into a pipe.
- Paths are inside the container: `--out ./data` lands in `$PWD/data`. Nothing outside the mounts is reachable.
- Tried with Docker's usual daemon. Rootless Docker and Podman map users differently.

## In CI

Any runner with Docker:

```bash
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/data \
	-e LINC_ORGANISATION -e LINC_ACCESS_KEY_ID -e LINC_ACCESS_KEY_SECRET \
	ghcr.io/lincworld/linc:<version> exports create panel:ABCDEFGH --last P1D --out ./data
```

A job that runs inside the image needs the `alpine` tag: the runner starts it through a shell. `sh` as the first word runs the shell, anything else is `linc`:

```bash
docker run --rm ghcr.io/lincworld/linc:alpine sh -c 'linc --version && linc fields panel --match voltage_rms'
```

Set `LINC_RETRY` to wait out a rate limit or an outage: away from a terminal it does not retry.

## Without the network

`fields`, `convert`, `help` and `completions` need no network and no key:

```bash
docker run --rm --network none ghcr.io/lincworld/linc fields panel --match voltage_rms
docker run --rm --network none -v "$PWD":/data ghcr.io/lincworld/linc convert ./data --to arrow
```

Everything else needs `api.linc.world`; exports and `update` also download from `cdn.linc.world`.

## Locked down

The image needs no writable root, no capability and no new privilege:

```bash
docker run --rm --read-only --cap-drop ALL --security-opt no-new-privileges ghcr.io/lincworld/linc elements
```

A TLS-inspecting proxy: mount its CA certificate and set `LINC_CA_CERT` to its path.

## Updates and pinning

`linc update` changes nothing in a container: pull a newer tag. Images are not signed; pin a digest where it matters:

```bash
docker pull ghcr.io/lincworld/linc:<version>
docker inspect --format '{{index .RepoDigests 0}}' ghcr.io/lincworld/linc:<version>
```

The binary is the release's own. To check it against the release archive's:

```bash
docker cp "$(docker create ghcr.io/lincworld/linc:<version>)":/usr/local/bin/linc - | tar -xO | sha256sum
```

## Without Docker

Signed releases for Linux, macOS, Windows and FreeBSD, x86_64 and ARM64. The install scripts check the signature and the checksum, then run `linc install`. Read them before running them:

```bash
curl -fsSLO https://cdn.linc.world/cli/install.sh
sh install.sh
```

```powershell
Invoke-WebRequest https://cdn.linc.world/cli/install.ps1 -OutFile install.ps1
powershell -ExecutionPolicy Bypass -File install.ps1
```

They need nothing but `openssl`, which macOS and FreeBSD ship, and on Windows nothing at all. `linc update` keeps it current from then on, checking the same signature.

By hand:

- `https://cdn.linc.world/cli/latest`: the current version
- `https://cdn.linc.world/cli/<version>/manifest.json`: each platform's archive and sha256, signed in `manifest.json.sig` (RSA SHA-256, base64)
- `https://cdn.linc.world/cli/<version>/linc-<version>-<target>.tar.gz` (`.zip` on Windows)

The release key, as `linc.pem`:

```
-----BEGIN PUBLIC KEY-----
MIIBojANBgkqhkiG9w0BAQEFAAOCAY8AMIIBigKCAYEAswNOxFCRxq/9t29Y7neK
UR4jGsZVENNT5W3OQ0s3la3of7Sfe57ZwORn6GsIhTxjvETjK6h0CUZkqXZw1n6z
IMBGRtAMup9Ciowge6GWV/vUnwAyxk3wHfefSUy1Qeq2VJhb12XIsODGob8ysvy7
NYIXI+JMxMSHllMTghRVdpOcvi4PY+zKV3/IISvs7JooAviIoUMNxZEIglV2NA4G
NRmGV33dtyxn+OMhQZMIjgkjUOIi893FQLLE477mbfAyDFbBn04ghOg+hq+heev1
TIDh2B7fFWgvU7FZXtPEwoKpbF9x6NP/RH7xxXGg3EQT5Q6ky9XCjLvol/jeyGLl
7NCYOw3GCKGIVQwUHeorl0sxBtlrXTWu+YDeStLQcKydewzzED72U7RvPrLA3ftF
RdtLz/b5MGz8yFAW22UIGxp/NbX4/L0kOtv2ezFmVT/sM3JFaSrEcu1NhGC9bJiv
g+vvQXz3+NbXqOWSqR+loyjjDhHmIO/w9w9K8JpZV5LlAgMBAAE=
-----END PUBLIC KEY-----
```

Check the manifest's signature, then the archive's sha256 against it:

```bash
openssl base64 -d -A -in manifest.json.sig -out manifest.sig
openssl dgst -sha256 -verify linc.pem -signature manifest.sig manifest.json
```

Unpack, then `./linc install`.

## Documentation

`linc help --all` prints every command's help in one document. `linc docs` reads the platform's documentation under your access key.
