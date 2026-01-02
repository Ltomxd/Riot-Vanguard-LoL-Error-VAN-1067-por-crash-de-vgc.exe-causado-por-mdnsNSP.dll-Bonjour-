# Riot-Vanguard-LoL-Error-VAN-1067-por-crash-de-vgc.exe-causado-por-mdnsNSP.dll-Bonjour-
Guía completa para diagnosticar y reparar el error VAN 1067 en League of Legends causado por la caída de Riot Vanguard (vgc) debido a Bonjour/mDNS (mdnsNSP.dll) y Winsock en Windows 11. 

# Fix: Riot Vanguard / LoL crash (VAN 1067) caused by Bonjour / mDNS (mdnsNSP.dll) on Windows 11

> **Síntoma:** League of Legends se cierra (o Vanguard falla) con **VAN 1067**.  
> **Causa real en este caso:** un *Winsock Network Service Provider* (NSP) de **Apple Bonjour / mDNS** (`mdnsNSP.dll`) quedó “colgado” en el catálogo de red y provocaba crash en `vgc.exe` y/o `LeagueClient.exe`.

---

## ✅ Contexto (qué es cada cosa)

### Riot Vanguard
- `vgk` = **driver de kernel** de Vanguard (nivel sistema).
- `vgc` = **servicio en user-mode** (nivel Windows/usuario) que se comunica con el driver y con el juego.
- Si `vgc` se cae o se detiene durante una partida, el juego suele terminar con **VAN 1067**.

### Bonjour / mDNS y `mdnsNSP.dll`
- **Bonjour** (Apple) instala componentes para descubrimiento de dispositivos en red (mDNS).
- En Windows puede instalar un **provider de Winsock** llamado **`mdnsNSP.dll`**.
- A veces, aunque desinstales Bonjour, puede quedar un “resto” registrado en **Winsock catalog** (un “provider huérfano”).
- En este caso, Windows registraba eventos tipo **Application Error (Event 1000)** donde el módulo implicado era:
  - `mdnsNSP.dll_unloaded`
- Eso hacía que procesos que usan red de forma agresiva (como `vgc.exe` o el cliente de LoL) se **crashearan**.

---

## 🔍 Evidencia típica del problema

### 1) Vanguard: driver OK pero servicio `vgc` se detiene
En CMD/PowerShell:

```bat
sc query vgc
sc query vgk
```
Caso típico del problema:

vgk = RUNNING

vgc = STOPPED o se detiene durante el juego
Esto encaja con VAN 1067 (el driver puede estar arriba, pero si el servicio cae, Vanguard falla).

2) Eventos de Windows apuntando a mdnsNSP.dll

En “Visor de eventos” → Windows Logs → Application (Evento 1000):

Aplicación con error: vgc.exe o LeagueClient.exe

Módulo con error: mdnsNSP.dll_unloaded

Código de excepción típico: 0xc0000005

<img width="857" height="480" alt="image" src="https://github.com/user-attachments/assets/bc25e293-c6bf-4593-98f8-5ec4bfd99ca2" />

--Solución  aplicada: eliminar Bonjour/mDNS y limpiar Winsock--


Confirmar si Bonjour existe (registro de desinstalación)
PowerShell:
``Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Where-Object { $_.DisplayName -match "Bonjour|mDNS|Apple" } |
  Select-Object DisplayName, UninstallString
``
Si aparece Bonjour y te da un UninstallString parecido a:

MsiExec.exe /X{B91110FB-33B4-468B-90C2-4D5E8AE3FAE1}
Entonces está instalado (o quedó registrado).



Paso 2 — Desinstalar Bonjour correctamente (GUID con llaves)

Importante: el GUID va con llaves {}, no con corchetes [].
CMD (Administrador):

```msiexec /x {B91110FB-33B4-468B-90C2-4D5E8AE3FAE1} /qn
```

/x = desinstalar

/qn = modo silencioso (sin ventanas)

Nota: en PowerShell, si lo haces “mal” puede abrir ayuda. Recomendado hacerlo desde CMD o con Start-Process en PowerShell.

Paso 3 — Verificar que los servicios Bonjour/mDNS ya no existen

``sc query "Bonjour Service"
sc query mDNSResponder``

Si devuelve ERROR 1060, perfecto:

“El servicio especificado no existe como servicio instalado.

Paso 4 — Confirmar si mdnsNSP.dll sigue en Winsock

``netsh winsock show catalog | findstr /i mdns``

Si aparece mdnsNSP.dll (o algo relacionado a mdns):
✅ confirmas que el provider quedó registrado y puede causar crashes.


Paso 5 — Reset de Winsock (la corrección definitiva)

``netsh winsock reset
netsh int ip reset
ipconfig /flushdns
shutdown /r /t 0
``
Qué hace esto:

winsock reset limpia y reconstruye el catálogo de providers.

int ip reset resetea configuración TCP/IP (por si quedó algo raro).

flushdns limpia caché DNS.

Reinicio para aplicar.


Paso 6 — Verificación final (post-reinicio)

CMD:

``
netsh winsock show catalog | findstr /i mdns
``


Debe no mostrar nada (vacío).

Y verificar Vanguard:

``sc query vgc
sc query vgk
``

Luego probar el juego:

iniciar LoL

entrar a partida

confirmar que VAN 1067 no vuelve a salir


Opcional: asegurar que vgc arranque automáticamente

PowerShell (Administrador):

``sc config vgc start= auto
sc start vgc
sc query vgc
``
Nota: si el problema era mdnsNSP.dll, esto solo es “higiene” para que Vanguard esté levantado antes de abrir el juego.


```✅ Resultado

Tras:

Desinstalar Bonjour

Confirmar que no quedan servicios Bonjour/mDNS

Resetear Winsock para borrar el provider mdnsNSP.dll

Reiniciar

👉 Se resolvió el crash y dejó de aparecer VAN 1067.

🧠 Por qué esto funciona (explicación corta)

mdnsNSP.dll es un Network Service Provider en Winsock.

Si queda inconsistente o “huérfano”, puede provocar errores al inicializar/usar red.

Vanguard (vgc.exe) y LoL usan red desde el inicio; si el stack/NSP falla, el proceso puede crashear.

El winsock reset reconstruye el catálogo y elimina providers rotos.

⚠️ Notas

En Windows 11 moderno, wmic puede no estar disponible (deprecated). Por eso se usó reg query y PowerShell.

Si después del reset pierdes alguna configuración de red (raro), revisa tu DNS / VPN.```
