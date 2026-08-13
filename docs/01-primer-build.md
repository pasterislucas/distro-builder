# Fase 3: Primer build funcional (lb config + lb build)

## Objetivo
Generar la primera ISO booteable, mínima (sin entorno de escritorio),
para validar que el pipeline de `live-build` funciona de punta a punta.

## Configuración base

```bash
lb config \
  --distribution bookworm \
  --architecture amd64 \
  --binary-images iso-hybrid \
  --archive-areas "main contrib non-free-firmware"
```

## Lista de paquetes final (config/package-lists/my.list.chroot)
linux-image-amd64
live-boot
live-config
live-config-systemd
sudo
network-manager

## Errores encontrados durante el build

### 1. `E: Unable to locate package task-standard`
Intenté declarar `task-standard` pensando que era un paquete instalable,
pero es una *prioridad* interna de Debian (`required`/`important`/`standard`),
no un nombre de paquete de tasksel. **No hace falta**: `debootstrap` ya
instala por defecto los paquetes `required` + `important`, que alcanza
como base mínima.

### 2. `cp: cannot stat 'chroot/boot/vmlinuz-*': No such file or directory`
Ningún paquete de la lista original traía el kernel como dependencia.
Solución: agregar `linux-image-amd64` (metapaquete que apunta siempre
a la última versión de kernel disponible) a la lista de paquetes.

### 3. `E: the following stage is required to be done first: config`
Un `sudo lb clean` (sin flags) fue más agresivo de lo esperado y borró
también el marcador interno de la etapa `config` (no los archivos de
`config/`, que quedaron intactos). Solución: volver a correr `lb config`
con los mismos flags — es idempotente, no pisa `package-lists/`.

**Lección general**: `lb clean --binary` limpia solo el armado del ISO;
`lb clean` a secas resetea más etapas de lo que parece a simple vista
(incluida `config`). Preferir `--binary` como primer intento.

## Resultado

- ISO generada: `live-image-amd64.hybrid.iso` (~724MB)
- Verificación de tipo: ISO 9660, DOS/MBR boot sector, bootable
- Hash MD5 (build original): `bfc11ab1f48b97151c705ebfe60badd2`
- Transferida a Proxmox vía la PC como intermediario (la VM de build no
  tenía su key autorizada en el host — se resolvió sin generar una key
  nueva, ya que era una transferencia puntual)

## Boot test (VM 106, test-live-iso, sin disco, boot desde CD)

- [x] Bootloader isolinux muestra menú correcto (kernel 6.1.180-1,
      live-boot 1.20230131+deb12u1, live-config 11.0.3+nmu1)
- [x] Autologin como usuario `user` funciona
- [x] `sudo` funciona sin pedir password
- [x] Red por DHCP funciona (`network-manager`)
- [x] Conectividad a internet (ping externo)
- [ ] SSH — **falla con Connection refused**: `openssh-server` no está
      en la lista de paquetes. Pendiente para la próxima sesión, junto
      con la decisión de diseño de cómo autenticar (password fija vs
      SSH keys pre-cargadas vs dejarlo deshabilitado por defecto).

## Estado al cierre de esta fase

- [x] Primera ISO funcional generada y validada por boot test
- [x] Repo actualizado con la config real que generó el ISO que funciona
- [ ] SSH en el sistema live (próxima sesión)
- [ ] Branding (nombre, logo, MOTD) — fase siguiente
