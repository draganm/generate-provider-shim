# Generate Terraform provider shim

This utility will generate Terraform provider shims for the community providers that are available in GitHub.

## Why shims?

In the Terraform 0.12.x world, there is no standardized way of installing providers that are not available from Hashicorp's registry, so one is encouraged to check in the binary of the provider together with Terraform files.
This leads to unnecessarily bloated git repositories.
Provider shim solves this issue by replacing the provider binary with a `bash` script that will download and cache on disk binary of the provider and execute it.
All of this is completely transparent to the terraform user and once created there is usually no further steps needed to use the providers.

An example of a shim script is:

```bash
#!/usr/bin/env bash
#
# Generated by generate-terraform-provider-shim: https://github.com/numtide/generate-terraform-provider-shim
#

set -e -o pipefail

plugin_url="https://github.com/numtide/terraform-provider-linuxbox/releases/download/v0.2.2/terraform-provider-linuxbox_v0.2.2_linux_amd64.tar.gz"
plugin_unpack_dir="${XDG_CACHE_HOME:-$HOME/.cache}/terraform-providers/linuxbox_v0.2.2"
plugin_binary_name="terraform-provider-linuxbox_v0.2.2"
plugin_binary_path="${plugin_unpack_dir}/${plugin_binary_name}"
plugin_binary_sha1="7232dbb6760d34e844ce731226b9eec67c5bb276"

if [[ ! -d "${plugin_unpack_dir}" ]]; then
    mkdir -p "${plugin_unpack_dir}"
fi

if [[ -f "${plugin_binary_path}" ]]; then
    current_sha=$(git hash-object "${plugin_binary_path}")
    if [[ $current_sha != "${plugin_binary_sha1}" ]]; then
        rm "${plugin_binary_path}"
    fi
fi

if [[ ! -f "${plugin_binary_path}" ]]; then
    curl -sL "${plugin_url}" | tar xzvfC - "${plugin_unpack_dir}"
    chmod 755 "${plugin_binary_path}"
fi

current_sha=$(git hash-object "${plugin_binary_path}")
if [[ $current_sha != "${plugin_binary_sha1}" ]]; then
    echo "plugin binary sha does not match ${current_sha} != ${plugin_binary_sha1}" >&2
    exit 1
fi

exec "${plugin_binary_path}" $@
```

With release of Terraform 0.13.x there is a standardized way of distributing the provider binaries, but the burden of running a provider registry and signing provider binaries is on the provider developers. 
This means that most of the providers will not be immediately available through a registry.
For such providers, provider shims is a stopgap solution until community providers can be downloaded through a registry.

## Usage

running

```sh
$ generate-terraform-provider-shim numtide/terraform-provider-linuxbox
```

will generate provider shims for Linuxbox provider in the `.terraform/plugins/` and `terraform.d/plugins/registry.terraform.io/hashicorp/linuxbox/<version>` directories for each arch respectively.

## Terraform 0.13 support

At the moment, there are plenty of community providers that are not available in the Hashicorp's registry and are not running registries of their own.
This will change with time and this tool will be less useful, but for the transition period, this tool will generate shims in the correct paths for both terraform 0.12 and 0.13.

Generated shims will let providers appear as if they were downloaded through Hashicorp's registry, so there is no need to add an entry in `terraform{ required_providers{} }` specifying the location of the provider registry.


## Limitations

### Only GitHub hosted providers
Shim generator is integrating tightly with GitHub's API to find release archives, hence will be able to generate shims only for the providers that are available on GitHub.

### Only providers with releases with attached binaries are supported
We are relying on developers of the providers to create releases with attached compiled binaries of the providers for different archs.
If that is not the case, there shim cannot be generated.

### Only .tar.gz and .zip archives are supported
There is no standard way of packaging providers.
Most of the time they are packaged in a Zip or Gzipped Tar archive - those are formats we are supporting.
If you would like us to add support for some other formats, please create an issue in this repo with a link to the specific provider you would like us to support.

### Windows is not supported
We do not have access to a Windows machine and have never run terraform in Windows environment, hence the generated shims will definitely not work under Windows.
