Guia rápida de instalacion de Gentoo



Esta guía expone cómo instalar Gentoo rápidamente. Gentoo es una distribución que pone su énfasis en las opciones. El handbook oficial de Gentoo sobre la instalación
es un buen recurso para conseguir un sistema Gentoo, pero como el propio manual explica, muchos pasos tienen varias opciones a elegir por el usuario, y se han elegido
las opciones más convenientes para la mayoría de casos. Por lo tanto, en esta guía se proponen otra serie de pasos a seguir, no como una alternativa "mejor" al
handbook, sino eligiendo otras opciones que pueden ser de interés para un usuario nuevo en Gentoo.

Se cubrirá una instalación de Gentoo dirigida a un PC de sobremesa para uso corriente (no para uso en servidor o en arquitecturas más exóticas), por lo que se usará:

- Paquetes binarios
- KDE Plasma
- Systemd
- BTRFS
- Arquitectura x86_64
- UEFI + GPT

Si alguna vez has realizado una instalación de Arch Linux, esta guía debería ser igual o más fácil.


0. Conseguir el medio de instalación
Para la instalación de Gentoo no hace falta usar el Live System de Gentoo, sirve cualquier distro de Linux. En este tutorial usaré el Live System de Ubuntu,
porque ofrece un entorno gráfico para abrir Firefox, usar el ratón en la terminal, etc.

