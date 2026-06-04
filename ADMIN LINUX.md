# 1. Iniciando el sistema

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

En Linux/Unix existe un único árbol de directorios. No hay letras de unidad como en Windows. **Montar** es la acción de integrar el sistema de ficheros de un dispositivo dentro de ese árbol, haciéndolo accesible desde un directorio llamado **punto de montaje**.

El punto de montaje debe existir y, en la práctica, estar vacío antes del montaje (cualquier contenido previo quedará oculto mientras el dispositivo esté montado).

### Comando `mount` y `umount`

```bash
mount <dispositivo> <punto_de_montaje>        # Monta un sistema de ficheros
mount -t ext4 /dev/sdb1 /mnt/datos           # Especificando el tipo explícitamente
mount -o ro /dev/sdb1 /mnt/datos             # Montaje en solo lectura
umount <punto_de_montaje>                     # Desmonta el sistema de ficheros
umount /dev/sdb1                             # También puede usarse el dispositivo
```

Opciones de montaje más comunes (se pasan con `-o`):

| Opción | Descripción |
|---|---|
| `ro` / `rw` | Solo lectura / lectura y escritura |
| `noexec` | Impide ejecutar binarios en esa partición |
| `nosuid` | Ignora los bits SUID/SGID |
| `noatime` | No actualiza el tiempo de acceso al leer ficheros (mejora el rendimiento) |
| `defaults` | Equivale a `rw,suid,dev,exec,auto,nouser,async` |
| `nofail` | No provoca un error de arranque si el dispositivo no está disponible |

Para ver los sistemas de ficheros actualmente montados:

```bash
df -h          # Muestra espacio usado y disponible en cada sistema montado
mount          # Lista todos los montajes activos
findmnt        # Vista jerárquica más legible de los montajes activos
```

---

### Montaje permanente: `/etc/fstab`

El fichero `/etc/fstab` controla qué sistemas de ficheros se montan automáticamente en el arranque. Referencia oficial: [`man 5 fstab`](https://man7.org/linux/man-pages/man5/fstab.5.html)

Cada línea tiene **seis campos** separados por espacios o tabulaciones:

```
<dispositivo>  <punto_montaje>  <tipo_fs>  <opciones>  <dump>  <fsck>
```

Ejemplo real:

```
UUID=9a8b7c6d-5e4f-3210-abcd-ef1234 /        ext4  errors=remount-ro  0  1
UUID=ABC1-DEF2                       /boot/efi vfat  umask=0077         0  1
/swapfile                            none      swap  sw                 0  0
```

Descripción de cada campo:

| Campo | Descripción |
|---|---|
| **Dispositivo** | Identificador del dispositivo. Se recomienda usar `UUID=<valor>` en lugar del nombre de dispositivo (`/dev/sda1`) para evitar problemas si cambia el orden de detección de discos |
| **Punto de montaje** | Directorio donde se montará el sistema de ficheros. Para swap se usa `none` |
| **Tipo de FS** | Sistema de ficheros: `ext4`, `xfs`, `btrfs`, `vfat`, `swap`, `nfs`, etc. |
| **Opciones** | Opciones de montaje separadas por comas (ver tabla anterior). `defaults` es el valor más habitual |
| **Dump** | `1` activa las copias de seguridad mediante la utilidad `dump` (prácticamente obsoleta); `0` la desactiva. **Casi siempre se pone `0`** |
| **fsck** | Orden de comprobación con `fsck` al arrancar: `0` = no comprobar, `1` = comprobar primero (solo para `/`), `2` = comprobar después. XFS, Btrfs y sistemas de red deben usar `0` |

Para obtener el UUID de un dispositivo:

```bash
blkid                  # Muestra UUID y tipo de todos los dispositivos
blkid /dev/sdb1        # Solo para un dispositivo concreto
lsblk -f               # Vista alternativa con árbol de particiones
```

> **Advertencia:** Un `/etc/fstab` mal configurado puede impedir que el sistema arranque. Es recomendable hacer una copia de seguridad antes de editarlo y probar los cambios con `mount -a` sin reiniciar.

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

# 4. Gestión avanzada de discos
