# 1. Iniciando el sistema
---
## Proceso de arranque

Al encender la máquina se ejecuta la siguiente secuencia:

1. Inicialización de la CPU.
2. La CPU localiza la dirección de memoria que almacena el firmware (BIOS o UEFI).
3. El firmware se inicializa y realiza el POST para verificar los elementos básicos de hardware.
4. Se carga el gestor de arranque (*boot loader*).
5. Se cargan en memoria el kernel del sistema operativo y el initramfs.
6. Una vez montado el sistema de ficheros raíz, el kernel busca el proceso `init` (históricamente en `/sbin/init`), que lee los ficheros de configuración e indica qué otros procesos deben iniciarse.

Si hay varios sistemas operativos o kernels instalados, Linux usa **GRUB** (*GRand Unified Bootloader*) para seleccionar cuál cargar en cada arranque.

---

## GRUB

Página oficial del proyecto: https://www.gnu.org/software/grub/index.html

### GRUB Legacy (versión antigua)

El fichero de configuración se encuentra en `/boot/grub/menu.lst` (o `/boot/grub/grub.conf`) y se lee directamente en el arranque.

Campos principales del fichero de configuración:

| Campo | Descripción |
|---|---|
| `title` | Nombre de la entrada que se muestra en el menú de arranque |
| `timeout` | Tiempo en segundos que se muestra el menú antes de arrancar la opción por defecto |
| `splashimage` | Imagen de fondo del menú |
| `root` | Disco y partición donde se encuentra el sector de arranque de GRUB |
| `kernel` | Fichero del kernel que se carga al arrancar ese sistema operativo |
| `chainloader` | Carga otro gestor de arranque en cadena (útil para arrancar Windows u otros OS) |

Más información: https://wiki.centos.org/HowTos/GrubInstallation

### GRUB 2 (versión moderna)

Admite condiciones lógicas para cargar módulos o mostrar entradas en el menú de forma condicional.

El fichero de configuración principal es `/boot/grub/grub.cfg`. **No debe editarse manualmente**, ya que se regenera automáticamente mediante un script y cualquier cambio directo se perdería.

Diferencias de sintaxis respecto a GRUB Legacy:

| GRUB Legacy | GRUB 2 |
|---|---|
| `title` | `menuentry` |
| `root` | `set root` |

Para modificar la configuración correctamente:

1. Editar los scripts en `/etc/grub.d/` y las variables en `/etc/default/grub`.
2. Regenerar el fichero principal ejecutando:

```bash
update-grub                         # Debian / Ubuntu
grub2-mkconfig -o /boot/grub2/grub.cfg   # Red Hat / Fedora / CentOS
```

---

## Niveles de arranque e inicialización

### Runlevels tradicionales

Linux define hasta 7 niveles de arranque (*runlevels*). Los estándar son:

| Nivel | Descripción |
|---|---|
| 0 | Apagado del sistema (*power off*) |
| 1 | Modo monousuario (*single-user*): solo `root`, sin servicios de red. Para mantenimiento crítico |
| 2 | Multiusuario sin red (varía según la distribución) |
| 3 | Multiusuario en modo consola, con red |
| 4 | Reservado para uso personalizado por el administrador |
| 5 | Multiusuario con entorno gráfico |
| 6 | Reinicio del sistema (*reboot*) |

Comandos para interactuar con los runlevels, apagar o reiniciar el sistema:

```bash
init <nivel>     # Cambia al runlevel indicado
shutdown         # Apaga o reinicia el sistema de forma controlada
reboot           # Reinicia el sistema
halt             # Detiene el sistema
poweroff         # Apaga el sistema
```

---

### SysV (System V)

Sistema de inicialización clásico heredado de Unix. El kernel lanza un proceso `init` que lee su fichero de configuración y arranca los scripts correspondientes al nivel de arranque activo.

##### Fichero principal: `/etc/inittab`

Formato de cada línea:

```
id:runlevels:action:process
```

Algunos valores posibles para el campo `action`:

| Valor | Descripción |
|---|---|
| `respawn` | Relanza el proceso si termina |
| `wait` | Espera a que el proceso termine antes de continuar |
| `once` | Ejecuta el proceso una sola vez |
| `sysinit` | Ejecuta el proceso durante la inicialización del sistema |
| `ctrlaltdel` | Se ejecuta al pulsar Ctrl+Alt+Supr |
| `powerwait` | Se ejecuta ante un evento de alimentación |

Ejemplo:

```
10:0:wait:/etc/init.d/rc 0
```

En el nivel de arranque 0 se ejecuta el script `rc 0`.

##### Scripts de arranque

Los scripts `rc X` llaman a los scripts del directorio `/etc/rcX.d/`. Los ficheros de estos directorios siguen la convención:

```
[K|S]NúmeroNombreServicio
```

- Los ficheros con prefijo **K** (*Kill*) se ejecutan primero para detener servicios.
- Los ficheros con prefijo **S** (*Start*) se ejecutan después para iniciarlos.
- El número determina el orden de ejecución.
- Todos son **enlaces simbólicos** a scripts reales ubicados en `/etc/init.d/`, los cuales aceptan al menos los parámetros `start` y `stop`.

Para añadir un nuevo servicio a un runlevel:

1. Colocar el script de inicio en `/etc/init.d/`.
2. Crear un enlace `SXXNombreServicio` en el directorio `/etc/rcX.d/` correspondiente.
3. Crear un enlace `KXXNombreServicio` para la parada. El número de orden de parada (*Kill*) debe ser el inverso al de arranque (*Start*).

> **Recomendación:** es mejor editar los scripts que se ejecutan desde `inittab` que modificar directamente el fichero `inittab`.

Herramientas para gestionar estos scripts de forma más cómoda: `ksysv` y `chkconfig`.

---

### Upstart

Sistema de inicialización más moderno que SysV, basado en **eventos**. Su principal ventaja es que se adapta mucho mejor a dispositivos de hardware que se conectan en caliente (*hot-plug*).

Mantiene compatibilidad con SysV. La configuración de cada servicio se gestiona mediante su fichero individual, ubicado en:

```
/etc/init/<nombre_proceso>.conf
```

---

### systemd (sistema actual)

La mayoría de las distribuciones modernas (Debian, Ubuntu, Red Hat, Fedora, etc.) han sustituido SysV y Upstart por **systemd**.

Diferencias principales respecto a los sistemas anteriores:

| Concepto SysV / Upstart | Equivalente en systemd |
|---|---|
| Runlevel 3 | `multi-user.target` |
| Runlevel 5 | `graphical.target` |
| Scripts en `/etc/init.d/` | Ficheros de unidad declarativos (`.service`) |
| `service` / `chkconfig` | `systemctl` |

Los servicios se gestionan con el comando `systemctl`:

```bash
systemctl start <servicio>      # Inicia un servicio
systemctl stop <servicio>       # Detiene un servicio
systemctl enable <servicio>     # Habilita el servicio en el arranque
systemctl disable <servicio>    # Deshabilita el servicio en el arranque
systemctl status <servicio>     # Muestra el estado del servicio
```

---

## Compilando e instalando programas

### Instalación desde código fuente

Para instalar software normalmente se usan los gestores de paquetes nativos (`.deb` en Debian/Ubuntu, `.rpm` en Red Hat/Fedora). Si el software no está disponible en los repositorios, es necesario compilarlo desde su código fuente.

#### Requisitos previos

- Compilador para el lenguaje del software (p. ej. `gcc` para C/C++).
- Herramientas de automatización (`make`).
- Librerías de desarrollo con sus archivos de cabecera (paquetes `-dev` o `-devel`).
- Utilidades para descomprimir el código fuente.
- La documentación de instalación incluida en el paquete (habitualmente `README` o `INSTALL`).

#### Pasos habituales

**1. Paquetes fuente RPM**

Si el fichero tiene extensión `.src.rpm`, se trata de código fuente empaquetado para sistemas RPM. Se compila y convierte en un paquete RPM binario instalable con:

```bash
rpmbuild --rebuild *.src.rpm
```

**2. Descomprimir el código fuente (tarball)**

```bash
tar -xvzf archivo.tgz
```

Opciones utilizadas:

| Opción | Significado |
|---|---|
| `-x` / `--extract` | Extrae el contenido del archivo |
| `-v` / `--verbose` | Muestra el progreso en pantalla |
| `-z` / `--gzip` | Descomprime usando gzip |
| `-f` / `--file` | Especifica el fichero de archivo |

**3. Configuración del entorno**

La mayoría del software incluye un script de configuración automática. Se ejecuta con:

```bash
./configure
```

Este script escanea el sistema, comprueba las dependencias y genera o adapta el fichero `Makefile`, que controla el proceso de compilación. Si el software no incluye este script, el administrador deberá crear o modificar el `Makefile` manualmente.

**4. Compilación**

```bash
make
```

Lee las instrucciones del `Makefile` y genera los binarios ejecutables a través del compilador.

**5. Instalación**

```bash
sudo make install
```

Copia los binarios compilados, librerías y manuales a las rutas definitivas del sistema (habitualmente bajo `/usr/local/`).

---

## Ejercicio: instalar el editor JED

```bash
cd /tmp
wget https://www.jedsoft.org/snapshots/jed-pre0.99.20-201.tar.gz
tar xzvf jed-pre0.99.20-201.tar.gz -C /usr/src
cd /usr/src/jed-pre0.99.20-201
cat INSTALL.unx                                              # Leer instrucciones de instalación
dnf install slang-devel                                      # Instalar dependencia necesaria
./configure --prefix=/usr/local --with-slang=/usr/include/slang
make clean
make
make install
```

---

# 2. Kernel de Linux
---
## ¿Qué es el kernel?

El kernel sirve de puente entre el hardware y el resto de funciones del sistema operativo (sistema de ficheros, acceso a la red, gestión de procesos, etc.).

El fichero principal del kernel reside generalmente en `/boot` (p. ej. `/boot/vmlinuz-<versión>`) y es el que indica al gestor de arranque qué cargar cuando el equipo arranca.

Además, se instalan módulos de kernel en `/lib/modules/`. Estos módulos contienen código objeto que puede extender el kernel en tiempo de ejecución, sin necesidad de recompilarlo. Su uso más habitual es dar soporte a nuevo hardware (drivers) o a nuevos sistemas de ficheros.

---

## Versiones y descarga