Teniendo la ISO y el pendrive, con el programa [Balena Etcher](https://www.balena.io/etcher/), se flashea la ISO en el pendrive, apagamos el PC y lo iniciamos desde
el pendrive.

Si estamos en una máquina virtual, añadimos el disco ISO y arrancamos desde ahí. También tenemos que asegurarnos
de que el arranque del sistema sea con UEFI y no MBR.

> NOTA: si el sistema tiene Windows ya instalado, hay que desactivar el arranque rápido de Windows.

Abrimos una terminal, y cambiamos al superusuario para ejecutar todos los comandos:
```sh
sudo -s
```

Verificamos que hemos arrancado con UEFI (si no es el caso, el directorio estará vació):
```sh
ls /sys/firmware/efi/efivars
```

![efivars](img/efivars.png)


1. Preparar los discos

Lo primero es crear las particiones para nuestro sistema operativo. El formato de archivos será BTRFS. Ya está siendo adoptado por Fedora y ofrece una serie de características muy buenas, como compresión transparente, subvolúmenes, snapshots, etc.

Para identificar nuestro disco, podemos usar el comando `lsblk`:

![lsblk](img/lsblk.png)

El identificador de nuestro disco será `sd__`, para un disco común, `vd__` para un disco virtual, `nvme____` para NVME. Irá seguido de `a, b, c...` si tenemos varias discos del mismo tipo, y de `1, 2, 3...` si ya tenemos varias particiones. Para el NVME será `p1, p2, p3` para las particiones.

En este tutorial, el disco será `/dev/vda`, por favor cambia el nombre por tu disco en todos los pasos (por ejemplo, `/dev/vda2` por `/dev/sda2` o `/dev/nvme0n1p2`)

Vamos a borrarlo enteramente, creando una nueva tabla de particiones GPT:

- Partición 1: EFI de 300MB
- Particion 2: SWAP de 6GB
- Partición 3: sistema

El tamaño de la Swap puede ser el que quieras, existen muchos artículos que discuten cuánto elegir. Como regla fácil, elegir uno de estos:

- +6GB para poder compilar paquetes grandes con más seguridad
- +3/4 de la RAM para poner el sistema en suspensión
- 0GB si tenemos +34GB de RAM

Para partionar el disco con fdisk:

```sh
fdisk /dev/vda
```
- Con h sacamos el manual de fdisk
- Usamos el comando `g`, para crear una nueva tabla de particiones.
- Con `n`, creamos una nueva partición, elegimos el tamaño (último sector) con `+300M`, `+6G` (el primer sector por defecto).
- Con `t` elegimos el tipo (`1` para EFI, `19` para swap, `20` para el sistema)
- Con `w` guardamos la tabla de particiones.

Formateamos las particiones:

```sh
mkfs.vfat -F32 -n BOOT /dev/vda1
mkswap -L SWAP /dev/vda2
mkfs.btrfs -L SYSTEM /dev/vda3
```
Para el sistema BTRFS, vamos a crear un par de subvolúmenes:

```sh
mount /dev/vda3 /mnt/
cd /mnt
btrfs subvolume create @gentoo
btrfs subvolume create @home
btrfs subvolume set-default \@gentoo
cd /
umount -R /mnt
```

Montamos todos las particiones:
```sh
mkdir /mnt/gentoo
mount -o compress=lzo,space_cache,noatime /dev/vda3 /mnt/gentoo
mkdir /mnt/gentoo/{boot,home}
mount -o compress=lzo,space_cache,noatime,subvol=@home /dev/vda3 /mnt/gentoo/home
mount /dev/vda1 /mnt/gentoo/boot
swapon /dev/vda2
```

![partitions](img/partitions.png)

Si has llegado hasta aquí, las particiones de deberían ver así. Desde aquí la instalación es cuesta abajo.


2. Instalar el sistema base

Visitamos la página de descargas de Gentoo desde nuestro navegador (https://www.gentoo.org/downloads/) y copiamos la URL de la **stage 3 systemd**:

![stage3](img/gentoo_org.png)

Desde nuestra terminal, navegamos hasta `/mnt/gentoo` y descargamos la stage 3 (click derecho):

```sh
cd /mnt/gentoo
wget "<link que hemos copiado>"
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

Ahora vamos a hacer una primera configuración de nuestro `make.conf`. Es el principal archivo con el que configuramos portage, el gestor de paquetes de Gentoo. Hay que modificar varias líneas. Si ya existen, añadimos lo que falta, o añadimos la línea completa. Necesitamos saber el número de procesadores lógicos que tiene nuestro PC usando el comando `nproc`. En este caso, es `8`. Sustituir por el tuyo.

`nano /mnt/gentoo/etc/portage/make.conf`

- `MAKEOPTS="-j8 -l8"`
  > Compilar un paquete con varios hilos en paralelo
- `EMERGE_DEFAULT_OPTS="--ask --verbose --quiet-build=y --jobs=8 --load-average=6 --binpkg-respect-use=y --getbinpkg=y --with-bdeps=y"`
  > Opciones por defecto para pasar a portage. En load average, usar `nproc - 2`, en este caso `8 - 2 = 6`
- `PORTAGE_NICENESS="19"`
  > Dar la prioridad más baja a portage, para que podamos usar el PC normalmente
- `USE="dist-kernel harfbuzz"`
  > Con las USE flags controlamos características de los paquetes (más explicación al final). Con dist-kernel preparamos para usar un kernel precompilado
- `ACCEPT_LICENSE="*"`
  > Aceptar todas las licencias
- `VIDEO_CARDS="  "`
  > Rellenamos, dependiendo de nuestra tarjeta gráfica con uno de: `intel nvidia radeon vesa`
- `PORTAGE_BINHOST="https://gentoo.osuosl.org/experimental/amd64/binpkg/default/linux/17.1/x86-64/"`
- `GENTOO_MIRRORS="  "`
  > Rellenamos con un mirror http de aquí: https://www.gentoo.org/downloads/mirrors/

![make.conf](img/make.conf.png)


Configuramos los repositorios base con:

```sh
mkdir -p /mnt/gentoo/etc/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

1. Instalar el sistema

Entramos el sistema con un chroot para configurar portage y terminar la instalación:

```sh
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

chroot /mnt/gentoo /bin/bash
source /etc/profile
```

Una vez dentro del chroot, vamos a actualizar el sistema, instalar lo que falta y configurarlo.

Primero, descargamos las ebuilds del repositorio gentoo. Las Ebuild son las instrucciones de construcción de paquetes. Son simpleas archivos de texto que se encuentra en `/var/db/repos/gentoo`. Más tarde se instalarán otros repositorios de ebuilds.

```sh
emerge --sync
```

Muy importante antes de continuar, tenemos que elegir un perfil de portage, que acabamos de descargar con `--sync`. Los perfiles son unas `plantillas` de configuraciones, que facilitan instalar el sistema. Con la stage3 tendremos elegida una plantilla base, pero tenemos que elegir la plantilla `default/linux/amd64/17.1/desktop/plasma/systemd` (a la fecha de esta guía 17.1 es la última versión).

```sh
eselect profile set "default/linux/amd64/17.1/desktop/plasma/systemd"
```
Ahora sí, actualizamos el sistema e instalamos algunos paquetes de paso:

```sh
emerge --update --deep --newuse @world app-portage/gentoolkit app-portage/eix app-admin/sudo
```

![emerge](img/emerge.png)

Los paquetes que se descargaran como binarios se marcan en morado, y los que se compilan en verde. Como en nuestro `make.conf`
configuramos portage para que usase paquetes binarios respetando nuestras USE flags, los que no coincidan con USE flags que queremos instalar con las que existen en el servidor se compilarán. No deberían ser muchos, ya que el mirror usa el mismo perfil que hemos configurado.
