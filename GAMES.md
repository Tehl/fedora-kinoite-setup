flatpak install com.valvesoftware.Steam

flatpak install flathub dev.goats.xivlauncher
sudo flatpak override \
  --nofilesystem=home \
  --nofilesystem=/media \
  --nofilesystem=/run/media \
  --nofilesystem=/mnt \
  --filesystem ~/.xlcore \
  dev.goats.xivlauncher

flatpak install flathub com.vysp3r.ProtonPlus
sudo flatpak override \
  --nofilesystem=host \
  --nofilesystem=xdg-run/gvfsd \
  com.vysp3r.ProtonPlus

flatpak install flathub com.heroicgameslauncher.hgl
sudo flatpak override \
  --nofilesystem=/run/media \
  --nofilesystem=/mnt \
  --nofilesystem=/media \
  --filesystem=/mnt/games/Heroic \
  com.heroicgameslauncher.hgl

# Arknights: ENDFIELD
dwproton-10.0-15-x86_64

# EVE Online
GE-Proton-Latest

PROTON_USE_NTSYNC=1
LD_PRELOAD=

# Guild Wars 2
Kron4ek wine-11.5-staging-tkg-amd64
DXVK v2.7.1-1-gplasync

winetricks: tahoma

# Return of Reckoning
https://cdn2.steamgriddb.com/grid/516625bd0e26a4f2cf4c06607a97c9f5.png
GE-Proton-Latest

winetricks: corefonts d3dx9_34 d3dcompiler_47

# Where Winds Meet
Kron4ek wine-11.3-staging-tkg-amd64
DXVK v2.7.1-1-gplasync

nb wine 11.5 crashes during startup