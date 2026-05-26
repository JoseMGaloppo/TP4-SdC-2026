# TP4-SdC-2026

##  Desafío #1: Empaquetado y Seguridad del Kernel

### 1. ¿Qué es `checkinstall` y para qué sirve?

`checkinstall` es una herramienta de línea de comandos en Linux diseñada para facilitar la gestión de programas que se compilan a partir de su código fuente. 

Normalmente, al compilar software manualmente, el último paso es ejecutar `make install`. Este comando copia los binarios y archivos directamente a los directorios del sistema operativo (`/usr/local/bin`, etc.). El gran problema de esto es que el gestor de paquetes del sistema (como APT en Ubuntu) no se entera de esta instalación, lo que hace que rastrear y desinstalar estos archivos en el futuro sea muy difícil y propenso a ensuciar el sistema.

**¿Para qué sirve?** `checkinstall` reemplaza al comando `make install`. Su función es monitorear la instalación y, en lugar de copiar los archivos directamente, crea un paquete de instalación estándar (`.deb` en sistemas basados en Debian/Ubuntu, o `.rpm` en RedHat) y lo instala a través del gestor de paquetes nativo. Esto permite que el software compilado manualmente quede registrado en el sistema y se pueda desinstalar limpiamente usando comandos convencionales (como `apt-get remove` o `dpkg -r`).

---

### 2. Empaquetando un "Hello World" con `checkinstall`

Para demostrar su uso con un programa clásico de "Hola Mundo" en C, es requisito que el proyecto cuente con un archivo `Makefile` que incluya una regla de instalación (`install`). A continuación se detalla el procedimiento:

**Paso A: Crear el código fuente (`hello.c`)**
```c
#include <stdio.h>

int main() {
    printf("Hello, World! Empaquetado con checkinstall.\n");
    return 0;
}
```

**Paso B: Crear el `Makefile`**
Debe contener las instrucciones para compilar y para instalar el binario.
```makefile
all: hello

hello: hello.c
	gcc hello.c -o hello

install:
	install -m 755 hello /usr/local/bin/

clean:
	rm -f hello
```

**Paso C: Compilar y Empaquetar**
En la terminal, dentro de la carpeta donde están ambos archivos, se ejecuta:
```bash
make
sudo checkinstall
```
*Nota: Durante la ejecución, `checkinstall` abrirá un asistente interactivo pidiendo confirmar los metadatos del paquete (nombre, versión, descripción, etc.). Se pueden aceptar los valores por defecto presionando `Enter`.*

**Paso D: Verificación y Desinstalación**
El programa ahora es un paquete oficial en el sistema.
* **Para ejecutarlo en cualquier parte:** `hello`
* **Para ver información del paquete:** `dpkg -l | grep hello`
* **Para desinstalarlo de forma limpia:** `sudo dpkg -r hello`

---

### 3. Seguridad del Kernel: Módulos Firmados y Rootkits

#### Módulos del Kernel Firmados (Module Signing)
Para garantizar la integridad del sistema operativo, los kernels modernos de Linux implementan la verificación de firmas criptográficas para los módulos. Cuando esta función está activa (especialmente obligatoria si el sistema tiene *UEFI Secure Boot* habilitado), el kernel exige que cada módulo (`.ko`) posea una firma digital válida generada por una clave pública de confianza registrada en el sistema. 

Si un usuario o un programa intenta cargar un módulo que no está firmado o cuyo código ha sido alterado, el kernel bloquea la acción mediante un error de tipo *"Key was rejected by service"* o *"Operation not permitted"*.

#### ¿Qué es un Rootkit y por qué importa firmar los módulos?
Un **rootkit** es un tipo de software malicioso diseñado para otorgar acceso no autorizado de administrador (root) a un atacante, mientras oculta activamente su presencia en el sistema (escondiendo procesos, archivos maliciosos o conexiones de red).

Los rootkits más sofisticados y peligrosos operan a nivel del núcleo del sistema, conocidos como **LKM Rootkits** (*Loadable Kernel Module Rootkits*). Al insertarse como un módulo del kernel, obtienen el máximo nivel de privilegios y pueden alterar las llamadas al sistema (*syscalls*), volviéndose completamente invisibles para los antivirus tradicionales que operan en el espacio de usuario.

**Acción de Mitigación:**
La principal defensa contra los LKM rootkits es **evitar la carga de módulos no firmados** (configurando `CONFIG_MODULE_SIG_FORCE=y` al compilar el kernel o manteniendo Secure Boot encendido). De esta forma, incluso si un atacante logra obtener permisos de root y compilar su rootkit malicioso, el sistema operativo le impedirá inyectarlo en el kernel porque el atacante no posee la clave criptográfica privada necesaria para firmarlo.


