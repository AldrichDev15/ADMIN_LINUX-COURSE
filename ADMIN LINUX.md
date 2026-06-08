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
Cada servidor DNS se configurará principalmente para al menos una de las siguientes tareas:
- Resolver consultas DNS planteadas por los equipos o servidores de nuestra red, poniéndose en contacto con otros servidores DNS, en un proceso recursivo.
- Resolver consultas autoritativas sobre aquellos servidores o equipos pertenecientes al dominio para el que él es autoritativo (estas respuestas son el punto y final a una consulta recursiva realizada desde otro servidor DNS, ya sea de dentro o fuera de nuestra red).
Hay 4 tipos de servidores de nombres:
- Maestro/Primario: Almacena los registros de zonas originales (primarias) y de autoridad para un espacio de nombre, y responde a preguntas acerca del espacio de nombres de otros servidores DNS
- Esclavo/Secundario: Responde a peticiones de otros servidores de nombres relativos a los espacios de nombres para el que tiene autoridad. Sin embargo, éstos obtienen la información del espacio de nombres desde los servidores maestros.
- Solo caché: Ofrece servicios de resolución de nombres a IP pero no tienen ninguna autoridad sobre ninguna zona. Las respuestas en general se introducen en una caché por un periodo fijo de tiempo.
- Solo reenvío: Reenvía la petición a una lista específica de servidores DNS para su resolución.
Un servidor de nombres puede implementar uno o varios de estos roles para según qué zonas.
Una zona DNS es una parte de un espacio de nombres de dominio que utiliza el sistema de nombres de dominio para los que la responsabilidad administrativa ha sido delegada. Es conveniente almacenar varios servidores DNS en cada zona.
- Zonas de búsqueda directa: Devuelven la IP.
- Zonas de búsqueda inversa: Buscan un nombre en función de la IP.
Un servidor DNS puede contar con información para el acceso directo a los servidores raíz del sistema DNS, aquellos de autoridad para los niveles superiores.
## Implementación de DNS en Linux
Hay varias implementaciones de Servidores DNS:
- [BIND](https://www.isc.org/downloads/bind/): Es el software más usado de Internet que proporciona una plataforma sólida y estable sobre la que las organizaciones pueden construir sistemas de computación distribuida con garantía de compatibilidad con los estándares DNS publicados.
- `dnsmasq`: Servidor ligero diseñado para proporcionar servicios DNS, DHCP y TFTP a una red a pequeña escala. 
- `djbdns`: Alternativa a BIND enfocada en la seguridad, aunque [djbdns](https://cr.yp.to/djbdns.html) no es mantenido frecuentemente.
Hablaremos concretamente de BIND. El fichero de configuración es `/etc/named.conf`. DOs de las funciones más importantes desde el punto de vista del funcionamiento del protocolo DNS es la definición de las opciones que indican en qué modo trabajará el servidor DNS y por otro lado la descripción de las zonas DNS almacenadas en ese servidor.
- Forwarders: indica los servidores DNS a los que reenviar consultas DNS.
- Listen-on: Sobre las redes sobre las cuales proporciona el servicio.
- Forward-only: Indica que no se intenta una resolución recursiva de consultas, sino que se reenvían directamente.
En el fichero de configuración definimos también las zonas como primarias o secundarias en función de si ese servidor actualiza registros en el fichero de la zona o los transfiere de un servidor autoritativo para esa zona. Además, pueden ser de búsqueda directa o de búsqueda inversa.
En este fichero de configuración, también se identifican los ficheros en los que se almacenan las zonas que mantiene dicho servidor DNS , es decir, las asociaciones entre IPs y nombre, o entre IPs y servicios para el dominio correspondiente a dicha zona. 
## Diagnóstico de DNS
Podemos usar `name-checkzone`y `named-checkconf` para verificar la sintaxis de los ficheros de zonas y configuración BIND respectivamente. El servidor de BIND se denomina `named`.
El comando `host` muestra los registros A, CNAME y PTR de un servidor.
`nslookup` ha sido _deprecated_ pero se sigue usando bastante. El comando `dig`es similar a los anteriores pero la información aparece depurada.
## Configuración de la transferencia de zona con DNSSEC
La transferencia de zona es cuando un servidor esclavo en modo solo lectura deba sincronizarse con el maestro para recibir las actualizaciones que pueda sufrir.
Se configura en `/etc/named.conf`, pero es vulnerable a ataques al no tener encriptación. Se puede configurar para que se use un modo seguro (DNSSEC).
Generamos dos ficheros de claves pública y privada que se usarán para la encriptación. La clave pública se reparte entre los servidores que van a recibir la transferencia y la privada se usa para firmar usando el comando `dnssec-signzone`.
El fichero encriptado que se genera será el que se use como fichero de zona, actualizándolo en `/etc/named.conf`.
