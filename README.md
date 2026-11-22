# Windows-Domain-Lab-Active-Directory-DNS-DHCP-y-GPO

Laboratorio virtualizado de un dominio Windows con administración centralizada de usuarios, equipos y políticas de seguridad. Lo realicé como práctica integradora de la asignatura Seguridad de Sistemas Operativos en el Instituto Tecnológico de Las Américas (ITLA).

Autor: Axel Silvestre
Herramientas: VMware Workstation, Windows Server (con experiencia de escritorio), Windows 11, Active Directory, DNS, DHCP, Directivas de grupo (GPO), PowerShell y CMD.

Objetivo

Implementar una infraestructura de dominio que permita administrar de forma centralizada usuarios, equipos, recursos compartidos y políticas de seguridad, aplicando el principio de mínimo privilegio y comprobando que los controles funcionan (incluyendo pruebas de rechazo de acciones no autorizadas).

Escenario

Una organización con cinco direcciones: Tecnología, Administrativa, Legal, Comunicaciones y Gestión Humana. Como administrador de sistemas, implementé el dominio, organicé las cuentas y controlé el acceso a la información.

1. Infraestructura del laboratorio
Equipo	Sistema operativo	Función	Direccionamiento
SRV-DC01	Windows Server	Controlador de dominio: AD DS, DNS y DHCP	IP estática
Windows 11 (cliente)	Windows 11	Estación de trabajo del dominio	DHCP
Red virtual
Parámetro	Valor
Red del laboratorio	192.168.50.0/24 (VMnet8, NAT)
SRV-DC01	192.168.50.10 (estática, fuera del rango DHCP)
Rango DHCP	192.168.50.100 – 192.168.50.150
DNS entregado a los clientes	192.168.50.10
Puerta de enlace	192.168.50.2 (dispositivo NAT de VMware)
Reenviadores DNS	8.8.8.8 y 1.1.1.1

Decisión importante: en el Editor de red virtual de VMware deshabilité la casilla "Usar el servicio DHCP local para distribuir la dirección IP a las máquinas virtuales". Así el DHCP del hipervisor no compite con el DHCP de Windows (se evita un rogue DHCP) y los clientes obtienen su configuración solo del servidor autorizado en Active Directory.

Editor de red virtual

La máquina virtual se conecta en modo NAT (VMnet8), lo que permite la comunicación privada entre el controlador de dominio y los clientes, y da salida controlada a Internet para descargas y actualizaciones.

Adaptador de red de la VM

Configuración IP del controlador de dominio
Parámetro	Valor
Dirección IPv4	192.168.50.10
Máscara	255.255.255.0 (/24)
Puerta de enlace	192.168.50.2
DNS preferido	192.168.50.10 (el propio servidor)
DHCP habilitado	No (configuración estática)

Configuración TCP/IPv4 Verificación con ipconfig /all

2. Roles y promoción a controlador de dominio

Desde el Administrador del servidor → Agregar roles y características instalé:

Servicios de dominio de Active Directory (AD DS): gestión centralizada de identidades, usuarios y equipos.
Servidor DHCP: distribución dinámica de direcciones IP.
Servidor DNS: resolución de nombres del dominio y de la infraestructura local.

Selección de roles

Tareas posteriores a la instalación:

Promover el servidor a controlador de dominio → Agregar un nuevo bosque.
Completar la configuración de DHCP → autorizar el servidor en Active Directory.

Notificaciones post-implementación Nuevo bosque

Parámetro del dominio	Valor
Nombre del dominio (FQDN)	Axel.Silvestre.SVR01
Nombre del servidor	ASILVESTRELOCAL
Nivel funcional (bosque y dominio)	Windows Server 2016
Servicios activados	Servidor DNS y Catálogo global (GC)
Contraseña DSRM	Definida durante la promoción (no se documenta)

Opciones del controlador de dominio

DNS: reenviadores

En el Administrador de DNS configuré los reenviadores 8.8.8.8 y 1.1.1.1 para resolver nombres externos desde el servidor DNS interno.

Reenviadores DNS

DHCP

El servicio quedó autorizado en Active Directory y en ejecución (marca verde sobre IPv4), con el ámbito 192.168.50.0 creado para los clientes del laboratorio.

Consola DHCP

3. Organización de Active Directory

Creé las cinco unidades organizativas departamentales y ubiqué los usuarios en cada una:

Dirección de Tecnología
Dirección Administrativa
Dirección Legal
Dirección de Comunicaciones
Dirección de Gestión Humana

Además existen una OU técnica (Empresa_Lab) y grupos de seguridad por dirección. Los usuarios de Tecnología no reciben privilegios de administrador de dominio solo por pertenecer a su dirección; las cuentas de prueba son cuentas estándar.

Estructura de OU y usuarios Listado de usuarios con PowerShell

4. Unión de equipos al dominio y DHCP en el cliente

El cliente Windows 11 recibió su configuración del servidor DHCP de Windows:

Parámetro	Valor
Sufijo DNS	Axel.Silvestre.SVR01
DHCP habilitado	Sí
Dirección IPv4	192.168.50.100
Servidor DHCP	192.168.50.10
Servidor DNS	192.168.50.10

ipconfig /all en el cliente

Pasos para unir el equipo al dominio:

Win + R → sysdm.cpl → pestaña Nombre de equipo → Cambiar.
Seleccionar Dominio e ingresar el FQDN del dominio.
Autenticarse con una cuenta con permisos para unir equipos.
Reiniciar para que se aplique la relación de confianza.

Resultado: "Se unió correctamente al dominio".

Unión al dominio

5. Políticas de usuario con GPO