## Desafío #2: Arquitectura del Sistema, Espacios de Memoria y Controladores

### 1. Funciones disponibles para un programa y un módulo

La principal diferencia entre el desarrollo de aplicaciones convencionales y el desarrollo de módulos del kernel radica en las bibliotecas de funciones a las que cada uno tiene acceso:

* **Programa de aplicación:** Tiene a su disposición la biblioteca estándar de C (como `glibc`). Esto le permite utilizar funciones comunes de manejo de memoria (`malloc`, `free`), entrada/salida (`printf`, `fopen`), manejo de cadenas (`strcpy`), entre otras. Estas funciones operan de manera segura delegando las tareas críticas al sistema operativo mediante llamadas al sistema (system calls).
* **Módulo del kernel:** No tiene acceso a ninguna biblioteca del espacio de usuario, incluida la biblioteca estándar de C. Un módulo solo puede invocar funciones que el propio kernel de Linux ha programado y exportado explícitamente para su uso (símbolos exportados). Por ejemplo, en lugar de `printf`, debe usar `printk`; en lugar de `malloc`, utiliza `kmalloc`; y no puede procesar operaciones de punto flotante de manera directa sin configuraciones especiales de contexto.

### 2. Espacio de usuario y Espacio del kernel

Los sistemas operativos modernos, como Linux, separan la memoria y los niveles de ejecución en dos grandes dominios por razones de seguridad, estabilidad y control de acceso al hardware. Esta separación se apoya en los anillos de privilegios del procesador (arquitectura x86).

* **Espacio de usuario (User Space):** Es el entorno de ejecución sin privilegios (Ring 3) donde se ejecutan todas las aplicaciones, demonios y procesos del usuario. Los programas aquí están aislados; si uno falla, el sistema operativo interviene y lo cierra (mediante un fallo de segmentación, por ejemplo), sin afectar al resto del sistema.
* **Espacio del kernel (Kernel Space):** Es el nivel de ejecución con máximos privilegios (Ring 0). Aquí reside el núcleo del sistema operativo y sus módulos. Tiene acceso total e irrestricto al procesador, a la memoria física y a todo el hardware. Un error de programación en este espacio (como una mala gestión de punteros) no se puede contener y provocará un "Kernel Panic", deteniendo todo el sistema abruptamente.

### 3. Espacio de datos

El manejo de la memoria difiere drásticamente dependiendo de dónde se esté operando:

* **Espacio de datos en User Space:** Cada programa dispone de su propio espacio de direcciones virtuales, el cual es ilusoriamente grande y continuo. El sistema operativo se encarga de traducir estas direcciones virtuales a direcciones físicas. Este espacio está dividido de forma ordenada en segmentos (código, datos inicializados, datos no inicializados/BSS, heap y stack). Además, esta memoria es "paginable" (swappable), lo que significa que el kernel puede mover temporalmente al disco rígido partes de la memoria que no se están utilizando activamente.
* **Espacio de datos en Kernel Space:** Todos los módulos del kernel comparten un único espacio de direcciones global. El kernel gestiona su propia memoria, la cual, por lo general, no es paginable y reside permanentemente en la memoria RAM principal (memoria física). El tamaño de la pila (stack) en el espacio del kernel es extremadamente limitado (típicamente 4KB u 8KB), por lo que declarar estructuras de datos locales grandes o usar recursividad profunda puede desbordar la pila rápidamente y causar un fallo catastrófico del sistema.

### 4. Drivers y el contenido del directorio /dev

Los **Controladores de Dispositivos (Drivers)** son módulos de software específicos cuya función es abstraer los detalles físicos de un componente de hardware particular, proporcionando al sistema operativo y a las aplicaciones una interfaz estándar para interactuar con él. En Linux, los drivers operan en el espacio del kernel.

En sistemas basados en UNIX, la filosofía subyacente es que "todo es un archivo". Esta filosofía se materializa en el directorio **`/dev`** (devices).

**El directorio `/dev`:**
Contiene archivos especiales de dispositivos (nodos de dispositivo). Estos archivos no almacenan datos en un disco de la manera tradicional; en su lugar, actúan como un canal de comunicación directo hacia el driver en el kernel. Cuando un programa de usuario lee o escribe en un archivo dentro de `/dev`, el kernel intercepta esa operación y transfiere los datos a la función correspondiente del driver.