El kernel de Linux es común a todas las distribuciones y se publica en [www.kernel.org](https://www.kernel.org).

Para conocer la versión instalada en el sistema se usa:

```bash
uname -r
```

También es posible descargar actualizaciones incrementales (*patches*) y aplicarlas con el comando `patch`.

Las versiones se dividen en las siguientes categorías:

| Categoría | Descripción |
|---|---|
| **mainline** | Rama principal de desarrollo, mantenida por Linus Torvalds |
| **stable** | Versiones estables para producción, con correcciones de seguridad y fallos |
| **longterm (LTS)** | Versiones con soporte extendido durante varios años |
| **linux-next** | Rama de integración para el siguiente ciclo de desarrollo |
| **EOL** (*End of Life*) | Versiones que ya no reciben mantenimiento |

> **Nota:** Las *release candidates* (RC) son versiones preliminares de mainline, no una categoría independiente. Aparecen con la forma `6.x-rc1`, `6.x-rc2`, etc.

El código fuente descargado de kernel.org se extrae en el directorio que el usuario elija. Si la distribución proporciona el paquete de fuentes del kernel, éste suele instalarse en `/usr/src/` e incluye documentación útil.

---

## Identificación del hardware

Antes de configurar o compilar un kernel es fundamental conocer bien el hardware de la máquina. Especial atención a:

- Controladora de discos
- Adaptador de red
- Controlador de vídeo
- Controlador USB

Comandos útiles para identificar el hardware:

| Comando | Función |
|---|---|
| `lspci` | Lista los dispositivos conectados al bus PCI |
| `lsusb` | Lista los dispositivos USB conectados |
| `lsmod` | Lista los módulos de kernel cargados actualmente en memoria |
| `modinfo <módulo>` | Muestra información detallada sobre un módulo concreto |

> **Importante:** `lsmod` solo muestra módulos cargados dinámicamente. Los drivers compilados directamente dentro del kernel no aparecen en su salida.

---

## Configuración del código fuente

La configuración se realiza con el comando `make` sobre el directorio de fuentes. Pasos habituales:

```bash
make mrproper       # Elimina temporales y configuraciones anteriores (limpieza total)
make defconfig      # Crea un fichero de configuración por defecto para la arquitectura actual
make oldconfig      # Adapta una configuración anterior al nuevo kernel
make allmodconfig   # Configura el máximo número de opciones como módulos
```

Para editar la configuración de forma interactiva:

```bash
make config         # Interfaz basada en texto (pregunta opción a opción)
make menuconfig     # Interfaz de menú en terminal (ncurses)
make gconfig        # Interfaz gráfica (GTK)
make xconfig        # Interfaz gráfica (Qt)
```

Para más información: `make help` o el directorio `Documentation/` dentro del árbol de fuentes.

---

## Compilación e instalación del kernel

### Paquetes necesarios

| Paquete | Para qué se necesita |
|---|---|
| `Development Tools` | Compilador gcc, make y herramientas básicas |
| `ncurses-devel` | Para usar `make menuconfig` |
| `qt-devel`, `libxi-devel` | Para usar `make xconfig` |

### Pasos

**1. Compilar el kernel y los módulos:**

```bash
make              # Compila la imagen del kernel y todos los módulos a la vez
```

También se pueden separar los pasos:

```bash
make bzImage      # Compila solo la imagen del kernel (arquitecturas x86)
make modules      # Compila solo los módulos
```

**2. Instalar la imagen del kernel:**
Copiarla al directorio `/boot`. En arquitecturas x86 la imagen compilada se encuentra en `arch/x86/boot/bzImage`.

**3. Instalar los módulos:**

```bash
make modules_install
```

Crea un nuevo directorio en `/lib/modules/<versión>/` y copia allí los módulos compilados.

**4. Preparar el disco inicial de RAM (`initramfs`):**

El `initramfs` contiene los módulos y utilidades críticos que el kernel necesita en el arranque temprano, antes de montar el sistema de ficheros raíz. Herramienta según la distribución:

| Herramienta | Distribución |
|---|---|
| `dracut` | Fedora, RHEL, CentOS, openSUSE |
| `mkinitramfs` | Debian, Ubuntu |
| `mkinitrd` | Herramienta antigua, prácticamente obsoleta en distros modernas |

**5. (Opcional) Crear un paquete RPM:**
Simplifica la distribución e instalación del kernel. Requiere los objetivos `rpm-pkg` o `binrpm-pkg`.

**6. Añadir el kernel al gestor de arranque GRUB.**

> La mayoría de estos pasos pueden automatizarse con `make install`.

Guía oficial para compilar un kernel personalizado en CentOS/RHEL:
https://wiki.centos.org/HowTos/Custom_Kernel

---

## Gestión de módulos

### Carga de módulos

| Comando | Descripción |
|---|---|
| `insmod` | Inserta un único módulo. Requiere la **ruta completa** al fichero `.ko`. No resuelve dependencias automáticamente |
| `modprobe` | Inserta un módulo por su **nombre**. Resuelve y carga automáticamente los módulos dependientes consultando `modules.dep` |

### Descarga de módulos

| Comando | Descripción |
|---|---|
| `rmmod <módulo>` | Elimina el módulo indicado. Con `-w` espera a que deje de estar en uso |
| `modprobe -r <módulo>` | Descarga el módulo y sus dependencias si ya no son necesarias (más seguro que `rmmod`) |

### Consulta de información

```bash
lsmod                  # Lista los módulos cargados y la memoria que consume cada uno
modinfo <módulo>       # Muestra parámetros, dependencias, autor, licencia, etc.
```

### Gestión de dependencias

Las dependencias entre módulos se registran en:

```
/lib/modules/<versión>/modules.dep
```

Para regenerar este fichero tras instalar un módulo manualmente:

```bash
depmod
```

> **Nota:** `depmod` **no** descarga módulos. Su función es recalcular y actualizar el mapa de dependencias que usa `modprobe`.

### Ficheros de configuración de módulos

La configuración de módulos (parámetros, alias, listas negras) se gestiona en:

```
/etc/modprobe.d/*.conf
```

> Los ficheros `/etc/modprobe.conf` y `/etc/modules.conf` son de sistemas muy antiguos (kernels 2.4 / primeros 2.6) y no se usan en distribuciones modernas.

# 3. Archivos de sistema y dispositivos

---

## Sistemas de ficheros en Linux

Linux admite una gran variedad de sistemas de ficheros que pueden convivir en distintos discos o volúmenes montados en el sistema. A diferencia de Windows, no existen letras de unidad: todo se integra en un único árbol de directorios.

### Sistemas de ficheros más habituales

#### Familia Extended (Ext)

| Sistema                                    | Descripción                                                                                                                                                                                                                                                          |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Ext2](https://es.wikipedia.org/wiki/Ext2) | Sistema clásico de Linux, sin journaling. Adecuado para medios flash donde el journaling no aporta ventajas                                                                                                                                                          |
| [Ext3](https://es.wikipedia.org/wiki/Ext3) | Ext2 con journaling. Mejora la recuperación tras fallos pero sin ventajas de rendimiento significativas respecto a Ext4                                                                                                                                              |
| [Ext4](https://es.wikipedia.org/wiki/Ext4) | El sistema más usado en Linux. Soporta volúmenes de hasta 1 exabyte y ficheros de hasta 16 TiB. Journaling de metadatos, asignación tardía (*delayed allocation*) y extents para reducir la fragmentación. Es el sistema por defecto en la mayoría de distribuciones |

#### Sistemas de alto rendimiento y modernos

| Sistema | Descripción |
|---|---|
| [XFS](https://es.wikipedia.org/wiki/XFS) | Desarrollado originalmente por Silicon Graphics (1993), portado a Linux en 2001. Diseñado para alto rendimiento con ficheros grandes y operaciones paralelas. Journaling de metadatos. Es el sistema por defecto en Red Hat Enterprise Linux |
| [Btrfs](https://es.wikipedia.org/wiki/Btrfs) | Sistema nativo moderno del kernel de Linux. Basado en *Copy-on-Write* (CoW): las modificaciones se escriben en nuevos bloques sin sobreescribir los anteriores. Ofrece snapshots instantáneos, checksums en datos y metadatos, compresión transparente y RAID integrado. Predeterminado en openSUSE y Fedora |
| [ZFS](https://es.wikipedia.org/wiki/ZFS) | Desarrollado por Sun Microsystems (2005). Sistema muy robusto orientado a servidores y NAS: integridad de datos mediante checksums, gestión de volúmenes integrada y snapshots. Su licencia CDDL le impide integrarse directamente en el kernel de Linux y requiere un módulo aparte (`zfs-dkms`) |

#### Sistemas de interoperabilidad

| Sistema | Descripción |
|---|---|
| [FAT / FAT32](https://es.wikipedia.org/wiki/Tabla_de_asignaci%C3%B3n_de_archivos) | Sistema heredado de DOS/Windows. Ampliamente soportado para memorias USB y tarjetas SD. Límite de 4 GiB por fichero en FAT32 |
| [NTFS](https://es.wikipedia.org/wiki/NTFS) | Sistema nativo de Windows NT y versiones posteriores. Linux accede a él mediante el módulo `ntfs3` (incluido en el kernel desde la versión 5.15) o la herramienta en espacio de usuario `ntfs-3g` |
| [HFS / HFS+](https://es.wikipedia.org/wiki/HFS+) | Sistemas de Apple. Linux los soporta en modo lectura |
| [ISO 9660](https://es.wikipedia.org/wiki/ISO_9660) | Estándar para imágenes de CD-ROM |
| [UDF](https://es.wikipedia.org/wiki/Universal_Disk_Format) | *Universal Disk Format*, utilizado en DVD y Blu-ray |

> **Recomendación práctica:** Ext4 es la opción más segura para uso general por su madurez y compatibilidad. XFS es preferible para cargas de trabajo con ficheros de gran tamaño. Btrfs es la mejor elección cuando se necesitan snapshots, compresión o RAID software avanzado.

---

## Montar un sistema de ficheros

En Linux/Unix existe un único árbol de directorios. No hay letras de unidad como en Windows. **Montar** es la acción de integrar un sistema de ficheros dentro de ese árbol, haciéndolo accesible desde un directorio llamado **punto de montaje**.

> **Importante:** Antes de poder montar una partición nueva, esta debe contener un sistema de ficheros válido. Normalmente primero se crea una partición y después se formatea con herramientas como `mkfs.ext3`, `mkfs.ext4`, `mkfs.xfs`, etc.

Ejemplo:

```bash
# Crear un sistema de ficheros ext4
mkfs.ext4 /dev/sdb1

# Crear el punto de montaje
mkdir -p /mnt/datos

# Montar el sistema de ficheros
mount /dev/sdb1 /mnt/datos
```

El punto de montaje debe existir previamente. Si contiene archivos, estos quedarán ocultos mientras el sistema de ficheros permanezca montado.

### Comando `mount` y `umount`

```bash
mount <dispositivo> <punto_de_montaje>       # Monta un sistema de ficheros
mount -t ext4 /dev/sdb1 /mnt/datos          # Especificando el tipo explícitamente
mount -o ro /dev/sdb1 /mnt/datos            # Montaje en solo lectura

umount <punto_de_montaje>                    # Desmonta el sistema de ficheros
umount /dev/sdb1                             # También puede usarse el dispositivo
```

Si no se especifica `-t`, `mount` intentará detectar automáticamente el tipo de sistema de ficheros.

También es posible montar entradas definidas en `/etc/fstab`:

```bash
mount /mnt/datos     # Monta la entrada correspondiente de fstab
mount -a             # Monta todas las entradas válidas de fstab
```

`mount -a` es especialmente útil para comprobar cambios en `/etc/fstab` antes de reiniciar el sistema.

Opciones de montaje más comunes (se pasan con `-o`):

|Opción|Descripción|
|---|---|
|`ro` / `rw`|Solo lectura / lectura y escritura|
|`noexec`|Impide ejecutar binarios en esa partición|
|`nosuid`|Ignora los bits SUID/SGID|
|`nodev`|Ignora archivos especiales de dispositivo|
|`noatime`|No actualiza el tiempo de acceso al leer ficheros (mejora el rendimiento)|
|`relatime`|Actualiza `atime` solo cuando es necesario (comportamiento habitual en sistemas modernos)|
|`defaults`|Equivale aproximadamente a `rw,suid,dev,exec,auto,nouser,async`|
|`nofail`|No provoca un error de arranque si el dispositivo no está disponible|

Para ver los sistemas de ficheros actualmente montados:

```bash
findmnt        # Vista jerárquica recomendada
mount          # Lista todos los montajes activos
cat /proc/mounts
df -h          # Espacio usado y disponible en cada sistema montado
```

---

### Montaje permanente: `/etc/fstab`

El fichero `/etc/fstab` controla qué sistemas de ficheros pueden montarse automáticamente durante el arranque o mediante `mount`.

Cada línea tiene **seis campos** separados por espacios o tabulaciones:

```
<dispositivo>  <punto_montaje>  <tipo_fs>  <opciones>  <dump>  <fsck>
```

Ejemplo real:

```
UUID=9a8b7c6d-5e4f-3210-abcd-ef1234 /          ext4  errors=remount-ro  0  1
UUID=ABC1-DEF2                    /boot/efi   vfat  umask=0077         0  1
/swapfile                         none        swap  sw                 0  0
```

Descripción de cada campo:

|Campo|Descripción|
|---|---|
|**Dispositivo**|Identificador del dispositivo. Se recomienda usar `UUID=<valor>` o `LABEL=<valor>` en lugar del nombre de dispositivo (`/dev/sda1`) para evitar problemas si cambia el orden de detección de discos|
|**Punto de montaje**|Directorio donde se montará el sistema de ficheros. Para swap se usa `none`|
|**Tipo de FS**|Sistema de ficheros: `ext4`, `xfs`, `btrfs`, `vfat`, `swap`, `nfs`, etc.|
|**Opciones**|Opciones de montaje separadas por comas. `defaults` es el valor más habitual|
|**Dump**|`1` activa las copias de seguridad mediante la utilidad `dump` (prácticamente obsoleta); `0` la desactiva. Casi siempre se usa `0`|
|**fsck**|Orden de comprobación con `fsck` al arrancar|

Valores habituales para `fsck`:

|Valor|Significado|
|---|---|
|`0`|No comprobar|
|`1`|Comprobar primero (normalmente solo para `/`)|
|`2`|Comprobar después de los sistemas marcados con `1`|

XFS, Btrfs y la mayoría de sistemas de ficheros de red suelen utilizar `0` porque no emplean comprobaciones mediante `fsck` de la forma tradicional.

Para obtener el UUID de un dispositivo:

```bash
blkid                  # Muestra UUID y tipo de todos los dispositivos
blkid /dev/sdb1        # Solo para un dispositivo concreto
lsblk -f               # Vista alternativa con árbol de particiones
```

---

### Flujo habitual con una partición nueva

```bash
# 1. Crear la partición (fdisk, gdisk, parted, etc.)

# 2. Crear el sistema de ficheros
mkfs.ext4 /dev/sdb1

# 3. Crear el punto de montaje
mkdir -p /mnt/datos

# 4. Montar el sistema de ficheros
mount /dev/sdb1 /mnt/datos

# 5. Obtener el UUID
blkid /dev/sdb1

# 6. Añadir una entrada en /etc/fstab
UUID=<uuid> /mnt/datos ext4 defaults 0 2

# 7. Comprobar la configuración
mount -a
```

> **Advertencia:** Un `/etc/fstab` mal configurado puede impedir que el sistema arranque correctamente. Es recomendable hacer una copia de seguridad antes de editarlo y verificar los cambios con `mount -a` sin reiniciar.
---

### Automontadores

Los automontadores montan automáticamente un sistema de ficheros en el momento en que se accede a él, y lo desmontan pasado un tiempo de inactividad.

#### Automontaje de dispositivos locales (escritorio)

Los gestores de archivos gráficos integran automontado para dispositivos extraíbles:

- [Thunar](https://es.wikipedia.org/wiki/Thunar) (XFCE)
- [Nautilus / GNOME Archivos](https://es.wikipedia.org/wiki/GNOME_Archivos) (GNOME)
- [Dolphin](https://es.wikipedia.org/wiki/Dolphin_(administrador_de_archivos)) (KDE)

Internamente, estos gestores se apoyan en **udisks2** y **GVfs** para el automontado mediante D-Bus.

#### Automontaje de sistemas de ficheros en red: `autofs`

Los sistemas de ficheros en red (NFS, SMB/CIFS) no conviene tenerlos montados permanentemente en `/etc/fstab` porque consumen ancho de banda y requieren verificar continuamente la disponibilidad del servidor remoto. Para estos casos se usa **`autofs`**.

`autofs` monta el recurso al acceder a él y lo desmonta tras un tiempo de inactividad (por defecto, 60 segundos). Referencia oficial: [`man 5 auto.master`](https://man7.org/linux/man-pages/man5/auto.master.5.html)

Ficheros de configuración principales:

| Fichero | Descripción |
|---|---|
| `/etc/auto.master` | Mapa maestro: lista los puntos de montaje gestionados por autofs y sus mapas asociados |
| `/etc/auto.<nombre>` | Mapa específico: define qué recursos se montan bajo cada punto |

Formato de `/etc/auto.master`:

```
<punto_montaje>  <fichero_mapa>  [opciones]
/mnt/nfs         /etc/auto.nfs  --timeout=60
```

Formato de un mapa específico (p. ej. `/etc/auto.nfs`):

```
<clave>  [opciones]  <ubicación_remota>
datos    -rw,soft    servidor:/export/datos
```

Para recargar la configuración de `autofs` sin reiniciar:

```bash
systemctl reload autofs      # En sistemas con systemd
automount --reload           # Método clásico
```

---

## Mantenimiento de un sistema de ficheros

### Creación de sistemas de ficheros

El comando `mkfs` es el punto de entrada estándar. Actúa como *wrapper* que delega en la utilidad específica de cada tipo de sistema de ficheros. Referencia: [`man 8 mkfs`](https://www.man7.org/linux/man-pages/man8/mkfs.8.html)

```bash
mkfs -t ext4 /dev/sdb1          # Forma genérica
mkfs.ext4 /dev/sdb1             # Llamada directa a la utilidad específica (equivalente)
mkfs.xfs /dev/sdb1
mkfs.btrfs /dev/sdb1
```

> **Nota:** En la práctica se prefiere llamar directamente a las utilidades específicas (`mkfs.ext4`, `mkfs.xfs`, etc.) ya que ofrecen más opciones y control.

Opciones habituales de `mke2fs` / `mkfs.ext4`:

| Opción | Descripción |
|---|---|
| `-t <tipo>` | Tipo de sistema de ficheros (`ext2`, `ext3`, `ext4`) |
| `-L <etiqueta>` | Asigna una etiqueta al volumen (alternativa al UUID en fstab) |
| `-m <porcentaje>` | Porcentaje de bloques reservados para root (por defecto 5%) |
| `-j` | Activa journaling (crea un Ext3 si el tipo base es Ext2) |

---

### Comprobación de errores: `fsck`

`fsck` comprueba y repara sistemas de ficheros. Internamente llama a la herramienta adecuada según el tipo de sistema (`e2fsck` para Ext, `xfs_repair` para XFS, etc.). Referencia: [`man 8 fsck`](https://man7.org/linux/man-pages/man8/fsck.8.html)

```bash
fsck /dev/sdb1           # Comprobación básica
fsck -n /dev/sdb1        # Solo lectura: muestra errores sin repararlos
fsck -y /dev/sdb1        # Responde "sí" automáticamente a todas las reparaciones
fsck -A                  # Comprueba todos los sistemas de ficheros listados en /etc/fstab
```

> **Importante:** Se recomienda desmontar el sistema de ficheros antes de ejecutar `fsck`. Ejecutarlo sobre un sistema de ficheros montado en escritura puede dañar los datos. La única excepción es el sistema raíz `/`, que se comprueba automáticamente en el arranque antes de ser montado en modo escritura.

---

### Inspección y ajuste fino (*tuning*)

#### Sistemas Ext2/3/4

| Comando | Función |
|---|---|
| `dumpe2fs /dev/sdb1` | Muestra información detallada del superbloque y grupos de bloques |
| `tune2fs /dev/sdb1` | Ajusta parámetros del sistema de ficheros sin reformatear |
| `debugfs /dev/sdb1` | Consola interactiva de bajo nivel: permite manipular inodos, bloques y ficheros individuales |

Opciones útiles de `tune2fs`. Referencia: [`man 8 tune2fs`](https://man7.org/linux/man-pages/man8/tune2fs.8.html)

```bash
tune2fs -l /dev/sdb1               # Lista los parámetros actuales del sistema de ficheros
tune2fs -L "MisDatos" /dev/sdb1    # Cambia la etiqueta del volumen
tune2fs -m 1 /dev/sdb1             # Reduce el espacio reservado para root al 1%
tune2fs -c 30 /dev/sdb1            # Fuerza fsck cada 30 montajes
tune2fs -i 30d /dev/sdb1           # Fuerza fsck cada 30 días
tune2fs -U random /dev/sdb1        # Genera y asigna un nuevo UUID aleatorio
```

#### Sistema XFS

| Comando | Función |
|---|---|
| `xfs_info /dev/sdb1` | Muestra los parámetros del sistema de ficheros XFS (equivale a `dumpe2fs`) |
| `xfs_admin /dev/sdb1` | Modifica parámetros (etiqueta, UUID) sin reformatear |
| `xfs_repair /dev/sdb1` | Repara un sistema de ficheros XFS dañado (equivale a `e2fsck`) |

> **Nota:** El comando mencionado en algunos materiales como `xsf_info` es incorrecto; el comando correcto es `xfs_info`.

#### Sistema ReiserFS

| Comando | Función |
|---|---|
| `debugreiserfs /dev/sdb1` | Muestra información del superbloque de ReiserFS |
| `reiserfstune /dev/sdb1` | Ajusta parámetros del sistema de ficheros ReiserFS |

> **Nota:** ReiserFS está considerado obsoleto y en desuso en las distribuciones modernas. Su desarrollo se detuvo y no se recomienda para nuevas instalaciones.

---

### Áreas de swap

Los sistemas operativos modernos implementan **memoria virtual**: cuando la RAM se agota, el kernel mueve páginas de memoria de procesos inactivos a un área de almacenamiento en disco denominada **swap** o espacio de intercambio. Esto permite que los procesos crean que disponen de más memoria de la físicamente instalada.

El swap puede residir en una **partición dedicada** o en un **fichero** dentro de un sistema de ficheros existente.

#### Creación e inicialización

```bash
mkswap /dev/sdb2          # Inicializa una partición como área de swap
mkswap /fichero_swap      # También funciona con un fichero regular
```

#### Activación y desactivación

```bash
swapon /dev/sdb2          # Activa el área de swap
swapoff /dev/sdb2         # Desactiva el área de swap
swapon --show             # Muestra las áreas de swap activas y su uso
free -h                   # Muestra el uso de RAM y swap de forma resumida
```

#### Swap permanente en `/etc/fstab`

Para que el área de swap se active automáticamente en cada arranque, debe añadirse una entrada en `/etc/fstab`:

```
UUID=<uuid>   none   swap   sw   0   0
```

> **Nota:** Las particiones y ficheros de swap **no almacenan un sistema de ficheros** convencional ni se montan en un directorio; de ahí que el punto de montaje sea `none` y el tipo de FS sea `swap`. Por este motivo tampoco aparecen en la salida de `df`, pero sí en `swapon --show` y `free`.

---

## Gestión de dispositivos con udev

`udev` es el subsistema encargado de gestionar dinámicamente los nodos de dispositivo en el sistema de ficheros virtual `/dev`. Es la implementación actual en prácticamente todas las distribuciones modernas, integrado con systemd. Referencia oficial: [`man 7 udev`](https://man7.org/linux/man-pages/man7/udev.7.html)

### ¿Qué hace udev?

El demonio `udev` (`systemd-udevd.service`) recibe eventos de dispositivo directamente del kernel cuando un dispositivo se conecta o desconecta del sistema, o cuando cambia de estado. Al recibir el evento, lo compara contra su conjunto de reglas configuradas para identificar el dispositivo y decidir qué acción tomar.

`udev` puede, entre otras cosas:

- Crear o eliminar nodos de dispositivo en `/dev`
- Asignar nombres estables y significativos a los dispositivos
- Establecer permisos, propietario y grupo de los nodos
- Ejecutar programas externos al conectar o desconectar un dispositivo
- Renombrar interfaces de red

### Nomenclatura estándar de dispositivos en `/dev`

| Patrón | Tipo de dispositivo |
|---|---|
| `/dev/sdXY` | Disco SATA/SCSI/USB. `X` = letra identificadora del disco (`a`, `b`…), `Y` = número de partición |
| `/dev/nvme0n1pN` | Disco NVMe. `0` = controladora, `1` = namespace, `N` = partición |
| `/dev/sr0` | Unidad óptica (CD/DVD) |
| `/dev/ttyS0` | Puerto serie |
| `/dev/ethX` | Interfaz de red (nomenclatura clásica; las distros modernas usan nombres predecibles como `enp3s0`) |

El acceso a los dispositivos puede ser en **modo bloque** (transferencias en bloques, normalmente 512 bytes o 4 KiB, típico en discos) o en **modo carácter** (flujo de bytes, típico en terminales y puertos serie). Esta distinción se puede ver en la salida de `ls -l /dev`: los dispositivos de bloque muestran `b` y los de carácter muestran `c` en el primer carácter de los permisos.

### Ficheros y directorios clave de udev

| Ruta | Descripción |
|---|---|
| `/etc/udev/rules.d/` | Reglas personalizadas del administrador (alta prioridad) |
| `/usr/lib/udev/rules.d/` | Reglas del sistema proporcionadas por los paquetes |
| `/etc/udev/udev.conf` | Fichero de configuración principal de udev |
| `/sys/` | Sistema de ficheros virtual del kernel (`sysfs`) que expone información de todos los dispositivos conocidos. udev lo consulta para obtener atributos de los dispositivos |

Los ficheros de reglas se procesan en orden alfanumérico. Las reglas en ficheros con número mayor modifican o sobreescriben a las reglas con número menor. Por eso los ficheros de reglas comienzan con un número de dos cifras (p. ej. `70-persistent-net.rules`).

### Sintaxis de las reglas udev

Cada regla es una línea con pares **clave-valor** separados por comas. Referencia completa: [Writing udev rules](https://www.reactivated.net/writing_udev_rules.html)

Toda regla debe contener al menos una **clave de coincidencia** (para identificar el dispositivo) y una **clave de asignación** (para indicar la acción):

```
# Ejemplo: dar un nombre fijo a un disco USB con un número de serie concreto
ACTION=="add", SUBSYSTEM=="block", ATTRS{serial}=="A1B2C3D4", SYMLINK+="usb_backup"

# Ejemplo: ejecutar un script al conectar cualquier dispositivo USB de almacenamiento
ACTION=="add", SUBSYSTEM=="usb", RUN+="/usr/local/bin/notificar_usb.sh"
```

Claves de coincidencia más habituales:

| Clave | Descripción |
|---|---|
| `ACTION` | Acción del evento: `add`, `remove`, `change` |
| `SUBSYSTEM` | Subsistema del kernel al que pertenece el dispositivo (`block`, `net`, `usb`…) |
| `KERNEL` | Nombre asignado por el kernel al dispositivo |
| `ATTR{nombre}` | Atributo del dispositivo obtenido desde `sysfs` |
| `ATTRS{nombre}` | Igual que `ATTR` pero busca en el dispositivo y en sus padres |

Claves de asignación más habituales:

| Clave | Descripción |
|---|---|
| `NAME` | Nombre del nodo de dispositivo en `/dev` |
| `SYMLINK` | Crea un enlace simbólico adicional en `/dev` |
| `OWNER`, `GROUP`, `MODE` | Propietario, grupo y permisos del nodo |
| `RUN` | Programa o script a ejecutar cuando se cumple la regla |

### Herramienta `udevadm`

`udevadm` es la utilidad de administración de `udev`. Referencia: [`man 8 udevadm`](https://man7.org/linux/man-pages/man8/udevadm.8.html)

```bash
udevadm info /dev/sdb                        # Muestra todos los atributos de un dispositivo
udevadm monitor                              # Monitoriza eventos de dispositivo en tiempo real
udevadm test /sys/class/block/sdb            # Simula el procesamiento de reglas para un dispositivo
udevadm control --reload-rules              # Recarga las reglas sin reiniciar
udevadm control --reload-rules && udevadm trigger  # Recarga y aplica las reglas a los dispositivos existentes
```

> **Nota:** `udev` está integrado en `systemd` desde la versión 183. El demonio se gestiona como una unidad de systemd: `systemctl status systemd-udevd`.

## Ejercicio: Creación de un sistema de ficheros en un nuevo disco
`mkfs.ext3 -m 1 /dev/sdb`
`mkdir -p /disco2`
`mount -t /dev/sdb /disco2`
`vi /etc/fstab`

# 4. Gestión avanzada de discos

---

## Particiones y tablas de particiones

Una **partición** es un agrupamiento de sectores contiguos en un disco duro. Sobre cada partición se genera un sistema de ficheros independiente, de forma que el sistema las ve como dispositivos distintos en `/dev`, aunque correspondan al mismo dispositivo físico. Al estar formadas por sectores contiguos, es importante planificar con antelación el espacio asignado a cada una.

Las particiones se describen en una estructura de datos llamada **tabla de particiones**. Existen tres tipos:

| Tipo | Descripción |
|---|---|
| **MBR** (*Master Boot Record*) | El más antiguo (desde 1983). Soporta discos de hasta 2 TiB y un máximo de 4 particiones primarias. Para tener más, una de ellas debe ser una *partición extendida* que contenga particiones lógicas. Se usa con firmware BIOS |
| **APM** (*Apple Partition Map*) | Propio de los Macintosh anteriores a la transición a Intel (2006). Prácticamente obsoleto |
| **GPT** (*GUID Partition Table*) | El estándar moderno, parte de la especificación UEFI. Soporta discos de hasta 2,2 ZiB, hasta 128 particiones primarias por defecto y almacena una cabecera de respaldo al final del disco para recuperarse ante corrupción. Se usa con firmware UEFI. Referencia: [Red Hat — Disk Partitions Overview](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/installation_guide/appe-disk-partitions-overview) |

> **Recomendación:** Para sistemas nuevos con UEFI se debe usar GPT. MBR solo es necesario en hardware antiguo con BIOS.

---

## Herramientas para gestionar particiones

### `parted` / libparted

[libparted](https://www.gnu.org/software/parted/) es el proyecto de software libre que desarrolla herramientas para manipular particiones y crear sistemas de ficheros sobre ellas. La herramienta de línea de comandos incluida es `parted`, que soporta tanto MBR como GPT y permite crear, eliminar y redimensionar particiones.

```bash
parted /dev/sdb print          # Muestra la tabla de particiones
parted /dev/sdb mklabel gpt    # Crea una tabla de particiones GPT
parted /dev/sdb mkpart primary ext4 1MiB 50GiB   # Crea una partición
```

### `fdisk` y familia

`fdisk` es la herramienta clásica para particionado en modo texto. Forma parte del paquete `util-linux`. Tiene variantes:

| Herramienta | Descripción |
|---|---|
| `fdisk` | Interfaz interactiva en modo texto. Soporta MBR y GPT (desde versiones recientes) |
| `cfdisk` | Interfaz de menú mejorada basada en ncurses |
| `sfdisk` | Versión orientada a scripting: lee y escribe tablas de particiones en formato de texto |

```bash
fdisk -lu /dev/sda             # Lista la tabla de particiones con tamaños en sectores
lsblk                          # Vista en árbol de todos los dispositivos de bloque y sus particiones
```

### GPT fdisk

[GPT fdisk](https://www.rodsbooks.com/gdisk/) es una suite específica para tablas GPT. Incluye:

| Herramienta | Descripción |
|---|---|
| `gdisk` | Equivalente a `fdisk` para GPT |
| `cgdisk` | Equivalente a `cfdisk`, con interfaz ncurses |
| `sgdisk` | Equivalente a `sfdisk`, orientado a scripting |

---

## Configuración de RAID

En una configuración RAID (*Redundant Array of Independent Disks*), varios discos se combinan para mejorar el rendimiento, la fiabilidad o ambas. El principio consiste en usar parte del espacio en disco para almacenar información de control o copias de los datos que permitan recuperarlos ante un fallo hardware.

### Niveles de RAID

Los niveles más habituales son:

| Nivel | Funcionamiento | Redundancia | Discos mínimos |
|---|---|---|---|
| **RAID 0** | *Striping*: los datos se distribuyen en franjas entre todos los discos. Mejora el rendimiento, pero sin redundancia. Si falla un disco, se pierden todos los datos | No | 2 |
| **RAID 1** | *Mirroring*: los datos se replican íntegramente en cada disco. Si falla uno, el sistema sigue funcionando con el resto | Sí | 2 |
| **RAID 5** | *Striping* con paridad distribuida entre los discos. Tolera el fallo de un disco. Requiere al menos 3 discos | Sí (1 disco) | 3 |
| **RAID 6** | Como RAID 5 pero con doble paridad. Tolera el fallo simultáneo de dos discos | Sí (2 discos) | 4 |
| **RAID 10** | Combinación de RAID 1 y RAID 0: espejo de conjuntos en *stripe*. Alto rendimiento y redundancia | Sí | 4 |

> **Nota importante: RAID no es una copia de seguridad.** Protege frente a fallos de hardware, pero no frente a borrados accidentales, corrupción de datos o desastres. Siempre deben mantenerse backups independientes.

### RAID por software vs. hardware

RAID puede implementarse por **hardware** (en la controladora de discos) o por **software** (en el kernel del sistema operativo). La implementación hardware es más eficiente en general, aunque suele imponer restricciones sobre el tipo de discos a usar.

En Linux, la implementación software de RAID se realiza a nivel de kernel mediante el subsistema **mdraid**. Las particiones destinadas a un array se marcan con un código especial (`0xFD` en MBR, `Linux RAID` en GPT) y el kernel las combina para crear nuevos dispositivos de bloque con nombres de la forma `/dev/md0`, `/dev/md1`, etc.

Consideraciones importantes:

- Todas las particiones de un mismo array deben tener el **mismo tamaño**.
- Es más seguro construir el array con **discos completos** que con particiones: si un disco con múltiples particiones RAID falla, se pierden todos los arrays en los que participa a la vez.
- Si se usan particiones de un mismo disco en distintos arrays, en la práctica se obtiene poca redundancia real ante fallos del disco completo.
- Los gestores de arranque no siempre pueden leer desde arrays RAID, por lo que se recomienda crear una **partición `/boot` fuera del array**.
- Si el kernel es personalizado, hay que asegurarse de incluir el soporte RAID necesario.

### `mdadm`: gestión de arrays RAID por software

La herramienta principal para gestionar arrays RAID por software en Linux es `mdadm` (*Multiple Device ADMinistration*). Referencia: [`man 8 mdadm`](https://linux.die.net/man/8/mdadm)

```bash
# Crear un array RAID 5 con tres dispositivos
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1

# Añadir un disco de repuesto (hot spare)
mdadm --create /dev/md1 --level=5 --raid-devices=3 --spare-devices=1 /dev/sd[bcde]1

# Consultar el estado del array
cat /proc/mdstat
mdadm --detail /dev/md0

# Guardar la configuración para que el array se ensamble automáticamente en el arranque
mdadm --examine --scan | tee -a /etc/mdadm/mdadm.conf
```

Una vez creado, un array RAID se trata como cualquier otro dispositivo de bloque: se puede formatear con `mkfs`, particionarlo con `fdisk` o añadirlo a `/etc/fstab`.

---

## Trabajo con volúmenes lógicos (LVM)

Las particiones son estructuras inflexibles: al estar formadas por sectores contiguos del disco, resulta complejo reorganizar el almacenamiento cuando cambian las necesidades. **LVM** (*Logical Volume Manager*) es la solución estándar en Linux para gestionar el almacenamiento de forma flexible y dinámica.

### Conceptos fundamentales

LVM se organiza en tres capas de abstracción:

```
Discos físicos / Particiones
        ↓
  Volúmenes Físicos (PV)
        ↓
  Grupos de Volúmenes (VG)
        ↓
  Volúmenes Lógicos (LV)
        ↓
  Sistema de ficheros (mkfs)
```

| Capa | Descripción |
|---|---|
| **Volumen Físico (PV)** | Disco completo o partición preparada para su uso por LVM. Se inicializa con `pvcreate` |
| **Grupo de Volúmenes (VG)** | Conjunto de uno o más PVs tratados como un único espacio de almacenamiento. Es el "pool" del que se extraen los volúmenes lógicos |
| **Volumen Lógico (LV)** | Partición abstracta creada sobre un VG. No referencia sectores físicos concretos, sino que el VG los asigna dinámicamente. Se comporta como un dispositivo de bloque estándar en `/dev/<vg>/<lv>` |

### Flujo de trabajo básico

```bash
# 1. Crear volúmenes físicos
pvcreate /dev/sdb /dev/sdc

# 2. Crear un grupo de volúmenes uniendo los PVs
vgcreate datos_vg /dev/sdb /dev/sdc

# 3. Crear volúmenes lógicos sobre el VG
lvcreate -L 50G -n apps_lv datos_vg          # LV de tamaño fijo
lvcreate -l 50%FREE -n logs_lv datos_vg       # LV usando un % del espacio libre

# 4. Formatear y montar el LV como cualquier otro dispositivo
mkfs.ext4 /dev/datos_vg/apps_lv
mount /dev/datos_vg/apps_lv /mnt/apps
```

### Ampliación de almacenamiento en caliente

Una de las principales ventajas de LVM es poder redimensionar volúmenes sin reiniciar el sistema:

```bash
# Añadir un disco nuevo al grupo de volúmenes existente
pvcreate /dev/sdd
vgextend datos_vg /dev/sdd

# Ampliar el volumen lógico y el sistema de ficheros
lvextend -L +20G /dev/datos_vg/apps_lv
resize2fs /dev/datos_vg/apps_lv            # Para Ext4
xfs_growfs /mnt/apps                       # Para XFS (se opera sobre el punto de montaje)
```

### Comandos de administración LVM

Las herramientas siguen una nomenclatura consistente: prefijo `pv` para volúmenes físicos, `vg` para grupos de volúmenes y `lv` para volúmenes lógicos.

**Volúmenes físicos:**

| Comando | Función |
|---|---|
| `pvcreate` | Inicializa un disco o partición como volumen físico |
| `pvdisplay` / `pvs` | Muestra información detallada / resumida de los PVs |
| `pvmove` | Mueve los extents de datos de un PV a otro |
| `pvremove` | Elimina la etiqueta LVM de un volumen físico |
| `pvresize` | Ajusta el tamaño de un PV (tras redimensionar el disco subyacente) |
| `pvscan` | Escanea todos los discos en busca de volúmenes físicos |
| `pvchange` / `pvck` | Cambia atributos / verifica la integridad de un PV |

**Grupos de volúmenes:**

| Comando | Función |
|---|---|
| `vgcreate` | Crea un nuevo grupo de volúmenes |
| `vgdisplay` / `vgs` | Muestra información detallada / resumida de los VGs |
| `vgextend` | Añade un PV a un VG existente |
| `vgreduce` | Elimina un PV de un VG |
| `vgmerge` / `vgsplit` | Fusiona o divide grupos de volúmenes |
| `vgremove` | Elimina un grupo de volúmenes |
| `vgscan` / `vgck` | Escanea / verifica la integridad de los VGs |
| `vgchange` | Activa o desactiva un VG y modifica sus atributos |

**Volúmenes lógicos:**

| Comando | Función |
|---|---|
| `lvcreate` | Crea un nuevo volumen lógico |
| `lvdisplay` / `lvs` | Muestra información detallada / resumida de los LVs |
| `lvextend` | Aumenta el tamaño de un LV |
| `lvreduce` | Reduce el tamaño de un LV (requiere desmontar el FS primero en la mayoría de casos) |
| `lvresize` | Cambia el tamaño de un LV (puede ampliar o reducir) |
| `lvremove` | Elimina un volumen lógico |
| `lvscan` | Escanea todos los VGs en busca de volúmenes lógicos |
| `lvchange` | Activa / desactiva un LV o modifica sus atributos |

### Herramientas gráficas y complementos

- [system-config-lvm](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/storage_administration_guide/s1-system-config-lvm): herramienta gráfica para gestionar LVM en Red Hat/Fedora (en desuso en versiones modernas).
- [KDE Volume and Partition Manager (kvpm)](https://sourceforge.net/projects/kvpm/): interfaz gráfica para LVM y GNU parted.
- [Enterprise Volume Management System (EVMS)](https://evms.sourceforge.net/): sistema unificado para gestionar RAID, LVM y múltiples sistemas de ficheros (proyecto discontinuado).

> **LVM sobre RAID:** Es habitual combinar ambas tecnologías, creando primero el array RAID (para obtener redundancia) y después configurando LVM sobre él (para obtener flexibilidad). En este esquema, el array `/dev/md0` se convierte en el volumen físico de LVM.

---

## Puesta a punto del acceso a discos

### Tecnologías de interfaz y drivers

Desde el punto de vista de la administración, todos los discos utilizan el mismo tipo de interfaz con Linux a través de sus drivers. Los drivers están escritos para la **controladora de discos** de la placa base o para la tarjeta controladora, no para el disco en sí.

| Driver | Uso | Nomenclatura en `/dev` |
|---|---|---|
| **PATA** (IDE) | Discos IDE/PATA, prácticamente obsoletos. Se mantienen en el kernel por compatibilidad | `/dev/hda`, `/dev/hdb`… |
| **SCSI/SATA/SAS/USB** | Subsistema unificado usado por discos SATA, SAS, USB y también discos virtuales en máquinas virtuales | `/dev/sda`, `/dev/sdb`… |
| **NVMe** | Discos NVMe conectados directamente al bus PCIe. Alto rendimiento | `/dev/nvme0n1`, `/dev/nvme1n1`… |

### Interrupciones y DMA

Como cualquier dispositivo hardware, los discos tienen asociada una [interrupción (IRQ)](https://es.wikipedia.org/wiki/Interrupci%C3%B3n) y, en los casos que aplica, un canal de [acceso directo a memoria (DMA)](https://es.wikipedia.org/wiki/Acceso_directo_a_memoria):

```bash
cat /proc/interrupts    # Muestra la asignación actual de IRQs por dispositivo
cat /proc/dma           # Muestra la asignación de canales DMA (relevante en hardware antiguo ISA/PATA)
sysctl -a               # Lista todos los parámetros del kernel ajustables en tiempo de ejecución
```

La configuración de parámetros del kernel se puede ajustar permanentemente en `/etc/sysctl.conf` o en ficheros bajo `/etc/sysctl.d/`.

---

### Prueba de rendimiento del disco

#### `hdparm`

`hdparm` es la herramienta clásica para obtener parámetros de rendimiento de discos IDE/SATA y compararlos con las especificaciones del fabricante. Referencia: [`man 8 hdparm`](https://man7.org/linux/man-pages/man8/hdparm.8.html)

```bash
hdparm -tT /dev/sda          # Test de velocidad de lectura (caché y disco)
hdparm -tT --direct /dev/sda # Test evitando la caché del sistema operativo (más realista)
hdparm -I /dev/sda           # Muestra información detallada del disco (modelo, funcionalidades)
```

Para discos con driver PATA, `hdparm` también permite ajustar parámetros de acceso como el modo de transferencia DMA.

> **Nota:** Para discos NVMe existe la herramienta equivalente `nvme-cli` con el comando `nvme` (`nvme smart-log /dev/nvme0`).

---

### Monitorización del disco con SMART

[S.M.A.R.T.](https://es.wikipedia.org/wiki/S.M.A.R.T.) (*Self-Monitoring, Analysis and Reporting Technology*) es una tecnología integrada en la mayoría de discos duros y SSD que permite monitorizar su estado interno y anticipar fallos hardware antes de que se produzcan.

La herramienta estándar en Linux es el paquete **`smartmontools`**, que incluye dos componentes:

| Componente | Función |
|---|---|
| `smartctl` | Utilidad de línea de comandos para consultar atributos SMART, ejecutar autodiagnósticos y ver los registros del disco |
| `smartd` | Demonio de monitorización continua en segundo plano. Puede enviar alertas por correo cuando detecta problemas |

Comandos habituales de `smartctl`:

```bash
smartctl -i /dev/sda             # Información básica del disco (modelo, serie, soporte SMART)
smartctl -H /dev/sda             # Comprobación rápida de salud: PASSED / FAILED
smartctl -a /dev/sda             # Muestra todos los atributos SMART y el historial de errores
smartctl -t short /dev/sda       # Inicia un autodiagnóstico corto (1-2 minutos)
smartctl -t long /dev/sda        # Inicia un autodiagnóstico completo (puede durar horas)
smartctl -l selftest /dev/sda    # Muestra el historial de autodiagnósticos anteriores
```

Atributos SMART críticos a vigilar (valores distintos de 0 son señal de alerta):

| ID | Nombre | Significado |
|---|---|---|
| 5 | `Reallocated_Sector_Ct` | Sectores reasignados por errores de lectura/escritura |
| 187 | `Reported_Uncorrect` | Errores no corregibles |
| 197 | `Current_Pending_Sector` | Sectores pendientes de reasignación |
| 198 | `Offline_Uncorrectable` | Sectores inaccesibles en tests offline |

> **SmartMonTools** tiene una versión con interfaz gráfica llamada **GSmartControl**, disponible en los repositorios de la mayoría de distribuciones.

---

## Backup y restauración del disco

### Estrategia de almacenamiento

Antes de elegir una herramienta, debe definirse el medio donde se almacenarán los backups:

| Medio | Descripción |
|---|---|
| **Cinta magnética** | El estándar tradicional en entornos empresariales. Alta capacidad y durabilidad, pero acceso secuencial |
| **Disco (interno/externo)** | El medio más habitual hoy en día. Permite acceso aleatorio y backups más rápidos |
| **Almacenamiento en red / nube** | Los datos se envían a través de la red a ubicaciones externas. Protege ante desastres físicos locales |

### Dispositivos de cinta en Linux

<cite index="85-1">El dispositivo de cinta con rebobinado automático (*rewind*) en Linux es `/dev/st0`, y la variante sin rebobinado es `/dev/nst0`. La variante sin rebobinado es útil en scripts para evitar rebobinados inesperados o para anexar datos a continuación de un backup anterior.</cite> Los discos IDE de cinta se asignan a `/dev/ht0`.

Opciones adicionales de denominación:

| Dispositivo | Comportamiento |
|---|---|
| `/dev/st0` | SCSI/SATA, rebobina al cerrar |
| `/dev/nst0` | SCSI/SATA, **sin** rebobinado al cerrar |
| `/dev/st0l` / `/dev/st0m` | Baja / media compresión hardware |
| `/dev/ht0` | IDE/PATA, rebobina al cerrar |

### Herramientas de backup

#### `tar` — Archivado y compresión

La herramienta más universal para crear archivos comprimidos de directorios y ficheros.

```bash
tar -czf backup.tar.gz /home/usuario        # Crea un archivo comprimido con gzip
tar -cjf backup.tar.bz2 /home/usuario       # Compresión con bzip2 (mayor ratio)
tar -cJf backup.tar.xz /home/usuario        # Compresión con xz (mayor ratio aún)
tar -xzf backup.tar.gz -C /destino          # Extrae en un directorio específico
tar -tzf backup.tar.gz                      # Lista el contenido sin extraer
```

Backup directo a cinta con `tar`:

```bash
tar -czf /dev/st0 /directorio_origen        # Backup a cinta con compresión
tar -tzf /dev/st0                           # Lista el contenido de la cinta
tar -xzf /dev/st0                           # Restaura desde cinta
```

#### `rsync` — Backups incrementales en red

Herramienta eficiente para backups incrementales: solo transfiere los ficheros que hayan cambiado desde el último backup.

```bash
rsync -avz /origen/ usuario@servidor:/destino/     # Backup remoto por SSH
rsync -avz --delete /origen/ /destino/             # Sincronización (borra lo que no existe en origen)
rsync -avz --link-dest=/backup/anterior /origen/ /backup/nuevo/   # Backup incremental con hardlinks
```

#### `dd` — Copia a nivel de bloque

Copia datos byte a byte, incluyendo el espacio vacío. Útil para clonar discos o particiones completas o crear imágenes de dispositivos.

```bash
dd if=/dev/sda of=/dev/sdb bs=4M status=progress   # Clona disco completo
dd if=/dev/sda1 of=/ruta/imagen.img bs=4M           # Crea imagen de una partición
dd if=/dev/zero of=/dev/sdb bs=4M                   # Borra un disco completamente
```

> **Advertencia:** `dd` opera directamente sobre los bloques del dispositivo sin entender el sistema de ficheros. Las imágenes creadas con `dd` solo pueden restaurarse en un destino de igual o mayor tamaño que el original.

#### `cpio` — Archivado con `find`

Herramienta de archivado que trabaja bien en combinación con `find` para seleccionar exactamente qué ficheros incluir:

```bash
find /etc -depth -print | cpio -ovH crc -O /dev/st0    # Backup de /etc a cinta
find /home -depth -print | cpio -ovH crc | gzip > backup.cpio.gz  # Backup comprimido
```

#### `mt` — Control de cintas magnéticas

`mt` (*Magnetic Tape*) es la herramienta para controlar la unidad de cinta: rebobinar, posicionarse en una marca, verificar el estado, etc. Referencia: [`man 1 mt`](https://man7.org/linux/man-pages/man1/mt.1.html)

```bash
mt -f /dev/st0 status      # Estado de la unidad de cinta
mt -f /dev/st0 rewind      # Rebobina la cinta al principio
mt -f /dev/st0 erase       # Borra la cinta
mt -f /dev/nst0 eod        # Avanza hasta el final de los datos (para anexar)
mt -f /dev/nst0 fsf 1      # Avanza una marca de fichero (para leer el siguiente backup en cinta)
```

### Resumen de herramientas de backup

| Herramienta | Uso principal |
|---|---|
| `tar` | Archivado y compresión de directorios. La opción más habitual en la práctica |
| `rsync` | Backups incrementales y sincronización, especialmente en red |
| `dd` | Clonado de discos/particiones a nivel de bloque |
| `cpio` | Archivado selectivo de ficheros, combinado con `find` |
| `mt` | Control de unidades de cinta magnética |

# 5. Configuración de red

---

## Conceptos previos: nomenclatura de interfaces

Las versiones modernas del kernel de Linux usan **nombres predecibles de interfaz** en lugar de los clásicos `eth0`, `wlan0`, etc. Estos nombres están basados en la topología física del hardware (p. ej. `enp3s0` para una tarjeta Ethernet en el bus PCI 3, slot 0; `wlp2s0` para WiFi). El objetivo es que el nombre no cambie entre reinicios aunque se añada o elimine hardware.

|Prefijo|Tipo de interfaz|
|---|---|
|`en`|Ethernet (cableada)|
|`wl`|Wireless LAN (WiFi)|
|`ww`|Wireless WAN (móvil)|
|`lo`|Loopback|

Para listar todas las interfaces disponibles:

bash

```bash
ip link show          # Muestra todas las interfaces y su estado
ip -br link show      # Vista compacta con colores
```

---

## Gestores de red modernos

En las distribuciones actuales la configuración de red no se gestiona directamente con `ifconfig` sino mediante uno de estos sistemas:

|Herramienta|Uso típico|Distros|
|---|---|---|
|**NetworkManager**|Entornos de escritorio y portátiles. Gestiona conexiones WiFi, VPN y perfiles de forma dinámica|Fedora, RHEL, Ubuntu Desktop, Debian Desktop|
|**systemd-networkd**|Servidores y entornos con configuración estática. Ligero y sin dependencias extra|Ubuntu Server, Arch, sistemas embebidos|
|**Netplan**|Capa de abstracción YAML por encima de NetworkManager o systemd-networkd|Ubuntu 17.10+|

> **Importante:** Los cambios realizados con `ip` o `ifconfig` directamente sobre las interfaces son **temporales** y se pierden al reiniciar. Para hacerlos permanentes hay que configurarlos a través del gestor de red de la distribución.

### NetworkManager desde la línea de comandos: `nmcli`

bash

```bash
nmcli device status                          # Estado de todas las interfaces
nmcli connection show                        # Lista las conexiones configuradas
nmcli connection up <nombre>                 # Activa una conexión
nmcli connection down <nombre>               # Desactiva una conexión
```

### Ficheros de configuración por distribución

|Distro|Fichero / ruta de configuración|
|---|---|
|RHEL / CentOS / Fedora|`/etc/NetworkManager/system-connections/` (formato keyfile)|
|Ubuntu / Debian moderno|`/etc/netplan/*.yaml`|
|Debian clásico|`/etc/network/interfaces`|

---

## DHCP y configuración de dirección IP

Linux puede actuar como **cliente** o **servidor** DHCP. Los clientes DHCP más comunes son:

|Cliente|Descripción|
|---|---|
|`dhclient`|El más extendido, parte del paquete `isc-dhcp-client`. Incluido por defecto en Debian/Ubuntu|
|`dhcpcd`|Ligero y rápido, habitual en Arch, Alpine y sistemas embebidos|
|`pump`|Antiguo cliente DHCP, prácticamente en desuso en distribuciones modernas|

Para solicitar una configuración IP desde el servidor DHCP de forma manual:

bash

```bash
dhclient enp3s0              # Solicita dirección IP para la interfaz indicada
dhclient -r enp3s0           # Libera la concesión DHCP actual
```

### Herramientas modernas: suite `iproute2`

`ifconfig` y las herramientas del paquete `net-tools` (`route`, `arp`, `netstat`) están **obsoletas y sin mantenimiento activo** en las distribuciones modernas. Su sustituta es la suite **`iproute2`**, cuya herramienta principal es el comando `ip`.

Equivalencias entre herramientas antiguas y modernas:

|Herramienta antigua (net-tools)|Equivalente moderno (iproute2)|
|---|---|
|`ifconfig`|`ip addr`, `ip link`|
|`route`|`ip route`|
|`arp`|`ip neigh`|
|`netstat`|`ss`|
|`iwconfig`|`iw`|

---

## Comando `ip` (iproute2)

El comando `ip` unifica la gestión de interfaces, direcciones, rutas, vecinos ARP y más. Referencia: [`man 8 ip`](https://man7.org/linux/man-pages/man8/ip.8.html)

Sintaxis general:

```
ip [OPCIONES] OBJETO {COMANDO [ARGUMENTOS] | help}
```

Los objetos principales son:

|Objeto|Descripción|
|---|---|
|`link`|Interfaces de red (capa de enlace)|
|`addr`|Direcciones IP asignadas a interfaces|
|`route`|Tabla de enrutamiento|
|`neigh`|Tabla ARP / caché de vecinos|
|`netns`|Espacios de nombres de red|

Cada objeto puede escribirse abreviado (`ip a` en lugar de `ip addr show`).

### Gestión de interfaces (`ip link`)

bash

```bash
ip link show                             # Lista todas las interfaces
ip link set enp3s0 up                    # Activa una interfaz
ip link set enp3s0 down                  # Desactiva una interfaz
ip link set enp3s0 mtu 9000              # Cambia el MTU (jumbo frames)
```

### Gestión de direcciones IP (`ip addr`)

bash

```bash
ip addr show                             # Muestra todas las IPs asignadas
ip addr show enp3s0                      # Solo para una interfaz concreta
ip addr add 192.168.1.100/24 dev enp3s0  # Asigna una IP (temporal)
ip addr del 192.168.1.100/24 dev enp3s0  # Elimina una IP
```

### Gestión de rutas (`ip route`)

bash

```bash
ip route show                            # Muestra la tabla de rutas
ip route add default via 192.168.1.1     # Añade la puerta de enlace predeterminada
ip route add 10.0.0.0/8 via 192.168.1.254 dev enp3s0  # Ruta estática
ip route del default                     # Elimina la ruta por defecto
ip route get 8.8.8.8                     # Muestra qué ruta usaría el kernel para una IP
```

### Tabla ARP (`ip neigh`)

bash

```bash
ip neigh show                            # Muestra la caché ARP
ip neigh flush all                       # Vacía la caché ARP
```

---

## El fichero `/etc/hosts`

En redes pequeñas o para resolución local, el fichero `/etc/hosts` permite asociar nombres de host a direcciones IP sin necesidad de DNS. Las aplicaciones consultan este fichero antes de realizar una consulta DNS (según la configuración de `/etc/nsswitch.conf`).

Formato:

```
<dirección_IP>   <nombre_host>   [alias...]
127.0.0.1        localhost
192.168.1.10     servidor01   servidor01.local
```

Para consultar o establecer el nombre del propio equipo:

bash

```bash
hostname                   # Muestra el nombre actual
hostname nuevo_nombre      # Lo cambia temporalmente
hostnamectl set-hostname nuevo_nombre   # Cambio permanente (systemd)
```

El nombre de host permanente se almacena en `/etc/hostname`.

---

## Configuración de WiFi

### Herramientas de bajo nivel

|Herramienta|Estado|Descripción|
|---|---|---|
|`iwconfig` / `iwlist`|**Obsoleta** (paquete `wireless-tools`)|Solo soporta WEP. No puede gestionar WPA/WPA2/WPA3 ni redes 5 GHz de forma fiable|
|`iw`|**Actual**|Herramienta moderna basada en el protocolo `nl80211`. Reemplaza a `iwconfig` y soporta todos los estándares 802.11 actuales|

Comandos habituales con `iw`:

bash

```bash
iw dev                              # Lista las interfaces WiFi disponibles
iw dev wlan0 scan                   # Escanea las redes disponibles
iw dev wlan0 link                   # Muestra el estado de la conexión actual
iw dev wlan0 connect "MiRedWiFi"    # Conecta a una red abierta (sin contraseña)
```

> **Nota:** El nombre correcto del comando de escaneo en tus apuntes originales era `iwlist wlan0 scan` (con `l`, no con `wilist`). En sistemas modernos el equivalente es `iw dev wlan0 scan`.

### Para redes cifradas (WPA/WPA2/WPA3): `wpa_supplicant`

`iw` no gestiona cifrado. Para redes protegidas se usa `wpa_supplicant` junto con `dhclient`:

bash

```bash
# Generar el fichero de configuración con la contraseña cifrada
wpa_passphrase "MiRedWiFi" "mi_contraseña" | sudo tee /etc/wpa_supplicant.conf

# Lanzar wpa_supplicant en segundo plano
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf

# Obtener IP por DHCP
sudo dhclient wlan0
```

### Gestión de alto nivel: `nmcli`

En la práctica, la forma más cómoda de gestionar WiFi desde línea de comandos es `nmcli` (NetworkManager):

bash

```bash
nmcli dev wifi list                                      # Lista redes disponibles
nmcli dev wifi connect "MiRedWiFi" password "mi_clave"   # Conecta a una red WPA2/WPA3
nmcli dev wifi connect "MiRedWiFi" password "mi_clave" ifname wlan0  # Especificando interfaz
```

### Herramientas gráficas

Para escritorio, los gestores de red con interfaz gráfica más comunes son **GNOME Network Manager** (integrado en el panel de GNOME), **nm-applet** (bandeja del sistema) y **plasma-nm** (KDE). La herramienta `wicd` mencionada en algunos materiales lleva años sin mantenimiento y se considera obsoleta.

---

## Configurar Linux como rúter

Si el equipo tiene varias interfaces de red, puede actuar como rúter entre las redes a las que está conectado. Para ello hay que habilitar el **reenvío de paquetes IP** (_IP forwarding_):

**Activación temporal (hasta el próximo reinicio):**

bash

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
# O equivalentemente:
sysctl -w net.ipv4.ip_forward=1
```

**Activación permanente:**

Editar (o crear) un fichero en `/etc/sysctl.d/` y añadir:

```
net.ipv4.ip_forward = 1
```

Aplicar sin reiniciar:

bash

```bash
sysctl -p /etc/sysctl.d/99-ip_forward.conf
```

> **Nota:** En versiones modernas de las distribuciones se prefiere crear un fichero propio en `/etc/sysctl.d/` en lugar de editar directamente `/etc/sysctl.conf`, para no mezclar configuración del sistema con la del administrador.

Después hay que añadir las rutas necesarias con `ip route` y, si se necesita NAT (enmascaramiento de la red interna), configurar `iptables` o `nftables`.

---

## Monitorización del tráfico de red

### Comandos de diagnóstico rápido

|Comando|Función|
|---|---|
|`ping <host>`|Comprueba conectividad básica enviando paquetes ICMP|
|`traceroute <host>`|Muestra la ruta de saltos hasta un destino|
|`mtr <host>`|Combina `ping` y `traceroute` en tiempo real: la herramienta más útil para diagnóstico de ruta|
|`ip route get <IP>`|Muestra qué ruta usaría el kernel para alcanzar una IP|
|`ip neigh`|Muestra la tabla ARP (equivalente a `arp -n`)|
|`dig <dominio>`|Consulta DNS detallada (reemplaza a `nslookup`)|

---

### Comando `ss` (socket statistics)

`ss` es el sustituto moderno y activamente mantenido de `netstat`. Pertenece a la suite `iproute2`, consulta directamente el kernel (más rápido y preciso que `netstat`) y ofrece soporte completo de IPv6. Referencia: [`man 8 ss`](https://man7.org/linux/man-pages/man8/ss.8.html)

bash

```bash
ss -tuln                   # Puertos TCP y UDP en escucha (sin resolver nombres)
ss -tulpn                  # Ídem, mostrando el proceso que los ocupa
ss -s                      # Estadísticas resumidas de todos los sockets
ss -tnp state established  # Conexiones TCP establecidas con el proceso
ss -4 -tnp                 # Solo IPv4
```

> **Nota:** `netstat` pertenece al paquete `net-tools`, que lleva años sin mantenimiento activo. Se debe preferir `ss` para nuevos scripts y uso diario.

---

### Comando `netcat` (`nc`)

[Netcat](https://netcat.sourceforge.net/) es una herramienta de propósito general para crear y depurar conexiones de red TCP/UDP. Se utiliza sobre todo en scripts que necesitan trabajar con sockets.

bash

```bash
nc -zv 192.168.1.1 22          # Comprueba si el puerto 22 está accesible
nc -l 8080                     # Escucha en el puerto 8080 (servidor simple)
nc 192.168.1.1 8080            # Conecta al puerto 8080 de un host
echo "hola" | nc 192.168.1.1 9999  # Envía texto a un socket remoto
```

---

### Comando `tcpdump`

Analizador de paquetes en línea de comandos. Para capturar tráfico, pone la tarjeta de red en **modo promiscuo**: en ese modo, la capa de enlace no descarta las tramas no dirigidas a la propia MAC y el sistema puede capturar todo el tráfico que circula por el segmento de red.

bash

```bash
tcpdump -i enp3s0                         # Captura todo el tráfico de una interfaz
tcpdump -i enp3s0 port 80                 # Filtra por puerto
tcpdump -i enp3s0 host 192.168.1.10       # Filtra por host
tcpdump -i enp3s0 -w captura.pcap         # Guarda la captura en un fichero (legible por Wireshark)
tcpdump -i enp3s0 -n -v                   # Sin resolución de nombres, modo verbose
```

> **Nota:** Para capturar en modo promiscuo se requieren privilegios de root o la capability `CAP_NET_RAW`.

---

### Wireshark

[Wireshark](https://www.wireshark.org/) es el analizador de paquetes gráfico de referencia: código abierto, multiplataforma y con soporte de cientos de protocolos. Puede leer capturas generadas por `tcpdump` (formato `.pcap`).

Su versión en línea de comandos se llama **TShark** y es funcionalmente equivalente a `tcpdump` con el motor de disección de protocolos de Wireshark.

bash

```bash
tshark -i enp3s0                    # Captura en tiempo real
tshark -r captura.pcap              # Lee un fichero de captura
tshark -r captura.pcap -Y "http"    # Filtra por protocolo
```

---

### Comando `nmap`

[nmap](https://nmap.org/) es la herramienta estándar de descubrimiento de red y auditoría de seguridad. Usa paquetes IP para determinar qué hosts están activos, qué servicios y versiones ofrecen, y qué sistema operativo tienen. Referencia: [`man 1 nmap`](https://man7.org/linux/man-pages/man1/nmap.1.html)

bash

```bash
nmap 192.168.1.0/24                # Descubrimiento de hosts en una subred
nmap -sV 192.168.1.10              # Detección de versiones de servicios
nmap -O 192.168.1.10               # Detección del sistema operativo
nmap -p 22,80,443 192.168.1.10     # Escanea solo los puertos indicados
nmap -sT localhost                 # Escaneo TCP de puertos locales abiertos
```

> **Importante:** Escanear redes o equipos sin autorización es ilegal en muchas jurisdicciones. `nmap` debe usarse únicamente en redes propias o con permiso explícito.

---

## Ficheros de log de red

Los registros del sistema se almacenan habitualmente en `/var/log/`. Los más relevantes para red son:

|Fichero / ruta|Contenido|
|---|---|
|`/var/log/syslog` (Debian/Ubuntu)|Registro general del sistema, incluye eventos de red|
|`/var/log/messages` (RHEL/CentOS)|Equivalente en distribuciones Red Hat|
|`/var/log/NetworkManager`|Eventos específicos de NetworkManager|
|`journalctl -u NetworkManager`|Logs de NetworkManager vía systemd journal|
|`journalctl -u systemd-networkd`|Logs de systemd-networkd|

En sistemas con **systemd**, `journalctl` es la herramienta principal para consultar logs:

bash

```bash
journalctl -u NetworkManager -f          # Sigue los logs de NetworkManager en tiempo real
journalctl -u NetworkManager --since "1 hour ago"   # Eventos de la última hora
journalctl -k | grep -i "eth\|enp\|wlan"            # Mensajes del kernel sobre interfaces
```

Para más información sobre los ficheros de log del sistema: [The Geek Stuff — Linux /var/log files](https://www.thegeekstuff.com/2011/08/linux-var-log-files/).

# 6. Configuración de servidores DNS

---

## El sistema DNS: conceptos fundamentales

El **Sistema de Nombres de Dominio** (_Domain Name System_, DNS) es una base de datos distribuida y jerárquica que traduce nombres de dominio legibles por humanos (como `www.ejemplo.com`) en direcciones IP, y viceversa. El estándar está definido en el [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034) y el [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035).

### Roles de un servidor DNS

Cada servidor DNS puede desempeñar uno o varios de los siguientes roles:

|Tipo|Descripción|
|---|---|
|**Maestro / Primario**|Almacena los registros originales (_zona primaria_) y tiene autoridad sobre ellos. Es la fuente de verdad para esa zona y responde consultas de otros servidores DNS|
|**Esclavo / Secundario**|Tiene autoridad sobre una zona, pero obtiene sus datos del servidor maestro mediante **transferencias de zona**. Proporciona redundancia y reparte carga|
|**Solo caché**|No tiene autoridad sobre ninguna zona. Realiza resolución recursiva en nombre de los clientes y almacena las respuestas en caché durante el tiempo indicado por el campo TTL (_Time To Live_) de cada registro|
|**Solo reenvío** (_forwarder_)|No resuelve por sí mismo: reenvía todas las consultas a una lista de servidores DNS configurados. Útil en redes sin acceso directo a Internet o cuando se quiere centralizar el tráfico DNS|

> Un mismo servidor puede combinar varios roles simultáneamente: por ejemplo, ser maestro para las zonas internas y solo reenvío para las externas.

### Zonas DNS

Una **zona DNS** es una porción del espacio de nombres de dominio sobre la que un servidor tiene autoridad administrativa delegada. Dentro de una zona puede haber múltiples servidores DNS para garantizar disponibilidad.

|Tipo de zona|Descripción|
|---|---|
|**Búsqueda directa** (_forward lookup_)|Resuelve un nombre de dominio → dirección IP|
|**Búsqueda inversa** (_reverse lookup_)|Resuelve una dirección IP → nombre de dominio. Usa el dominio especial `in-addr.arpa` para IPv4 e `ip6.arpa` para IPv6|
|**Zona raíz** (_hint zone_)|Contiene las direcciones de los servidores raíz del DNS (los 13 grupos de servidores raíz), punto de partida de toda resolución recursiva|

---

## Registros de recursos DNS

Los datos de una zona se almacenan como **registros de recursos** (_Resource Records_, RR). Los más importantes son:

|Tipo|Nombre completo|Función|
|---|---|---|
|`A`|Address|Mapea un nombre de dominio a una dirección **IPv4**|
|`AAAA`|IPv6 Address|Mapea un nombre de dominio a una dirección **IPv6**|
|`CNAME`|Canonical Name|Crea un alias que apunta a otro nombre de dominio. No puede coexistir con otros registros del mismo nombre ni usarse en el vértice de zona (_apex_)|
|`MX`|Mail Exchanger|Especifica el servidor de correo responsable de un dominio. Incluye un valor de prioridad|
|`NS`|Name Server|Identifica los servidores DNS autoritativos de una zona. Siempre apunta a nombres, no a IPs|
|`PTR`|Pointer|Resolución inversa: mapea una IP a un nombre. Reside en zonas `in-addr.arpa`|
|`SOA`|Start of Authority|Registro obligatorio en toda zona. Contiene metadatos de administración: servidor primario, contacto, número de serie, TTLs de refresco y expiración|
|`TXT`|Text|Almacena texto arbitrario. Se usa para verificación de dominio, SPF, DKIM y DMARC (correo)|
|`SRV`|Service|Indica el host y puerto de un servicio específico (p. ej. SIP, XMPP)|

### Estructura del registro SOA

El registro SOA es el primero de toda zona y controla el comportamiento de las transferencias de zona:

```
ejemplo.com.  IN  SOA  ns1.ejemplo.com.  admin.ejemplo.com. (
    2024010501   ; Serial: número de versión (se incrementa en cada cambio)
    3600         ; Refresh: segundos que el esclavo espera antes de consultar al maestro
    900          ; Retry: segundos que espera el esclavo si falla el refresco
    604800       ; Expire: segundos tras los que el esclavo descarta su copia si no contacta al maestro
    300          ; Negative TTL: tiempo de caché para respuestas NXDOMAIN
)
```

> **Convención recomendada para el campo Serial:** usar el formato `YYYYMMDDNN` (año, mes, día, número de cambio del día). Por ejemplo, el tercer cambio del 5 de enero de 2024 sería `2024010503`. El esclavo solo descargará la zona si el serial del maestro es mayor que el suyo.

---

## Implementaciones de DNS en Linux

|Software|Descripción|
|---|---|
|[**BIND**](https://www.isc.org/bind/) (Berkeley Internet Name Domain)|La implementación más utilizada en Internet. Mantenida por el ISC (_Internet Systems Consortium_). Soporta todos los roles DNS, DNSSEC y es el estándar de facto en entornos empresariales|
|[**Unbound**](https://nlnetlabs.nl/projects/unbound/)|Resolvedor y caché moderno, orientado a la seguridad. Ligero y con soporte completo de DNSSEC. Muy usado como resolvedor local o en sistemas embebidos|
|[**dnsmasq**](https://thekelleys.org.uk/dnsmasq/doc.html)|Servidor ligero para redes pequeñas. Combina DNS, DHCP y TFTP en un solo proceso. Habitual en routers domésticos y entornos de laboratorio|
|[**djbdns**](https://cr.yp.to/djbdns.html)|Alternativa histórica a BIND enfocada en la seguridad. Prácticamente sin mantenimiento activo desde 2001 y en desuso en distribuciones modernas|

---

## Configuración de BIND

El paquete se llama `bind9` (Debian/Ubuntu) o `bind` (RHEL/Fedora). El proceso servidor es **`named`**. Documentación oficial: [bind9.readthedocs.io](https://bind9.readthedocs.io/)

```bash
sudo apt install bind9 bind9-utils    # Debian / Ubuntu
sudo dnf install bind bind-utils      # RHEL / Fedora / CentOS
```

### Estructura de ficheros de configuración

|Fichero / ruta|Descripción|
|---|---|
|`/etc/named.conf`|Fichero principal (RHEL/Fedora)|
|`/etc/bind/named.conf`|Fichero principal (Debian/Ubuntu)|
|`/etc/bind/named.conf.options`|Opciones globales del servidor (en Debian/Ubuntu se divide en varios ficheros)|
|`/etc/bind/named.conf.local`|Definición de zonas locales|
|`/var/named/` o `/etc/bind/`|Directorio donde se almacenan los ficheros de zona|

### Secciones principales de `named.conf`

#### Bloque `options`

Controla el comportamiento global del servidor:

```
options {
    directory "/var/cache/bind";        // Directorio de trabajo de named

    listen-on { 127.0.0.1; 192.168.1.1; };   // IPs en las que escucha (puerto 53 UDP/TCP)
    listen-on-v6 { none; };             // Deshabilitar IPv6 si no se necesita

    allow-query { 192.168.1.0/24; };    // IPs autorizadas a hacer consultas
    allow-recursion { 192.168.1.0/24; }; // IPs que pueden pedir resolución recursiva
    allow-transfer { none; };           // Deshabilitar transferencias de zona por defecto

    forwarders {
        8.8.8.8;                        // Google DNS
        1.1.1.1;                        // Cloudflare DNS
    };
    forward only;                       // No intentar resolución propia; reenviar siempre

    dnssec-validation auto;             // Validar DNSSEC automáticamente
    recursion yes;                      // Permitir consultas recursivas
};
```

Diferencia entre `forward first` y `forward only`:

|Opción|Comportamiento|
|---|---|
|`forward first`|Consulta primero a los _forwarders_. Si no responden, intenta la resolución recursiva directa (valor por defecto)|
|`forward only`|Reenvía siempre a los _forwarders_ y nunca intenta resolución propia|

#### Bloque `zone`

Define las zonas que gestiona el servidor e indica si actúa como `master` (primario) o `slave` (secundario):

```
// Zona directa — servidor primario
zone "ejemplo.com" {
    type master;
    file "/etc/bind/db.ejemplo.com";
    allow-transfer { 192.168.1.2; };    // Permitir transferencia solo al servidor esclavo
};

// Zona inversa — servidor primario
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};

// Zona directa — servidor secundario
zone "ejemplo.com" {
    type slave;
    masters { 192.168.1.1; };
    file "/var/cache/bind/db.ejemplo.com.slave";
};
```

### Ejemplo de fichero de zona directa

```
$TTL 86400
@   IN  SOA  ns1.ejemplo.com.  admin.ejemplo.com. (
        2024010501  ; Serial
        3600        ; Refresh
        900         ; Retry
        604800      ; Expire
        300 )       ; Negative TTL

; Servidores de nombres
@       IN  NS   ns1.ejemplo.com.
@       IN  NS   ns2.ejemplo.com.

; Registros A (IPv4)
ns1     IN  A    192.168.1.1
ns2     IN  A    192.168.1.2
@       IN  A    192.168.1.10
www     IN  A    192.168.1.10
mail    IN  A    192.168.1.20

; Alias
ftp     IN  CNAME  www.ejemplo.com.

; Correo
@       IN  MX   10  mail.ejemplo.com.

; Registro TXT (verificación de dominio / SPF)
@       IN  TXT  "v=spf1 mx -all"
```

### Gestión del servicio

```bash
systemctl start named          # Inicia el servidor
systemctl enable named         # Habilita el inicio automático
systemctl reload named         # Recarga la configuración sin interrumpir el servicio

rndc reload                    # Recarga zonas mediante la herramienta de control de named
rndc reload ejemplo.com        # Recarga solo una zona específica
rndc status                    # Estado del servidor named
```

> `rndc` (_Remote Name Daemon Control_) es la herramienta de administración en tiempo real de BIND, preferida a reiniciar el proceso completo para aplicar cambios en zonas.

---

## Diagnóstico de DNS

### Herramientas de validación de BIND

|Comando|Función|
|---|---|
|`named-checkconf`|Verifica la sintaxis de `/etc/named.conf` y los ficheros que incluye. **No comprueba semántica, solo sintaxis**|
|`named-checkzone <zona> <fichero>`|Verifica la sintaxis y consistencia de un fichero de zona concreto|
|`named-compilezone`|Similar a `named-checkzone` pero vuelca el contenido de la zona en un formato diferente (útil para conversión)|

```bash
named-checkconf                                          # Comprueba named.conf
named-checkconf /etc/named.conf                         # Explícito
named-checkzone ejemplo.com /etc/bind/db.ejemplo.com   # Comprueba un fichero de zona
```

> **Buena práctica:** ejecutar siempre `named-checkconf` y `named-checkzone` antes de recargar el servicio. Un error de sintaxis en un fichero de zona puede dejar el servidor sin responder a esa zona.

### Herramientas de consulta DNS

#### `dig` — La herramienta recomendada

`dig` (_Domain Information Groper_) es la herramienta de referencia para consultar servidores DNS. Su salida es detallada, estructurada y fácil de parsear. Forma parte del paquete `bind-utils` / `bind9-utils`. Referencia: [`man 1 dig`](https://linux.die.net/man/1/dig)

```bash
dig ejemplo.com                          # Consulta el registro A por defecto
dig ejemplo.com MX                       # Consulta registros MX
dig ejemplo.com NS                       # Servidores de nombres de la zona
dig -x 192.168.1.10                      # Consulta inversa (PTR)
dig @8.8.8.8 ejemplo.com                 # Consulta a un servidor DNS específico
dig ejemplo.com +short                   # Salida mínima (solo el resultado)
dig ejemplo.com +noall +answer           # Solo la sección Answer
dig ejemplo.com AXFR @ns1.ejemplo.com   # Solicita una transferencia de zona completa
dig ejemplo.com +dnssec                  # Incluye registros DNSSEC en la respuesta
```

#### `host` — Consultas simples

`host` proporciona consultas rápidas y salida compacta. Muestra por defecto los registros A, AAAA y MX.

```bash
host ejemplo.com                         # Registros A, AAAA y MX
host -t CNAME www.ejemplo.com            # Registro CNAME específico
host 192.168.1.10                        # Resolución inversa
host ejemplo.com 8.8.8.8                 # Consultar un servidor concreto
```

#### `nslookup` — Compatibilidad

La documentación oficial de BIND desaconseja el uso de `nslookup` por su interfaz arcana y su comportamiento frecuentemente inconsistente, y recomienda usar `dig` en su lugar. Sin embargo, `nslookup` sigue siendo muy conocido porque también está disponible en Windows, lo que lo hace útil para diagnósticos multiplataforma.

```bash
nslookup ejemplo.com                     # Consulta básica
nslookup -type=MX ejemplo.com           # Tipo de registro específico
nslookup ejemplo.com 8.8.8.8            # Servidor DNS específico
```

### Tabla resumen de herramientas de diagnóstico

|Herramienta|Uso recomendado|Observaciones|
|---|---|---|
|`dig`|**Principal**: diagnóstico detallado, scripting, verificación de DNSSEC|Salida completa y estructurada|
|`host`|Consultas rápidas y simples|Salida compacta|
|`nslookup`|Compatibilidad con Windows o usuarios habituados|Desaconsejado por BIND en favor de `dig`|
|`named-checkconf`|Validar sintaxis de `named.conf` antes de recargar|Solo sintaxis, no semántica|
|`named-checkzone`|Validar ficheros de zona antes de recargar|Detecta registros malformados|
|`rndc`|Administración en caliente del servidor BIND|Preferido a reinicios del servicio|

---

## Transferencia de zona y DNSSEC

### Transferencia de zona

La **transferencia de zona** es el mecanismo mediante el cual un servidor esclavo se sincroniza con el maestro para obtener actualizaciones. Hay dos tipos:

|Tipo|Descripción|
|---|---|
|**AXFR** (_Full Zone Transfer_)|Transfiere la zona completa. Se usa en la primera sincronización o cuando el diferencial no está disponible|
|**IXFR** (_Incremental Zone Transfer_)|Solo transfiere los cambios desde el último serial conocido. Más eficiente en zonas grandes|

La transferencia se configura en el bloque `zone` de `named.conf` mediante `allow-transfer`. Sin restricciones, cualquier servidor podría solicitar una copia completa de la zona, lo que supone un riesgo de reconocimiento (_zone enumeration_).

### Seguridad: TSIG (Transaction Signature)

**TSIG** (_Transaction Signatures_, [RFC 2845](https://www.rfc-editor.org/rfc/rfc2845)) es el mecanismo estándar para autenticar transferencias de zona y actualizaciones dinámicas mediante una **clave secreta compartida** (HMAC). Es más sencillo de desplegar que DNSSEC para proteger las comunicaciones servidor-a-servidor.

Generar una clave TSIG con `tsig-keygen`:

```bash
tsig-keygen -a HMAC-SHA256 clave-maestro-esclavo
```

Salida (bloque para incluir en `named.conf` de ambos servidores):

```
key "clave-maestro-esclavo" {
    algorithm hmac-sha256;
    secret "base64==";
};
```

Configurar la transferencia usando la clave:

```
// En el maestro
zone "ejemplo.com" {
    type master;
    allow-transfer { key "clave-maestro-esclavo"; };
};

// En el esclavo
zone "ejemplo.com" {
    type slave;
    masters { 192.168.1.1 key "clave-maestro-esclavo"; };
};
```

### DNSSEC

**DNSSEC** (_DNS Security Extensions_, [RFC 4033–4035](https://www.rfc-editor.org/rfc/rfc4033)) añade **autenticación criptográfica** a las respuestas DNS mediante firma digital. Protege frente a ataques de envenenamiento de caché (_cache poisoning_) y falsificación de respuestas.

DNSSEC no cifra el tráfico DNS; garantiza la **integridad y autenticidad** de los registros.

#### Tipos de claves DNSSEC

|Clave|Nombre|Función|
|---|---|---|
|**KSK**|_Key Signing Key_|Firma la clave ZSK. Se cambia con poca frecuencia y su hash (registro DS) se registra en la zona padre|
|**ZSK**|_Zone Signing Key_|Firma el resto de registros de la zona. Se rota con mayor frecuencia|

#### Proceso de firma de una zona con BIND

**1. Generar las claves:**

```bash
# Generar la KSK (Key Signing Key)
dnssec-keygen -a ECDSAP256SHA256 -b 256 -f KSK -n ZONE ejemplo.com

# Generar la ZSK (Zone Signing Key)
dnssec-keygen -a ECDSAP256SHA256 -b 256 -n ZONE ejemplo.com
```

Cada llamada genera dos ficheros: `Kejemplo.com.+NNN+XXXXX.key` (clave pública) y `Kejemplo.com.+NNN+XXXXX.private` (clave privada).

**2. Firmar la zona:**

```bash
# Incluir las claves públicas en el fichero de zona antes de firmar
cat Kejemplo.com.*.key >> /etc/bind/db.ejemplo.com

# Firmar la zona (genera db.ejemplo.com.signed)
dnssec-signzone -o ejemplo.com -k Kejemplo.com.+NNN+XXXXX.key \
    /etc/bind/db.ejemplo.com Kejemplo.com.+NNN+XXXXX.key
```

**3. Actualizar `named.conf`** para apuntar al fichero firmado:

```
zone "ejemplo.com" {
    type master;
    file "/etc/bind/db.ejemplo.com.signed";
};
```

**4. Publicar el registro DS** en la zona padre (el registro DS es el hash de la KSK y es lo que establece la cadena de confianza hacia abajo en la jerarquía DNS).

> **Nota:** En BIND moderno (versión 9.16+) el proceso de firma puede automatizarse completamente con la directiva `dnssec-policy` en `named.conf`, que gestiona la generación de claves, la firma y la rotación automáticamente sin necesidad de ejecutar `dnssec-keygen` ni `dnssec-signzone` manualmente.

#### Activar la validación DNSSEC en el resolvedor

Para que el servidor valide las respuestas firmadas al hacer resolución recursiva:

```
options {
    dnssec-validation auto;   // Usa el ancla de confianza incluida en BIND para la raíz
};
```

---

## El resolvedor del cliente: `/etc/resolv.conf`

En el lado del cliente, el sistema consulta los servidores DNS indicados en `/etc/resolv.conf`:

```
nameserver 192.168.1.1
nameserver 8.8.8.8
search ejemplo.com local.lan    # Dominios de búsqueda para nombres cortos
```

> **Nota:** En sistemas modernos con **systemd-resolved**, `/etc/resolv.conf` es gestionado automáticamente como un enlace simbólico. No debe editarse directamente; se configura con `resolvectl` o a través de NetworkManager.

```bash
resolvectl status               # Estado del resolvedor systemd
resolvectl query ejemplo.com   # Consulta usando el resolvedor del sistema
```

# 7. Configuración de servicios avanzados de red

---

## Configuración de un servidor DHCP

[DHCP](https://es.ccm.net/aplicaciones-e-internet/museo-de-internet/enciclopedia/11104-que-es-el-protocolo-dhcp-y-para-que-sirve/) (_Dynamic Host Configuration Protocol_) es un protocolo TCP/IP que asigna automáticamente una configuración de red a los equipos de una LAN: dirección IP, máscara de subred, puerta de enlace, servidor DNS, nombre de dominio y otros parámetros. Está definido en el [RFC 2131](https://www.rfc-editor.org/rfc/rfc2131).

### Implementaciones de servidor DHCP

|Software|Descripción|
|---|---|
|[**ISC DHCP**](https://www.isc.org/dhcp/) (`isc-dhcp-server` / `dhcp`)|La implementación más extendida históricamente. Mantenida por el ISC. Incluye servidor (`dhcpd`), cliente (`dhclient`) y agente relay (`dhcrelay`)|
|[**Kea**](https://www.isc.org/kea/)|El sucesor moderno de ISC DHCP, también del ISC. Arquitectura modular, API REST y base de datos de concesiones en MySQL/PostgreSQL. Recomendado para nuevas instalaciones|
|[**dnsmasq**](https://thekelleys.org.uk/dnsmasq/doc.html)|Servidor ligero que combina DNS, DHCP y TFTP. Integra automáticamente los hosts con IP dinámica en el servidor DNS local. Muy usado en redes pequeñas y entornos de laboratorio|

> **Nota sobre ISC DHCP:** el ISC anunció el fin de vida (_End of Life_) de ISC DHCP en 2022. Para nuevas instalaciones se recomienda usar **Kea**, su sucesor oficial.

### Instalación y ficheros clave

```bash
sudo apt install isc-dhcp-server    # Debian / Ubuntu
sudo dnf install dhcp-server        # RHEL / Fedora / CentOS
```

|Fichero / ruta|Descripción|
|---|---|
|`/etc/dhcp/dhcpd.conf`|Fichero de configuración principal|
|`/var/lib/dhcpd/dhcpd.leases`|Base de datos de concesiones activas (RHEL/Fedora)|
|`/var/lib/dhcp/dhcpd.leases`|Base de datos de concesiones activas (Debian/Ubuntu)|

La base de datos de concesiones se actualiza continuamente y registra qué IP ha sido asignada a cada cliente (identificado por su dirección MAC), junto con los tiempos de inicio y expiración de cada concesión.

### Estructura de `dhcpd.conf`

El fichero se compone de dos categorías de sentencias: **parámetros** y **declaraciones**.

- Los **parámetros** indican cómo realizar una tarea, si debe realizarse o qué opciones de configuración de red se envían al cliente. Los que comienzan con la palabra clave `option` corresponden a opciones DHCP estándar (RFC); los que no, controlan el comportamiento del propio servidor.
- Las **declaraciones** describen la topología de red y los clientes: definen rangos de direcciones a asignar o aplican un grupo de parámetros a un conjunto de declaraciones.

Los parámetros globales (antes de cualquier bloque `{}`) se aplican a todos los clientes. Los que aparecen dentro de un bloque `subnet` o `host` solo afectan a ese ámbito.

### Ejemplo completo de `dhcpd.conf`

```
# Parámetros globales
option domain-name "ejemplo.com";
option domain-name-servers 192.168.1.10, 8.8.8.8;

default-lease-time 3600;     # Duración de la concesión si el cliente no especifica (segundos)
max-lease-time 86400;        # Duración máxima permitida para una concesión

authoritative;               # Este servidor es autoritativo para la red: envía DHCPNAK a clientes
                             # que solicitan IPs fuera del rango configurado
log-facility local7;

# Declaración de subred: obligatoria para cada interfaz del servidor
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;        # Rango dinámico de IPs a asignar
    option routers 192.168.1.1;               # Puerta de enlace
    option subnet-mask 255.255.255.255;
    option broadcast-address 192.168.1.255;
    option ntp-servers 192.168.1.10;          # Servidor NTP
}

# IP fija para un host específico por dirección MAC
host servidor-web {
    hardware ethernet 08:00:27:ab:cd:ef;
    fixed-address 192.168.1.50;
}
```

> **Nota:** `dhcpd` solo escucha en las interfaces para las que existe una declaración `subnet` que coincida con la IP de esa interfaz. Si una interfaz no tiene declaración de subred, el servidor no escucha en ella.

### Verificación y gestión del servicio

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf   # Verifica la sintaxis antes de reiniciar (¡siempre hacer esto!)
systemctl restart isc-dhcp-server    # Debian / Ubuntu
systemctl restart dhcpd              # RHEL / Fedora

# Consultar concesiones activas
cat /var/lib/dhcp/dhcpd.leases
grep "binding state active" /var/lib/dhcp/dhcpd.leases | wc -l   # Contar concesiones activas
```

### DHCP Relay: `dhcrelay`

En redes segmentadas (con VLANs o subredes separadas), los mensajes DHCP son broadcasts que no atraviesan los rúters. El **agente DHCP Relay** (`dhcrelay`) soluciona esto: recibe los broadcasts DHCP en una subred y los reenvía como unicast al servidor DHCP en otra subred.

```bash
dhcrelay -i eth0 192.168.1.1    # Retransmite peticiones DHCP recibidas en eth0 al servidor 192.168.1.1
```

---

## Gestión básica de cuentas LDAP

[LDAP](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/system-level_authentication_guide/openldap) (_Lightweight Directory Access Protocol_) es un protocolo de nivel de aplicación que permite el acceso a un servicio de directorio jerárquico y distribuido. Es el estándar para la gestión centralizada de identidades en redes corporativas: usuarios, grupos, equipos, impresoras, etc.

### Conceptos clave

|Concepto|Descripción|
|---|---|
|**DIT** (_Directory Information Tree_)|Estructura jerárquica en forma de árbol que organiza toda la información del directorio|
|**DN** (_Distinguished Name_)|Identificador único y completo de una entrada en el DIT (p. ej. `uid=jdoe,ou=usuarios,dc=ejemplo,dc=com`)|
|**RDN** (_Relative Distinguished Name_)|La parte del DN que identifica la entrada dentro de su nivel (p. ej. `uid=jdoe`)|
|**Entrada** (_Entry_)|Cada nodo del DIT. Está formado por un conjunto de atributos definidos por su _objectClass_|
|**LDIF** (_LDAP Data Interchange Format_)|Formato de texto estándar para representar entradas y cambios en un directorio LDAP. Es el formato de intercambio de datos|

Los niveles superiores del DIT suelen seguir la estructura de nombres DNS. Por ejemplo, el dominio `ejemplo.com` se representa como `dc=ejemplo,dc=com`. Por debajo aparecen unidades organizativas (`ou=`), usuarios (`uid=`), grupos (`cn=`), etc.

### OpenLDAP

[OpenLDAP](https://www.openldap.org/) es la implementación libre y de código abierto del protocolo LDAP más usada en Linux. Consta de un servidor (`slapd`, _Stand-alone LDAP Daemon_) y un conjunto de utilidades cliente.

**Ficheros de configuración:**

|Fichero / ruta|Descripción|
|---|---|
|`/etc/openldap/slapd.d/` o `cn=config`|Configuración del servidor `slapd` en el formato moderno (_online configuration_). No debe editarse directamente; se modifica con `ldapmodify`|
|`/etc/openldap/ldap.conf`|Configuración de las utilidades cliente (servidor por defecto, base DN, etc.)|
|`/etc/ldap/schema/`|Ficheros de esquema que definen los tipos de objetos y atributos disponibles|

> **Nota sobre la configuración de slapd:** el fichero `slapd.conf` clásico ha sido sustituido por un sistema de configuración dinámica almacenado en el propio directorio bajo `cn=config`. Aunque los ficheros de configuración de `slapd-config` se almacenan como ficheros LDIF en texto plano, **nunca deben editarse directamente**. Los cambios deben realizarse a través de operaciones LDAP como `ldapadd`, `ldapdelete` o `ldapmodify`.

### Utilidades cliente de OpenLDAP

Las principales utilidades de cliente son:

|Comando|Función|
|---|---|
|`ldapadd`|Añade entradas al directorio desde un fichero LDIF|
|`ldapmodify`|Modifica entradas existentes del directorio|
|`ldapdelete`|Elimina entradas del directorio|
|`ldapsearch`|Realiza búsquedas en el directorio|
|`ldappasswd`|Cambia la contraseña de un usuario LDAP|

### Ejemplo de fichero LDIF

```ldif
# Añadir un usuario
dn: uid=jdoe,ou=usuarios,dc=ejemplo,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
uid: jdoe
cn: John Doe
sn: Doe
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/jdoe
loginShell: /bin/bash
userPassword: {SSHA}hashedpassword==
```

```bash
# Añadir la entrada al directorio
ldapadd -x -D "cn=admin,dc=ejemplo,dc=com" -W -f nuevo_usuario.ldif

# Buscar un usuario
ldapsearch -x -b "dc=ejemplo,dc=com" "(uid=jdoe)"

# Buscar todos los usuarios
ldapsearch -x -b "ou=usuarios,dc=ejemplo,dc=com" "(objectClass=posixAccount)"
```

---

## Configuración de Linux como rúter y firewall

Para que un equipo Linux actúe como rúter entre varias redes, necesita al menos dos interfaces de red y tener habilitado el **IP forwarding** (ver Punto 5). Además, es necesario configurar las reglas de firewall y, habitualmente, NAT.

### El subsistema Netfilter

**Netfilter** es el subsistema del kernel de Linux que proporciona filtrado de paquetes con o sin estado, NAT y modificación de cabeceras IP. Desde él se gestionan todas las reglas de firewall. Referencia oficial: [Netfilter Project](https://www.netfilter.org/)

A lo largo del tiempo, Netfilter ha tenido dos interfaces de usuario principales:

|Herramienta|Estado|Descripción|
|---|---|---|
|`iptables`|**Obsoleta** (mantenida por compatibilidad)|La herramienta clásica. Requiere herramientas separadas para IPv4 (`iptables`), IPv6 (`ip6tables`), ARP (`arptables`) y bridges (`ebtables`)|
|`nftables` / `nft`|**Actual** (predeterminada desde RHEL 8, Debian 10, Ubuntu 20.04)|Sustituto unificado de todas las anteriores. IPv4 e IPv6 en un solo conjunto de reglas, sintaxis más limpia y mejor rendimiento|

> **`iptables` en sistemas modernos:** en las distribuciones actuales, el comando `iptables` es en realidad un alias de `iptables-legacy` o una capa de compatibilidad sobre `nftables` (`iptables-nft`). Para nueva configuración se debe usar `nft` directamente.

### `iptables`: concepto y estructura (referencia histórica)

`iptables` organiza las reglas en **tablas** y **cadenas**:

|Tabla|Propósito|
|---|---|
|`filter`|Filtrado de paquetes (la tabla por defecto)|
|`nat`|Traducción de direcciones de red (NAT, MASQUERADE)|
|`mangle`|Modificación de cabeceras de paquetes|

Cadenas principales de la tabla `filter`:

|Cadena|Tráfico que procesa|
|---|---|
|`INPUT`|Paquetes destinados al propio equipo|
|`OUTPUT`|Paquetes originados en el propio equipo|
|`FORWARD`|Paquetes que pasan a través del equipo (enrutados)|

Sintaxis básica:

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT       # Permite SSH entrante
iptables -A INPUT -p tcp --dport 80 -j ACCEPT       # Permite HTTP
iptables -A INPUT -j DROP                           # Descarta todo lo demás
iptables -L -n -v                                   # Lista las reglas actuales
```

### `nftables`: la herramienta moderna

`nftables` unifica el filtrado de IPv4, IPv6, ARP y bridges en una sola herramienta (`nft`). A diferencia de `iptables`, no tiene tablas ni cadenas predefinidas: se crean solo las que se necesitan. Referencia: [nftables wiki](https://wiki.nftables.org/)

**Conceptos fundamentales:**

```
Familia de direcciones (ip, ip6, inet, arp, bridge)
    └── Tabla (contenedor de cadenas y sets)
            └── Cadena (contenedor de reglas, enganchada a un hook del kernel)
                    └── Regla (condición + acción)
```

La familia `inet` (IPv4 + IPv6 combinados) es la recomendada para la mayoría de configuraciones de servidor.

**Ejemplo de firewall básico con `nftables`:**

```
# /etc/nftables.conf
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        iif lo accept                          # Permitir tráfico de loopback
        ct state established,related accept    # Permitir respuestas a conexiones ya establecidas
        ct state invalid drop                  # Descartar paquetes con estado inválido
        ip protocol icmp accept                # Permitir ICMP (ping IPv4)
        ip6 nexthdr icmpv6 accept              # Permitir ICMPv6 (ping IPv6)
        tcp dport 22 accept                    # Permitir SSH
        tcp dport { 80, 443 } accept           # Permitir HTTP y HTTPS
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

```bash
nft -f /etc/nftables.conf          # Aplica el fichero de reglas
nft list ruleset                   # Lista todas las reglas activas
systemctl enable --now nftables    # Habilita y activa nftables con systemd
```

> **Importante:** la configuración de firewall con herramientas de GUI (como `firewalld` o `ufw`) y con scripts de `nft`/`iptables` directamente **son incompatibles**: pueden sobreescribirse mutuamente. Hay que elegir una sola forma de gestionar el firewall.

### Configuración de NAT (_Network Address Translation_)

NAT permite que equipos con IPs privadas (no enrutables en Internet) accedan a redes públicas usando la IP pública del rúter. El tipo más usado es **IP Masquerade** (SNAT dinámico): el rúter sustituye la IP origen privada por su propia IP pública y mantiene una tabla de seguimiento de conexiones para saber a qué cliente interior reenviar cada respuesta.

**Con `nftables` (recomendado):**

```bash
# Habilitar IP Forwarding (permanente)
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

# Añadir la regla de masquerade (la interfaz de salida a Internet es enp3s0)
nft add table ip nat
nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
nft add rule ip nat postrouting oifname "enp3s0" masquerade
```

**Con `iptables` (referencia clásica):**

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### Protocolo de enrutamiento RIP

Para que los rúters compartan información sobre las subredes disponibles se usan **protocolos de enrutamiento dinámico**. El más sencillo es **RIP** (_Routing Information Protocol_). En Linux, el demonio `routed` implementa RIPv1, mientras que el paquete [**Quagga**](https://www.quagga.net/) (o su sucesor [**FRRouting**](https://frrouting.org/)) implementa RIPv1/v2, OSPF, BGP y otros protocolos de enrutamiento avanzados.

---

## Implementación de SSH

**SSH** (_Secure Shell_) es un protocolo de red que proporciona acceso remoto cifrado a sistemas Unix/Linux, así como transferencia segura de ficheros y tunelización de tráfico. Está definido en el [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251). La implementación libre más usada es [**OpenSSH**](https://www.openssh.com/). Referencia Red Hat: [Configuring OpenSSH](https://docs.redhat.com/es/documentation/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/configuring-and-starting-an-openssh-server_using-secure-communications-between-two-systems-with-openssh)

### Ficheros de configuración

|Fichero|Descripción|
|---|---|
|`/etc/ssh/sshd_config`|Configuración del **servidor** SSH (`sshd`)|
|`/etc/ssh/ssh_config`|Configuración por defecto del **cliente** SSH para todos los usuarios del sistema|
|`~/.ssh/config`|Configuración del cliente SSH específica del usuario (tiene prioridad sobre `ssh_config`)|
|`~/.ssh/known_hosts`|Claves públicas de los servidores remotos que el usuario ha aceptado previamente|
|`~/.ssh/authorized_keys`|Claves públicas de clientes autorizados a conectarse a esta cuenta sin contraseña|

### Funcionamiento del cifrado en SSH

SSH usa **criptografía asimétrica** para dos propósitos distintos:

1. **Autenticación del servidor:** el servidor tiene su propio par de claves (las _host keys_, en `/etc/ssh/`). Cuando un cliente se conecta por primera vez, acepta y guarda la clave pública del servidor en `~/.ssh/known_hosts`. En conexiones sucesivas, verifica que la clave del servidor no haya cambiado (protección frente a ataques _man-in-the-middle_).
2. **Autenticación del cliente:** el método más seguro es la autenticación por clave pública. El cliente genera su par de claves; la pública se instala en el servidor (en `~/.ssh/authorized_keys` de la cuenta de destino) y la privada nunca sale del cliente.

Una vez autenticadas ambas partes, la sesión se cifra con **criptografía simétrica** (p. ej. AES) negociada durante el intercambio inicial.

### Generación de claves con `ssh-keygen`

```bash
# Generar un par de claves Ed25519 (algoritmo moderno y recomendado)
ssh-keygen -t ed25519 -C "comentario_identificativo"

# Generar RSA de 4096 bits (compatible con sistemas más antiguos)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_servidor

# Generar claves para el servidor (las host keys)
ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
```

Los algoritmos recomendados en 2024+ son **Ed25519** (el más moderno y eficiente) y **ECDSA**. RSA sigue siendo ampliamente compatible pero requiere claves de al menos 3072 bits para considerarse seguro.

**Permisos correctos para los ficheros de claves** (SSH rechaza claves con permisos demasiado abiertos):

|Fichero|Permisos|Propietario|
|---|---|---|
|Clave **privada** del cliente (`~/.ssh/id_ed25519`)|`600`|el usuario|
|Clave **pública** del cliente (`~/.ssh/id_ed25519.pub`)|`644`|el usuario|
|Directorio `~/.ssh/`|`700`|el usuario|
|`~/.ssh/authorized_keys`|`600`|el usuario|
|Claves **privadas** del servidor (`/etc/ssh/ssh_host_*`)|`600`|`root`|
|Claves **públicas** del servidor (`/etc/ssh/ssh_host_*.pub`)|`644`|`root`|

> **Corrección respecto a los apuntes originales:** los permisos estaban intercambiados. La clave **privada** debe ser `600` (solo el propietario puede leerla y escribirla) y la **pública** `644`. Si la clave privada tiene permisos más abiertos, OpenSSH se negará a usarla.

### Instalar la clave pública en el servidor

La forma recomendada es usar `ssh-copy-id`, que gestiona automáticamente los permisos correctos:

```bash
ssh-copy-id usuario@servidor.ejemplo.com             # Copia la clave pública por defecto
ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@servidor  # Especifica el fichero de clave
```

Equivalente manual (si `ssh-copy-id` no está disponible):

```bash
cat ~/.ssh/id_ed25519.pub | ssh usuario@servidor "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Acceso remoto con `ssh`

```bash
ssh servidor.ejemplo.com                       # Conecta como el usuario actual
ssh usuario@servidor.ejemplo.com               # Especifica el usuario
ssh -p 2222 usuario@servidor.ejemplo.com       # Puerto no estándar
ssh -i ~/.ssh/id_ed25519 usuario@servidor      # Especifica el fichero de clave privada
```

La primera conexión a un servidor nuevo muestra su huella (_fingerprint_) para que el administrador la verifique. Al aceptarla, se añade a `~/.ssh/known_hosts`.

> Si la clave del servidor cambia (p. ej. por una reinstalación), SSH mostrará una advertencia de seguridad. Puede eliminarse la entrada antigua con: `ssh-keygen -R servidor.ejemplo.com`

### Copia segura de ficheros: `scp` y `rsync`

```bash
# scp: copia puntual de ficheros
scp fichero.txt usuario@servidor:/ruta/destino/          # Local → remoto
scp usuario@servidor:/ruta/fichero.txt ./local/          # Remoto → local
scp -r directorio/ usuario@servidor:/destino/            # Copia recursiva de directorio

# rsync sobre SSH: sincronización eficiente (solo transfiere cambios)
rsync -avz -e ssh directorio/ usuario@servidor:/destino/ # Sincronización remota
```

> `rsync` es preferible a `scp` para transferencias grandes o repetidas, ya que solo copia los ficheros que han cambiado.

### Parámetros clave de `sshd_config`

```bash
# Editar la configuración del servidor
sudo nano /etc/ssh/sshd_config

# Aplicar los cambios sin interrumpir sesiones activas
sudo systemctl reload sshd
```

Parámetros más importantes para la seguridad:

|Parámetro|Valor recomendado|Descripción|
|---|---|---|
|`PermitRootLogin`|`no`|Deshabilita el inicio de sesión directo como root|
|`PasswordAuthentication`|`no`|Deshabilita la autenticación por contraseña (solo claves)|
|`PubkeyAuthentication`|`yes`|Habilita la autenticación por clave pública|
|`Port`|(p. ej. `2222`)|Cambiar el puerto reduce el ruido de bots en los logs|
|`AllowUsers`|`usuario1 usuario2`|Lista blanca de usuarios que pueden conectarse|
|`MaxAuthTries`|`3`|Limita los intentos de autenticación fallidos por conexión|
|`ClientAliveInterval`|`300`|Envía un keepalive al cliente cada 300 s para detectar sesiones caídas|

> **Buena práctica:** antes de deshabilitar `PasswordAuthentication`, verificar que la autenticación por clave pública funciona correctamente en otra sesión SSH abierta, para no quedarse sin acceso.

# 8. Configuración de servidores de ficheros
## Servidores Samba
En los sistemas de red actuales es común que convivan dispositivos Windows y Unix, y ambos deben comunicarse entre sí.
SMB es un protocolo de red que pertenece al nivel de aplicación en el modelo OSI. Que permite compartir archivos e impresoras (entre otras cosas) entre nodos de una red. Se usa pricipalmente en ordenadores con Windows y DOS, por tanto para que un sistema linux pueda compartir archivos e impresoras, debe estar implementado este protocolo. Hay varias versiones de SMB usadas por los sistemas Windows (CIFS, que era parte de Microsoft Windows NT 4.0; SMB1 usada por Windows 2000, XP, Windows Server 2003 y Windows Server 2003 R2; SMB2, usada por Windows Vista y Windows Server 2008; SMB2.1, usada en Windows 7 y Windows Server 2008 y SMB3, para Windows 8 y Windows Server 2012).
[Samba](https://www.samba.org/) es una implementación de código abierto de la suite de protocolos de red de Microsoft Windows. Samba es un servidor para los clientes que utilizan el protocolo Microsoft CIFS/SMB para el uso compartido de impresoras y ficheros. Hay otras opciones, como [PowerBroker Identity Services](https://powerbroker-identity-services-open.soft112.com/)
Samba se instala usando el paquete `samba`o `samba-server`. La suite completa de Samba se compone de varios paquetes además de el mencionado:
`samba-common`: Archivos comunes de Samba usados por clientes y servidores.
- `smbclient`: Cliente de Samba
- `swat`: Herramienta de administración de Samba vía web.
- `samba-doc`: Documentación
- `smbfs`: Comandos para montar/desmontar unidades de red Samba
- `winbind`: Resuelve información de usuarios y grupos de servidores Windows NT.
### Configuración de un servidor Samba
Para configurar Samba, debemos ir al archivo `/etc/smb.conf` o `/etc/sanmba/smb.conf`. El fichero tiene secciones identificadas por un nombre entre corchetes y en cada sección se definen los valores de las distintas opciones por medio de una asignación (`=`).
Hay 3 secciones obligatorias:
- `[global]`: Valores por defecto que afectan al servidor en conjunto
- `[homes]`: Permite compartir carpetas home del usuario.
- `[printers]`: Permite compartir impresoras.
Además de estas secciones, si queremos compartir una carpeta, debemos crear una nueva sección, cuyo nombre será el nombre del nuevo recurso compartido. [Ejemplo de fichero.](https://cdn.adrformacion.com/c/linux5/8/samba_conf.txt)
Los sistemas Windows pueden configurarse como:
- Grupos de trabajo, basados en NetBIOS.
- Dominios NT, solo si el controlador de dominio es Windows NT
- Dominios Active Directory, de Windows Server 2000 en adelante.
Uno de los parámetros de configuración más importantes es `security`, en `[global]`. Tiene las siguientes opciones:
- `user`: Si el servidor acepta la combinación usuario/contraseña el usuario podrá montar varios recursos compartidos sin tener que especificar una contraseña para cada instancia. Es el `default`.
- `share`: Acepta una contraseña sin nombre de usuario explícito desde el cliente.
- `domain`: Samba es miembro del dominio NT. Dispone de una cuenta de máquina y hace que todas las solicitudes de autenticación pasen a través de los controladores de dominio.
- `ADS`: Si tienes AD, puedes unirte al dominio como miembro nativo de AD. Se puede unir al ADS usando Kerberos.
- `server`: Se usa este modo si Samba no es servidor miembro del dominio.
### Configuración de opciones de Grupos de Trabajo y Dominios
SMB funciona sobre NetBIOS. Si no establecemos correctamente el parámetro `Workgroup`, los clientes Windows no nos encontrarán. Para saber qué grupo de trabajo está nuestra máquina, `nmblookup -MS -`.  
LAs opciones `server`, `domain` y `ADS` no necesitan mantener cuentas de usuario Samba, pero implican cierta conectividad entre el servidr Samba y la red Windows. Para unir el servidor Samba a un dominio, primero debe configurarse ne Windws esta posibilidad y luego usarse desde el servidor Samba `net join member -U usuario_admin dominio`. Para mantener sincronizada la DB, puede usarse LDAP, compatible con el AD, o bien puede utilizarse [winbindd](https://www.samba.org/samba/docs/current/man-html/winbindd.8.html), que habilita a Linux para usar al conexión LDAP del AD.
### Configuración de las opciones de contraseña
En `[global]` tenemos la opción `encrypt passwords`. Si la poenmos con `No`, limitará la autenticación a la DB usuarios de Linux y ciertos clientes de Windows no podrán conectarse. Si se establece en `Yes`, Samba requerirá de su propia DB independiente de la de Linux y habrá que añadirlos con `smbpasswd -a usuario_del_sistema`. Se puede automatizar la tarea con el script `/usr/bin/mksmbpasswd`. La opción `user` debe estar habilitada y fuerza a Samba a tener su propia DB.
### Utilidades de depuración de errores en Samba
Hay que reiniciar el sistema Samba aplicar los cambios en la configuración. Antes de ello, podemos usar `testparam` para ayudar a depurar errores en el fichero.
Podemos usar también desde el cliente `nmblokup` para localizar información sobre un servidor Samba. 
La utilidad `smbstatus` informa sobre el estado actual del servidor, clientes conectados a él y ficheros abiertos.
Una vez que Samba está funcionando, consiste principalmente en dos procesos (daemons) `/usr/bin/smbd` (Suministra servicios para compartir archivos e impresión y es responsable por la autenticación de usuarios, el boqueo y compartir daros a través del protocolo SMB) y `/usr/sbin/nmbd` (Responde a las peticiones del servicio de nombres NetBIOS tales como aquellas producidas por SMB/CIFS en sistemas basados en Windows y participa en protocolos de navegación que forman la vista Entorno de Red Windows. EL tráfico NMB se escucha por el puerto UDP 137). Los ficheros de log están en `/var/log/samba`.
### Configuración del cliente de Samba
Nos permite interactuar con servidores Samba o sistemas Windows usando el protocolo SMB/CIFS como cliente. El programa `smbclient` proporciona un entorno de línea de comandos similar a FTP para [acceder](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/5/html/deployment_guide/s1-samba-connect-share) a recursos compartidos.
Con `mount` también montar los recursos compartidos Samba en nuestro sistema de ficheros. Para ello lo indicamos como tipo `cifs` (`mount -t cifs //servidor_samba/nombre_recurso /mnt/directorio`).
## Servidores NFS
## Servidores FTP