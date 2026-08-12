# Fase 0-1: Setup del laboratorio en Proxmox

## Objetivo
Preparar una VM dedicada a build de la distro (`distro-builder`), con `live-build`
y herramientas asociadas, y dejar el repo de documentación conectado a GitHub.

## Infraestructura

- **Host**: Proxmox, Ryzen 5 7000, 16GB RAM, 1TB NVMe
- **Storage**: ZFS (`rpool/data`, mapeado como `local-zfs` en Proxmox)
- **VM base**: clon full de `debian-cloud-template` (VMID 9000)
- **VM resultante**: `distro-builder` (VMID 105)
  - 2 cores, 2GB RAM, 20GB disco (resize desde 3GB original)
  - Red: DHCP en `vmbr0` (misma LAN que el host)
  - Acceso: SSH directo desde la PC (no vía bastión, ya que no es una red
    aislada y esta VM es descartable/experimental)

## Comandos clave

```bash
# Clonar el template (full clone, no linked, para no acoplar el ciclo
# de vida de esta VM al template original)
qm clone 9000 105 --name distro-builder --full --storage local-zfs

qm set 105 --cores 2 --memory 2048
qm resize 105 scsi0 +17G
qm start 105

# Obtener la IP vía el guest agent (sin escanear la red a mano)
qm guest cmd 105 network-get-interfaces
```

## Herramientas instaladas en la VM

- `live-build` (20230502, versión estable de Debian 12 Bookworm)
- `debootstrap`, `xorriso`, `syslinux`, `isolinux`
- `grub-efi-amd64-bin` + `grub-pc-bin` (boot UEFI y BIOS legacy)
- `mtools`, `dosfstools` (partición FAT del ESP)
- `squashfs-tools`
- zsh + Oh My Zsh + Powerlevel10k (setup personal, no esencial para el build)

## Repo de documentación

- GitHub, público, licencia GPL-3.0
- Estructura: `docs/`, `config/` (config de live-build), `scripts/` (hooks/automatización)
- `.gitignore` excluye artefactos de `live-build` (`config/chroot`, `config/cache`,
  `config/binary`, `*.iso`, `*.img`) para no versionar GBs de builds

## Errores encontrados (y por qué quedan documentados)

1. **`~/.ssh/config` no se guardó en el primer intento.**
   El heredoc para crear el archivo no persistió (probablemente corrido en otra
   sesión). Resultado: SSH intentaba autenticar con la key default en vez de
   `github_distro`, dando `Permission denied (publickey)`.
   **Lección**: verificar con `cat` después de cada heredoc, no asumir que se escribió.

2. **Rama `master` vs `main`.**
   `git init` en esta versión de git usa `master` por defecto, pero GitHub
   espera `main`. El primer `git push -u origin main` falló con
   `error: src refspec main does not match any` porque esa rama no existía
   localmente.
   **Lección**: `git branch -M main` antes del primer push, o chequear
   `git branch` si el push falla por refspec.

3. **Historias no relacionadas al conectar el remoto.**
   El repo se creó en GitHub con LICENSE (commit inicial propio), mientras
   que local también tenía su propio commit inicial sin ancestro común.
   Se resolvió con `git pull origin main --allow-unrelated-histories`.

## Estado al cierre de esta fase

- [x] VM de build operativa y accesible por SSH
- [x] Herramientas de `live-build` instaladas
- [x] Repo en GitHub sincronizado
- [ ] Primer build mínimo con `lb config` / `lb build` (próxima fase)
