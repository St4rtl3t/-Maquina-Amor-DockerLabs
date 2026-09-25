# Reporte de Intrusión y Análisis de Seguridad de la Maquina Amor en DockerLabs
## [ 1. PASO A PASO / POC (EL "CÓMO SE EXPLOTA") ]

### Fase 1: Reconocimiento y Enumeración Web
Iniciamos comprobando que la conexión a la máquina es exitosa y luego procedemos a lanzar un escaneo con `nmap`.

![Escaneo NMAP](assets/Pasted%20image%2020260924194322.png)

Al ver los resultados, ingresamos al puerto 80 por medio de Firefox y nos muestra información de posibles usuarios del sistema.

![Pasted image 20260924221152.png](assets/Pasted%20image%2020260924221152.png)

> [!NOTE] Desglose de Comandos
> 
> - `nmap`: Herramienta de exploración de red y auditoría de seguridad utilizada para determinar qué puertos están abiertos en la máquina objetivo.
>     

### Fase 2: Ataque de Fuerza Bruta y Acceso Inicial SSH

Con esta información brindada en la web, ejecutamos una herramienta de fuerza bruta (`hydra`) contra el usuario `carlota` para intentar conseguir su contraseña.

![Pasted image 20260924195231.png](assets/Pasted%20image%2020260924195231.png)

> [!NOTE] Desglose de Comandos
> 
> - `hydra`: Herramienta rápida de inicio de sesión por fuerza bruta que soporta múltiples protocolos (en este caso, SSH).
>     

**Resultado obtenido:**
- `[ssh] host: 172.17.0.2 login: carlota password: babygirl`

Ingresamos vía SSH al usuario de Carlota y procedemos a verificar a qué grupos y permisos tiene acceso.

![Pasted image 20260924195516.png](assets/Pasted%20image%2020260924195516.png)

Luego procedemos a usar el comando `cat` sobre el archivo passwd para enumerar más usuarios del sistema.
![Pasted image 20260924195545.png](assets/Pasted%20image%2020260924195545.png)

> [!NOTE] Desglose de Comandos
> 
> - `cat`: Comando de Linux utilizado para concatenar y mostrar el contenido de archivos en la salida estándar.
>     
> - `/etc/passwd`: Archivo del sistema que contiene la información de las cuentas de usuario registradas.
>     

Conseguimos identificar otro usuario en el sistema llamado `oscar`:
`carlota:x:1001:1001::/home/carlota:/bin/sh`  
`oscar:x:1002:1002::/home/oscar:/bin/sh`

----------------------------------------------------------------------------

### Fase 3: Esteganografía y Extracción de Secretos

Indagando entre las carpetas de Carlota, entramos al directorio `Desktop`, encontramos una carpeta llamada `Vacaciones` y dentro una imagen (`imagen.jpg`), la cual analizamos en busca de metadatos o datos ocultos. Utilizando el comando `steghide info`, verificamos si contiene algo:

![Pasted image 20260924201908.png](assets/Pasted%20image%2020260924201908.png)

> [!NOTE] Desglose de Comandos
> 
> - `steghide`: Herramienta de esteganografía que oculta o extrae datos confidenciales en archivos de imagen and audio.
>     
> - `info`: Parámetro que muestra información detallada sobre el contenido oculto en el archivo portador.
>     

Vemos que contiene un archivo `secret.txt`, por lo que procedemos a extraerlo utilizando `steghide`:
`steghide extract -sf imagen.jpg`

![Pasted image 20260924211335.png](assets/Pasted%20image%2020260924211335.png)

> [!NOTE] Desglose de Comandos
> 
> - `extract`: Instrucción para descomprimir o extraer el archivo incrustado dentro de la imagen.
>     
> - `-sf`: Bandera que especifica el archivo portador (_stegofile_).
>     

El archivo extraído nos devuelve un texto cifrado en Base64, por lo que procedemos a descifrarlo mediante una herramienta de decodificación online.

![Pasted image 20260924211532.png](assets/Pasted%20image%2020260924211532.png)

Nos devolvió que la contraseña del usuario Oscar es: `eslacasadepinypon`.

### Fase 4: Escalada de Privilegios y Explotación con Ruby

Ahora que sabemos la contraseña de Oscar, procedemos a iniciar sesión vía SSH con sus credenciales.
![Pasted image 20260924211825.png](assets/Pasted%20image%2020260924211825.png)

Ejecutando los comandos que se aprecian en la imagen, confirmamos que Oscar tiene permisos para ejecutar como `root` el binario de Ruby: `User oscar may run the following commands on 564fdd50fcd5: (ALL) NOPASSWD: /usr/bin/ruby`

Ya volveremos con este dato, ya que primero revisamos los directorios en busca de alguna pista adicional.


![Pasted image 20260924212148.png](assets/Pasted%20image%2020260924212148.png)
Vemos que se indica que en el usuario `root` debemos ir al escritorio a revisar un archivo de texto (`.txt`). 

Volviendo a Ruby, consultamos en la plataforma GTFOBins si existía algún comando para ejecutar como superusuario y elevarnos a `root`. 
El comando es el siguiente:

`ruby -e 'exec "/bin/sh"'`

![Pasted image 20260924212539.png](assets/Pasted%20image%2020260924212539.png)

> [!NOTE] Desglose de Comandos
> 
> - `ruby`: Invoca el intérprete del lenguaje de programación Ruby.
>     
> - `-e`: Bandera que permite evaluar y ejecutar directamente una línea de código fuente pasada como argumento.
>     
> - `'exec "/bin/sh"'`: Instrucción interna para ejecutar una shell interactiva del sistema operativo heredando los privilegios vigentes.
>     

Estando en `root`, vamos a ver el archivo de texto que se nos indicó previamente; para ello nos elevamos empleando `sudo su` y navegamos entre los directorios correspondientes.

Resultado![Pasted image 20260924212800.png](assets/Pasted%20image%2020260924212800.png)

Con esto podemos confirmar que llegamos al fin de la máquina de forma exitosa.

## [ 4. ANÁLISIS DE CAUSA RAÍZ Y CONCEPTOS CLAVE ]

La vulnerabilidad principal que permitió la elevación total de privilegios radica en una **mala configuración de las reglas de sudoers (`/etc/sudoers`)**. Otorgar permisos a un usuario sin privilegios para ejecutar un lenguaje de programación interpretado como `ruby` mediante la directiva `NOPASSWD` facilita la ejecución arbitraria de llamadas al sistema operativo con privilegios de superusuario (`root`).

Asimismo, la filtración de nombres de usuario en servicios web expuestos y el uso de contraseñas débiles propiciaron el compromiso inicial de los sistemas.

## [ 5. RECOMENDACIONES Y REMEDIACIÓN ]

1. **Modificar la política de sudoers:** Remover la regla que permite la ejecución de `/usr/bin/ruby` u otros binarios interactivos por parte de usuarios no privilegiados.
    
2. **Implementar el principio de privilegio mínimo:** Restringir estrictamente el uso de `sudo` solo a herramientas administrativas indispensables y acotadas.
    
3. **Fortalecer las políticas de contraseñas:** Establecer restricciones robustas de complejidad y longitud para las credenciales de los servicios SSH.
    
4. **Ocultación de información sensible:** Evitar la exposición de nombres de usuarios reales en servicios web públicos (puerto 80) y auditar regularmente los directorios de usuario en busca de esteganografía o datos confidenciales.
