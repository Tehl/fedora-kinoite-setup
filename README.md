
# fedora-kinoite-setup
Personal setup notes for Fedora 43 Kinoite Atomic Desktop.

## Why immutable Linux?
Much of my recent Linux experience has been either producing containerised applications at work, or hosting them as part of my home lab. As such I have a passing familiarity with how a Linux OS hangs together, but haven't really used it as a daily driver. And as a long-time Windows user, I'm also used to needing to clear down and reinstall everything every few years to keep things running smoothly.

These reasons all lead me to favour reproducability in a system; I'm happy to tinker to get something working, but once it does I want to know (and record) exactly which steps led to the desired outcome, so that I can do it again in the future. This repository is a part of that reproducability, as much a set of notes for myself as it is reference material for anybody else.

I initially looked into declarative distributions such as [NixOS](https://nixos.org/) or [GNU Guix](https://guix.gnu.org/) as there's something appealing about being able to commit some declarations to Git and reproduce the entire deployment if I manage to break something. However, both options seem to produce a system which differs significantly from a more traditional Linux deployment, which adds a layer of additional learning when getting used to the OS for the first time.

Immutable distributions such as [carbonOS](https://carbon.sh/), [Vanilla OS](https://vanillaos.org/) or [Fedora Atomic](https://www.fedoraproject.org/atomic-desktops/) seem to strike a good middle ground; the core OS is relatively safe from harm, with atomic updates, an easy rollback strategy if something goes wrong, and an auditable list of which modifications have been performed on top of the base image. Combined with [Flatpak](https://flatpak.org/) applications and [manifest](https://distrobox.it/usage/distrobox-assemble/) files for anything deployed via [Distrobox](https://distrobox.it), I should be able to maintain a clear view of the state of my system.

## Why Fedora Kinoite?
There are a few different gaming/performance-focused distributions making the rounds lately, such as [Bazzite](https://bazzite.gg/), [Nobara](https://nobaraproject.org/) and [CachyOS](https://cachyos.org/). Of these, Bazzite fits the bill as an immutable OS which comes pre-layered with game-ready drivers, launchers, and performance optimisations, which is very appealing as a plug-and-play solution.

However, projects like this are somewhat at the mercy of a small group of maintainers, and are by nature focused on the bleeding-edge of support. There's a few risks here, both in terms of longer term support but also in terms of both stability and security of what's being added into the upstream. It's all open source but understanding the provenance of every kernel patch or privileged service is a lot to take on if you care to.

In addition to gaming, I also use my PC for personal software development, 3D modelling and CAD, media consumption, and general web browsing. Once I started thinking about dual-booting Bazzite with a more mainstream distribution for e.g. online banking or administering my cloud hosting, I decided to just start with the upstream distribution it's based on, and add any modifications myself if I feel I need an extra % of performance here or there. After all, everything Bazzite is doing is open source, so I can pick and choose from the same options if I need to.

Bazzite is based on [Fedora Atomic](https://www.fedoraproject.org/atomic-desktops/), which has both GNOME ([Fedora Silverblue](https://fedoraproject.org/atomic-desktops/silverblue/)) and KDE Plasma ([Fedora Kinoite](https://fedoraproject.org/atomic-desktops/kinoite/)) desktops available (and others, but per the [Nobara FAQ](https://wiki.nobaraproject.org/FAQ/FAQ) only these options support Variable Refresh Rate on Wayland). As a long time Windows user, KDE Plasma feels immediately familiar, and while I had some initial issues with KWin crashes when using the built-in `nouveau` open-source Nvidia driver, switching to the proprietary drivers sorted this out, and I've been otherwise happy with KDE so far.

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