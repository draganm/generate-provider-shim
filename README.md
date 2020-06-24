# Generate terraform provider shim

This utility will generate terraform provider shims for the community providers that are available in GitHub.

## Usage

running

```sh
$ generate-terraform-provider-shim numtide/terraform-provider-linuxbox
```

will generate provider shims for linuxbox provider in the `.terraform/plugins/` directory.


## Why shims and not binaries?

Easiest way to use custom providers is to check them in with the terraform code.
Given that providers are usually >10mb each, that makes git repos really bloated.

Alternative to that, we could check in a bash script that will download the provider on demand and start it.

An exaple of a shim script is:

```bash
#!/usr/bin/env bash
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
    curl -L "${plugin_url}" | tar xzvfC - "${plugin_unpack_dir}"
    chmod 755 "${plugin_binary_path}"
fi

current_sha=$(git hash-object "${plugin_binary_path}")
if [[ $current_sha != "${plugin_binary_sha1}" ]]; then
    echo "plugin binary sha does not match ${current_sha} != ${plugin_binary_sha1}" >&2
    exit 1
fi

exec "${plugin_binary_path}" $@
```

This utility automates creation of such a script.