Al investigar el contenido de `/dev`, se pueden clasificar los dispositivos principalmente en dos tipos:
1.  **Dispositivos de caracteres (Character devices):** Identificados con la letra 'c' en sus permisos (ej. `ls -l /dev/tty`). Transmiten datos de forma secuencial, un byte (carácter) a la vez. Ejemplos: teclados, ratones, puertos serie (`/dev/ttyS0`), y terminales virtuales.
2.  **Dispositivos de bloques (Block devices):** Identificados con la letra 'b'. Permiten leer o escribir datos en bloques de tamaño fijo a la vez (por ejemplo, 512 bytes o 4KB) y suelen admitir acceso aleatorio. Ejemplos: discos duros y particiones (`/dev/sda`, `/dev/nvme0n1`), unidades USB, etc.

Adicionalmente, `/dev` contiene **pseudodispositivos**, que son drivers puramente virtuales gestionados por el kernel, tales como:
* `/dev/null`: Descarta toda la información que se le envía.
* `/dev/zero`: Produce un flujo infinito de ceros (caracteres nulos).
* `/dev/urandom`: Generador de números pseudoaleatorios del kernel.

### 1) ¿Qué diferencias se pueden observar entre los dos modinfo?

La diferencia principal entre las dos ejecuciones del comando `modinfo` radica en la cantidad de metadatos (información descriptiva) que el kernel puede leer del archivo compilado (`.ko`).

* **Primer `modinfo`(modinfo mimodulo.ko):** Corresponde a la compilación del módulo en su versión más básica. Al ejecutar el comando, la salida solo muestra la información técnica mínima generada automáticamente por el compilador y el entorno de construcción (*Kbuild*). Esto incluye campos obligatorios como el nombre del archivo (`filename`), la versión del kernel compatible (`vermagic`), la versión del código fuente (`srcversion`), dependencias (`depends`) y el nombre del módulo (`name`).

* **Segundo `modinfo` (modinfo /lib/modules/$(uname -r)/kernel/crypto/des_generic.ko):** Corresponde a la versión del módulo tras haberle agregado las macros de información en el código fuente en C. La gran diferencia es que la salida ahora incluye campos descriptivos legibles por humanos, los cuales son fundamentales para documentar el módulo. Se añaden explícitamente detalles como:
    * **`author`**: El nombre y/o correo del creador del módulo (generado por `MODULE_AUTHOR`).
    * **`description`**: Una breve explicación de lo que hace el módulo (generado por `MODULE_DESCRIPTION`).
    * **`license`**: El tipo de licencia del código, crucial para que el kernel sepa si el módulo es de código abierto o propietario (generado por `MODULE_LICENSE`).
    * **`version`**: La versión actual del software (generado por `MODULE_VERSION`).

En conclusión, el segundo `modinfo` refleja un módulo correctamente documentado e integrado con los estándares del kernel de Linux, mientras que el primero es solo un binario funcional pero anónimo.

### 2) ¿Qué drivers/módulos están cargados en sus propias PC?

Para visualizar la lista completa de los módulos del kernel y controladores (drivers) que se encuentran actualmente cargados en la memoria del sistema operativo, se utiliza el comando `lsmod` en la terminal.

**Uso del comando y funcionamiento:**
Al ejecutar `lsmod`, el sistema lee e interpreta la información en tiempo real contenida en el archivo virtual `/proc/modules`. La salida se presenta en un formato de tabla con tres columnas principales:

1. **Module:** Indica el nombre del módulo o controlador.
2. **Size:** Representa la cantidad de memoria (en bytes) que el módulo está ocupando en el Kernel Space.
3. **Used by:** Muestra un número que indica cuántas instancias están usando el módulo, seguido de una lista de otros módulos que dependen de él para funcionar.

**Ejemplos de módulos comunes observados:**
Dado que la lista exacta varía dependiendo del hardware físico o de la configuración de la máquina virtual de cada usuario, algunos de los módulos más representativos que se suelen encontrar cargados incluyen:

* **Controladores de video:** `vboxvideo` (en caso de VirtualBox), `nouveau`, `amdgpu` o `nvidia`.
* **Controladores de red:** `e1000` (tarjetas Intel), `iwlwifi` (redes inalámbricas).
* **Gestión de periféricos:** `usbcore` (soporte base para dispositivos USB), `hid_generic` (teclados y ratones estándar).
* **Sistemas de archivos y almacenamiento:** `ext4`, `ahci` (controladores SATA).

*Nota: Para completar esta respuesta de forma empírica en el entorno de trabajo, se ejecutó el comando `lsmod` en la terminal de la máquina virtual, comprobando la presencia de módulos específicos de virtualización y periféricos básicos asignados por el hipervisor.*
