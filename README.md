# Examples for [apt-transport-oci](https://github.com/AkihiroSuda/apt-transport-oci)

This repository contains example images for <https://github.com/AkihiroSuda/apt-transport-oci>,
an `apt-get` plugin to support distributing `*.deb` packages over an OCI registry such as `ghcr.io` .

> [!NOTE]
> "OCI" here refers to the "[Open Container Initiative](https://opencontainers.org/)", not to the "Oracle Cloud Infrastructure".

See [`.github/workflows/push.yml`](./.github/workflows/push.yml) for the GitHub Actions workflow that builds and pushes the image to
[`oci://ghcr.io/akihirosuda/apt-transport-oci-examples`](https://ghcr.io/akihirosuda/apt-transport-oci-examples).

## hello-apt-transport-oci

- Install [`apt-transport-oci`](https://github.com/AkihiroSuda/apt-transport-oci) (officially [packaged](https://repology.org/project/apt-transport-oci/versions) in Debian and Ubuntu since Debian 14 and Ubuntu 26.04):

```bash
sudo apt install apt-transport-oci
```

- Create `/etc/apt/sources.list.d/oci.sources` with the following content:
```
Types: deb
URIs: oci://ghcr.io/akihirosuda/apt-transport-oci-examples:latest
Suites: stable
Components: main
Signed-By: /etc/apt/keyrings/apt-transport-oci-examples.gpg
```

- Register a GPG key:
```bash
curl -fsSL https://raw.githubusercontent.com/AkihiroSuda/apt-transport-oci-examples/refs/heads/master/apt-transport-oci-examples.gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/apt-transport-oci-examples.gpg
```

- Run:
```bash
sudo apt update
sudo apt install hello-apt-transport-oci
```

- Make sure `hello-apt-transport-oci` is installed
```console
$ hello-apt-transport-oci
Hello, apt-transport-oci
```

- - -

# Appendix: Manually building and pushing an image

## Step 1: Prepare a GPG key pair

### Build an image locally

You can just use your existing GPG key pair you have been using for signing git commits.
Your public key is available at `https://github.com/USERNAME.gpg`.

### Build an image on GitHub Actions

While it is possible to upload your existing GPG key pair to GitHub Actions, it is highly recommended to create a new GPG key pair.

You can use a container or a VM to create a new GPG key pair:
```bash
docker run -it --rm -v alpine
```

```bash
# Run the following commands in the container, not on the host
apk add -q gnupg
gpg --batch --passphrase "PASSPHRASE" --quick-gen-key "apt-transport-oci-examples <example@example.com>" default default 3y
gpg --armor --export
gpg --armor --export-secret-key
```

Distribute your public key to your users, and add your private key to GitHub Actions as a [secret](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets) named `GPG_PRIVATE_KEY`.
See <https://github.com/crazy-max/ghaction-import-gpg>.

## Step 2: Create dpkg
Create `hello-apt-transport-oci_0.1_all.deb` from [`helloapt-transport-oci` directory](./hello-apt-transport-oci).

```bash
dpkg-deb --build --root-owner-group hello-apt-transport-oci hello-apt-transport-oci_0.1_all.deb
```

## Step 3: Create repository data

Use [aptly](https://www.aptly.info) to create the repository data.

```bash
sudo apt-get install aptly
```

```bash
aptly repo create hello-apt-transport-oci
aptly repo add hello-apt-transport-oci hello-apt-transport-oci_0.1_all.deb
aptly publish repo -distribution=stable -architectures=all,amd64,arm64 hello-apt-transport-oci
```

The repository data will be locally published on `~/.aptly/public`.

```console
$ tree ~/.aptly/public
/home/USER/.aptly/public
├── dists
│   └── stable
│       ├── Contents-all.gz
│       ├── Contents-amd64.gz
│       ├── Contents-arm64.gz
│       ├── InRelease
│       ├── main
│       │   ├── binary-all
│       │   │   ├── Packages
│       │   │   ├── Packages.bz2
│       │   │   ├── Packages.gz
│       │   │   └── Release
│       │   ├── binary-amd64
│       │   │   ├── Packages
│       │   │   ├── Packages.bz2
│       │   │   ├── Packages.gz
│       │   │   └── Release
│       │   ├── binary-arm64
│       │   │   ├── Packages
│       │   │   ├── Packages.bz2
│       │   │   ├── Packages.gz
│       │   │   └── Release
│       │   ├── Contents-all.gz
│       │   ├── Contents-amd64.gz
│       │   └── Contents-arm64.gz
│       ├── Release
│       └── Release.gpg
└── pool
    └── main
        └── h
            └── hello-apt-transport-oci
                └── hello-apt-transport-oci_0.1_all.deb

11 directories, 22 files
```

## Step 4: Push to an OCI registry
Push the deb file and the `Packages` file to an OCI registry (e.g., `ghcr.io`).

The easiest way is to use [ORAS](https://github.com/oras-project/oras) CLI.

```bash
sudo apt-get install oras
```

```bash
echo YOUR_GITHUB_PERSONAL_ACCESS_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

```bash
cd ~/.aptly/public
find . -type f -printf "%p:application/octet-stream\n" | xargs oras push ghcr.io/USERNAME/hello-apt-transport:latest
```

- See [GHCR documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) to learn
how to create a [GitHub Personal Access Token](https://github.com/settings/personal-access-tokens) (PAT) for `ghcr.io`.
- Use GitHub Web UI to make the image public / private.

## Step 5: Tell users the instruction

Tell your users how to install your package:

> - Create `/etc/apt/sources.list.d/oci.sources` with the following content:
> ```
> Types: deb
> URIs: oci://ghcr.io/USERNAME/hello-apt-transport-oci:latest
> Suites: stable
> Components: main
> Signed-By: /etc/apt/keyrings/USERNAME.gpg
> ```
>
> - Register a GPG key:
> ```bash
> curl -fsSL https://github.com/USERNAME.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/USERNAME.gpg
> ```
