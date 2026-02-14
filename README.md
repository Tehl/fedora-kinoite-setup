
# fedora-kinoite-setup
Personal setup notes for Fedora 43 Kinoite Atomic Desktop.

## Why immutable Linux?
TBC

## Why Fedora Kinoite?
TBC

# Official Nvidia Drivers

## 1. Enable RPMFusion repositories
This two-step process installs a version-pinned repository definition from the RPMFusion website, and then replaces it with the unversioned definitions retrieved from the repo itself; this allows the definition to be updated automatically over time.

```bash
$ rpm-ostree install \
    https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
    https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
$ systemctl reboot
```
```bash
$ rpm-ostree update \
    --uninstall rpmfusion-free-release \
    --uninstall rpmfusion-nonfree-release \
    --install rpmfusion-free-release \
    --install rpmfusion-nonfree-release
$ systemctl reboot
```

ref: [docs.fedoraproject.org](https://docs.fedoraproject.org/en-US/quick-docs/rpmfusion-setup/#_enabling_the_rpm_fusion_repositories_for_ostree_based_systems)

## 2. Enable SecureBoot-compatible kernel signing
When using SecureBoot, kernel modules must be signed by a trusted party in order to boot successfully. Once we start installing kernel mods we either need to disable SecureBoot or provide a way for the modified kernel to be signed; we do this by enrolling a custom Machine Owner Key (MOK) with our BIOS and making this available for `rpm-ostree` to use to sign each image.

This is tricky to achieve as our MOK would not normally be available in the `rpm-ostree` build environment. This [tool](https://github.com/CheariX/silverblue-akmods-keys) by CheariX builds a local package which makes our key available for use.

```bash
$ rpm-ostree install --apply-live rpmdevtools akmods

$ sudo kmodgenca
$ sudo mokutil --import /etc/pki/akmods/certs/public_key.der

$ git clone https://github.com/CheariX/silverblue-akmods-keys
$ cd silverblue-akmods-keys

$ sudo bash setup.sh
rpm-ostree install akmods-keys-0.0.2-8.fc$(rpm -E %fedora).noarch.rpm
$ rm akmods-keys-0.0.2-8.fc$(rpm -E %fedora).noarch.rpm
```

ref: [github.com/CheariX](https://github.com/CheariX/silverblue-akmods-keys)

## 3. Install the Nvidia drivers from RPMFusion
Now that our modified kernel can be signed, we can install the drivers from RPMFusion.
```bash
$ rpm-ostree install \
    kmod-nvidia \
    xorg-x11-drv-nvidia \
    xorg-x11-drv-nvidia-cuda \
    libva-nvidia-driver
$ rpm-ostree kargs \
    --append=rd.driver.blacklist=nouveau,nova-core \
    --append=modprobe.blacklist=nouveau,nova-core \
    --append=nvidia-drm.modeset=1 \
    --append=initcall_blacklist=simpledrm_platform_driver_init
$ systemctl reboot
```
ref: [docs.fedoraproject.org](https://docs.fedoraproject.org/en-US/atomic-desktops/troubleshooting/#_using_nvidia_drivers)

**Note on installed packages:**
- `kmod-nvidia` and `xorg-x11-drv-nvidia` provide the actual drivers
- `libva-nvidia-driver` enables hardware video encoding/decoding using NVDEC
- `xorg-x11-drv-nvidia-cuda` is only required if you need CUDA support for something

**Note on driver versions:** Nvidia is ending support for Pascal and older GPUs with the 590 driver series. Once the main driver packages update to 590xx, you'll need to use the pinned 580xx packages if you still use an older graphics card:
```
nvidia-580xx-kmod
xorg-x11-drv-nvidia-580xx
```

# Mullvad VPN
```bash
$ curl --tlsv1.3 -fsS https://repository.mullvad.net/rpm/stable/mullvad.repo | sudo tee /etc/yum.repos.d/mullvad.repo
$ rpm-ostree update --install mullvad-vpn
$ systemctl reboot
```
```bash
$ systemctl enable --now mullvad-daemon.service
```
ref: [github.com/boredsquirrel](https://github.com/boredsquirrel/MullvadVPN-Tricks)

**Note:**
If you want to block access to the internet while not connected to Mullvad, you should also run
```bash
$ systemctl enable --now mullvad-early-boot-blocking.service
```
Make sure to log into the Mullvad client and set up your device credentials before enabling the blocking service, otherwise you won't be able to connect.