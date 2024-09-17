# cagenix-os &nbsp; [![bluebuild build badge](https://github.com/coreywinkelmann/cagenix-os/actions/workflows/build.yml/badge.svg)](https://github.com/coreywinkelmann/cagenix-os/actions/workflows/build.yml)

See the [BlueBuild docs](https://blue-build.org/how-to/setup/) for quick setup instructions for setting up your own repository based on this template.

After setup, it is recommended you update this README to describe your custom image.

## Installation

> **Warning**  
> [This is an experimental feature](https://www.fedoraproject.org/wiki/Changes/OstreeNativeContainerStable), try at your own discretion.

To rebase an existing atomic Fedora installation to the latest build:

- First rebase to the unsigned image, to get the proper signing keys and policies installed:
  ```
  rpm-ostree rebase ostree-unverified-registry:ghcr.io/coreywinkelmann/cagenix-os:latest
  ```
- Reboot to complete the rebase:
  ```
  systemctl reboot
  ```
- Then rebase to the signed image, like so:
  ```
  rpm-ostree rebase ostree-image-signed:docker://ghcr.io/coreywinkelmann/cagenix-os:latest
  ```
- Reboot again to complete the installation
  ```
  systemctl reboot
  ```

The `latest` tag will automatically point to the latest build. That build will still always use the Fedora version specified in `recipe.yml`, so you won't get accidentally updated to the next major version.

## Post-install

### Nvidia

Run this after installation:

```
rpm-ostree kargs \
    --append=rd.driver.blacklist=nouveau \
    --append=modprobe.blacklist=nouveau \
    --append=nvidia-drm.modeset=1
```


## ISO

If build on Fedora Atomic, you can generate an offline ISO with the instructions available [here](https://blue-build.org/learn/universal-blue/#fresh-install-from-an-iso). These ISOs cannot unfortunately be distributed on GitHub for free due to large sizes, so for public projects something else has to be used for hosting.

## Verification

These images are signed with [Sigstore](https://www.sigstore.dev/)'s [cosign](https://github.com/sigstore/cosign). You can verify the signature by downloading the `cosign.pub` file from this repo and running the following command:

```bash
cosign verify --key cosign.pub ghcr.io/coreywinkelmann/cagenix-os
```

## SecureBoot setup

```bash
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=CagenixOS Kernel Module Signing/"
sudo mokutil --import MOK.der
```

Reboot and Enroll the Key: After the previous step, reboot your system. During the boot process, you will see the MOK Manager screen.

    Select "Enroll MOK".
    Choose "Continue" and confirm.
    Enter the password you set earlier when prompted.

```bash
sudo /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 MOK.priv MOK.der $(modinfo -n nvidia)
```
### Create Boot script

Create file `/usr/local/bin/sign-nvidia-modules.sh`

```bash
#!/bin/bash
for module in $(find /lib/modules/$(uname -r)/extra/nvidia* -type f -name '*.ko'); do
    /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 /root/MOK.priv /root/MOK.der $module
done
```

```bash
sudo chmod +x /usr/local/bin/sign-nvidia-modules.sh
systemctl reboot
```