Creé la GPO GPO_Restricciones (Configuración de usuario) y la vinculé a las OU de Administrativa, Legal, Comunicaciones y Gestión Humana. Tecnología queda fuera de las restricciones.

Política	Ruta	Estado
Impedir el acceso al símbolo del sistema	Plantillas administrativas → Sistema	Habilitada
Quitar el menú Ejecutar del menú Inicio	Plantillas administrativas → Menú Inicio y barra de tareas	Habilitada
Prohibir el acceso a Configuración de PC y Panel de control	Plantillas administrativas → Panel de control	Habilitada
Tapiz del escritorio	Plantillas administrativas → Active Desktop	Habilitada, desde un recurso compartido (UNC)
Impedir cambiar el fondo de pantalla	Panel de control → Personalización	Habilitada

Impedir acceso a CMD Quitar Ejecutar y bloquear Panel de control Vinculación de la GPO a las OU

Fondo de pantalla institucional

El fondo se guarda en una carpeta compartida (Compartido) del servidor con permiso de solo lectura, y la GPO apunta a esa ruta UNC. Los usuarios pueden leer la imagen pero no modificarla.

Carpeta compartida GPO del tapiz Impedir cambiar el fondo

6. Política de contraseñas y bloqueo de cuentas

Configuré la política de cuentas en Default Domain Policy (ámbito de dominio), en Configuración del equipo → Configuración de seguridad → Directivas de cuenta:

Configuración	Valor
Longitud mínima de contraseña	14 caracteres
Complejidad	Habilitada
Historial de contraseñas	10
Cifrado reversible	Deshabilitado
Vigencia máxima / mínima	42 días / 1 día
Umbral de bloqueo	3 intentos fallidos
Duración del bloqueo	15 minutos
Restablecer contador de bloqueo	15 minutos

Directiva de contraseñas Directiva de bloqueo

Efecto de un umbral de solo 3 intentos

Un umbral bajo aumenta la seguridad frente a ataques de fuerza bruta, pero afecta la disponibilidad: los usuarios legítimos pueden bloquearse con facilidad por un error al escribir, lo que genera más carga para el Help Desk. Además, un atacante podría provocar bloqueos masivos a propósito (denegación de servicio sobre las cuentas).

7. Sincronización horaria (NTP)

Configuré el controlador de dominio para consultar una fuente NTP externa:

powershell
w32tm /config /manualpeerlist:"pool.ntp.org,0x9" /syncfromflags:manual /reliable:YES /update

Si el servidor no se anuncia como fuente confiable, se puede forzar desde el registro:

cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Services\W32Time\Config" /v AnnounceFlags /t REG_DWORD /d 5 /f

Los equipos miembros siguen la jerarquía horaria del dominio.

Configuración NTP

8. Pruebas realizadas
Prueba	Resultado esperado	Resultado
Renovar la IP de una estación	Concesión del DHCP de Windows	Cliente con 192.168.50.100 entregada por 192.168.50.10
Iniciar sesión con un usuario del dominio	Autenticación correcta	Correcto
Abrir CMD con un usuario restringido	Bloqueo por GPO	Mensaje: "El administrador ha deshabilitado el símbolo del sistema"
gpresult con un usuario de Administrativa	GPO aplicada	GPO_Restricciones listada como aplicada
Iniciar sesión con un usuario de Tecnología	Excepción de GPO	CMD abre normalmente
Fallar tres autenticaciones	Bloqueo de la cuenta de prueba	"La cuenta a que se hace referencia está bloqueada y no se puede utilizar"

Cuenta bloqueada CMD bloqueado por GPO

Verificación con gpresult
cmd
gpresult /h C:\Users\<usuario>\Desktop\Reporte_GPO.html

El informe confirma el usuario, el dominio, la OU (Dirección Administrativa) y que GPO_Restricciones aparece en GPO aplicados.

Informe gpresult

Excepción de Tecnología

Con un usuario de la Dirección de Tecnología, cmd.exe abre sin mensaje de bloqueo, porque GPO_Restricciones no está vinculada a esa OU. Esto demuestra el diseño de herencia y exclusión de la GPO.

CMD permitido en Tecnología Renovación de IP

9. Incidencias y soluciones
Contraseña del Administrador local durante la promoción a DC: para evitar errores, establecí una contraseña robusta en la cuenta de Administrador local antes de promover el servidor, con net user Administrador * (el * pide la contraseña sin dejarla escrita en el comando).
DHCP del hipervisor vs. DHCP de Windows: se resolvió deshabilitando el DHCP local de VMware en la red del laboratorio.
Fuente de hora no confiable: se forzó el anuncio del servidor como fuente confiable con AnnounceFlags.
10. Limitaciones y mejoras
Restringir CMD y Ejecutar no impide toda ejecución de programas o scripts. Son controles de endurecimiento, no una barrera de seguridad absoluta. Para algo más robusto se usaría AppLocker o WDAC.
El recurso compartido del fondo usa permiso de lectura para Todos. Una mejora sería limitarlo a Usuarios autenticados y proteger la escritura con permisos NTFS.
El recurso compartido está en el controlador de dominio. En un entorno real se alojaría en un servidor de archivos aparte.
Pendientes de ampliar en este repositorio: incorporación de más equipos al dominio (servidor de archivos y segunda estación), matriz de grupos y permisos, y las consultas w32tm /query /source y /status como evidencia de la jerarquía horaria.
Qué aprendí
Cómo se relacionan AD DS, DNS y DHCP, y por qué el DNS interno es crítico para el dominio.
La diferencia entre políticas de usuario y de equipo, y dónde debe aplicarse la política de cuentas (a nivel de dominio).
Cómo verificar que una GPO se aplica con gpresult, y no solo configurarla.
El equilibrio entre seguridad y disponibilidad en el bloqueo de cuentas.
