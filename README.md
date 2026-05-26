# TP4-SdC-2026

## 🛡️ Desafío #1: Empaquetado y Seguridad del Kernel

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
