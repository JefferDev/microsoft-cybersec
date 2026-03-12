---
title: "BitLocker Silent Encryption + Pre-Boot PIN via Microsoft Intune"
date: 2026-03-11
draft: false
description: "Guia completa para implementar cifrado silencioso BitLocker con PIN pre-boot en entornos empresariales usando Microsoft Intune, PowerShell y scheduled tasks."
tags:
  - BitLocker
  - Intune
  - PowerShell
  - Endpoint Security
  - Windows 11
  - Entra ID
categories:
  - Microsoft Security
  - Endpoint Management
author: "Jefferson Castiblanco"
slug: "bitlocker-silent-encryption-preboot-pin-intune"
cover:
  image: ""
  alt: "BitLocker Silent Encryption + Pre-Boot PIN"
toc: true
---

## El problema

Microsoft Intune permite cifrar discos silenciosamente con BitLocker usando solo TPM. Sin embargo, cuando necesitas agregar un **PIN pre-boot** como capa adicional de autenticacion, Intune no ofrece una forma nativa de solicitarlo al usuario. Si configuras la politica para requerir PIN, el cifrado silencioso deja de funcionar y el usuario debe interactuar con el asistente de BitLocker manualmente.

Esto crea un dilema en entornos empresariales de alta seguridad (banca, gobierno, defensa): necesitas el cifrado inmediato sin friccion **y** la proteccion adicional del PIN pre-boot contra ataques de cold boot y DMA.

## La solucion: arquitectura en dos fases

La solucion es separar el proceso en dos fases con diferentes contextos de ejecucion:

```
FASE 1 (SYSTEM - inmediata)          FASE 2 (USER - al logon)
+--------------------------+         +---------------------------+
| Platform Script / Intune |         | Scheduled Task (wscript)  |
|                          |         |                           |
| 1. Validar TPM           |         | 1. Mostrar dialogo WPF    |
| 2. Cifrar con TPM-only   |         | 2. Validar PIN            |
| 3. Backup recovery key   |         | 3. Guardar pin.dat (DPAPI)|
| 4. Crear scripts locales |         | 4. Confirmar al usuario   |
| 5. Crear scheduled tasks |         +---------------------------+
+--------------------------+                     |
                                                 v
                                   +---------------------------+
                                   | Apply-Pin (SYSTEM, 5 min) |
                                   |                           |
                                   | 1. Detectar pin.dat       |
                                   | 2. Descifrar PIN (DPAPI)  |
                                   | 3. Agregar TpmPin         |
                                   | 4. Remover Tpm-only       |
                                   | 5. Backup recovery key    |
                                   | 6. Auto-limpiar tasks     |
                                   +---------------------------+
```

La comunicacion entre los contextos USER y SYSTEM se realiza a traves de un archivo temporal `pin.dat` cifrado con DPAPI (scope `LocalMachine`), que garantiza que solo procesos en la misma maquina puedan descifrarlo.

## Prerequisitos

Antes de desplegar los scripts, necesitas configurar las politicas de BitLocker en Intune. Sin estas politicas, el cifrado silencioso no funcionara.

### Licenciamiento y dispositivos

- Microsoft Intune Plan 1 (incluido en Microsoft 365 E3/E5, EMS E3/E5)
- Dispositivos Windows 10/11 Pro o Enterprise
- TPM 2.0 habilitado en BIOS/UEFI
- Secure Boot habilitado
- Dispositivos unidos a Entra ID (Azure AD Join o Hybrid Join)

### Perfil de cifrado de disco

Navega a **Endpoint Security > Disk Encryption > Create Policy > Windows > BitLocker**.

Configura los siguientes valores:

**BitLocker - Base Settings:**

| Setting | Valor | Razon |
|---------|-------|-------|
| Require device encryption | Yes | Habilita el cifrado obligatorio |
| Warning for other disk encryption | Disabled | Permite cifrado silencioso sin prompt al usuario |
| Allow standard users to enable encryption during Entra ID Join | Allowed | Permite cifrado sin privilegios admin |
| Configure encryption methods | Enabled | Para especificar XTS-AES 256 |
| Encryption for OS drives | XTS-AES 256 | Estandar recomendado para unidades fijas |

**BitLocker - OS Drive Settings:**

| Setting | Valor | Razon |
|---------|-------|-------|
| Additional authentication at startup | Required | Habilita opciones de TPM + PIN |
| Compatible TPM startup | Allowed | Permite TPM-only (Fase 1 cifra asi) |
| Compatible TPM startup PIN | Allowed | Permite TPM+PIN (Fase 2 lo agrega) |
| Compatible TPM startup key | Blocked | No usamos llave USB |
| Compatible TPM startup key and PIN | Blocked | No usamos llave USB |
| Minimum PIN length | 8 (o segun politica) | Longitud minima del PIN pre-boot |

> **Importante:** `Compatible TPM startup PIN` debe estar en **Allowed**, no en **Required**. Si lo pones en Required, el cifrado silencioso falla porque Intune no puede solicitar el PIN durante el cifrado automatico.

**BitLocker - OS Drive Recovery:**

| Setting | Valor | Razon |
|---------|-------|-------|
| Recovery options in setup wizard | Blocked | Cifrado silencioso |
| Save BitLocker recovery info to Entra ID | Yes | Backup obligatorio antes de cifrar |
| Store recovery info before enabling | Require | No cifrar sin backup |
| Recovery password | Allowed | Clave de 48 digitos |

### Perfil de Settings Catalog (adicional)

Crea un segundo perfil en **Devices > Configuration > Create > Settings Catalog** con estas configuraciones:

| Setting (BitLocker > OS Drives) | Valor | Razon |
|---------------------------------|-------|-------|
| Allow enhanced PINs for Startup | Enabled | Permite PINs alfanumericos si se necesita |
| Enable use of BitLocker authentication requiring preboot keyboard input on slates | Enabled | Soporte para tablets y dispositivos touch |
| Hide prompt about third-party encryption | Enabled | Evita alertas innecesarias |

Asigna ambos perfiles al mismo grupo de dispositivos.

## Los scripts

La solucion consiste en tres scripts para dos metodos de despliegue:

| Script | Proposito | Despliegue |
|--------|-----------|------------|
| `Set-BitLockerStartupPin.ps1` | Cifrado silencioso + crea tasks para PIN | Platform Script |
| `Detect-BitLockerPin.ps1` | Detecta si falta TpmPin | Proactive Remediation (detection) |
| `Remediate-BitLockerPin.ps1` | Cifra + crea tasks para PIN | Proactive Remediation (remediation) |

Puedes usar **uno u otro** metodo de despliegue, no ambos al mismo tiempo sobre los mismos dispositivos.

### Script principal: Set-BitLockerStartupPin.ps1

Este script se despliega como **Platform Script** en Intune. Al ejecutarse como SYSTEM:

1. Valida que el TPM este presente y listo
2. Si el disco no esta cifrado, lo cifra silenciosamente con TPM-only
3. Respalda la recovery key en Entra ID
4. Crea tres archivos locales en `C:\ProgramData\BitLockerPIN\`:
   - `Request-Pin.ps1` - Dialogo WPF que captura el PIN del usuario
   - `Request-Pin.vbs` - Launcher VBS que oculta la ventana de PowerShell
   - `Apply-Pin.ps1` - Script SYSTEM que aplica el protector TpmPin
5. Crea dos scheduled tasks:
   - **BitLocker_RequestPin** - Trigger: at logon, ejecuta como Users (sesion interactiva)
   - **BitLocker_ApplyPin** - Trigger: cada 5 minutos como SYSTEM (polling)

El flujo completo cuando el usuario inicia sesion:

1. `wscript.exe` ejecuta `Request-Pin.vbs` (sin ventana de PowerShell visible)
2. El VBS lanza `Request-Pin.ps1` con `-WindowStyle Hidden`
3. El usuario solo ve el dialogo WPF solicitando el PIN
4. El PIN se guarda cifrado con DPAPI en `pin.dat`
5. En maximo 5 minutos, `Apply-Pin.ps1` (SYSTEM) detecta `pin.dat`
6. Lee el PIN, aplica el protector TpmPin, remueve el Tpm-only, respalda la recovery key
7. Crea `completed.flag` y elimina ambos scheduled tasks

### Despliegue como Platform Script

Navega a **Devices > Scripts and remediations > Platform scripts > Add > Windows 10 and later**.

| Setting | Valor |
|---------|-------|
| Script file | `Set-BitLockerStartupPin.ps1` |
| Run this script using the logged-on credentials | **No** |
| Enforce script signature check | **No** |
| Run script in 64-bit PowerShell host | **Yes** |

> **Critico:** "Run script in 64-bit PowerShell host" **debe ser Yes**. Los modulos de BitLocker (`Get-BitLockerVolume`, `Get-Tpm`) solo existen en PowerShell 64-bit. En 32-bit el script falla silenciosamente.

### Despliegue como Proactive Remediation (alternativo)

Navega a **Devices > Scripts and remediations > Remediations > Create**.

| Setting | Valor |
|---------|-------|
| Detection script file | `Detect-BitLockerPin.ps1` |
| Remediation script file | `Remediate-BitLockerPin.ps1` |
| Run this script using the logged-on credentials | **No** |
| Enforce script signature check | **No** |
| Run script in 64-bit PowerShell | **Yes** |

Schedule: cada 1 hora o diariamente segun la urgencia del despliegue.

## Validacion y troubleshooting

### Verificar estado de BitLocker en el dispositivo

```powershell
# Estado completo
manage-bde -status C:

# Protectores activos (lo mas importante)
(Get-BitLockerVolume -MountPoint C:).KeyProtector | Format-Table KeyProtectorType, KeyProtectorId

# Resultado esperado tras completar ambas fases:
# KeyProtectorType  KeyProtectorId
# ----------------  --------------
# TpmPin            {XXXXXXXX-XXXX-...}
# RecoveryPassword  {YYYYYYYY-YYYY-...}
```

### Verificar logs del script

```powershell
# Log del script principal (Fase 1)
Get-Content "C:\ProgramData\BitLockerPIN\Logs\Set-BitLockerStartupPin_*.log"

# Log de la solicitud de PIN (Fase 2 - usuario)
Get-Content "C:\ProgramData\BitLockerPIN\Logs\RequestPin.log"

# Log de la aplicacion del PIN (Fase 2 - SYSTEM)
Get-Content "C:\ProgramData\BitLockerPIN\Logs\ApplyPin.log"

# Log de Proactive Remediation
Get-Content "C:\ProgramData\BitLockerPIN\Logs\Remediation_*.log"
```

### Verificar scheduled tasks

```powershell
# Ver si existen los tasks (desaparecen tras completarse)
Get-ScheduledTask -TaskName "BitLocker_RequestPin" -ErrorAction SilentlyContinue
Get-ScheduledTask -TaskName "BitLocker_ApplyPin" -ErrorAction SilentlyContinue

# Ver el flag de completado
Test-Path "C:\ProgramData\BitLockerPIN\completed.flag"
```

### Verificar desde Intune

```powershell
# Log de Intune Management Extension
Get-Content "C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log" -Tail 200 |
    Select-String "BitLocker|exitcode"
```

### Ver el script descargado por Intune

```powershell
# Los Platform Scripts se guardan con nombre de GUID
Get-ChildItem "C:\Program Files (x86)\Microsoft Intune Management Extension\Policies\Scripts" -Recurse -Filter *.ps1 |
    Select-String "BitLocker Silent Encryption" -List |
    Select-Object Path
```

### Problemas comunes y soluciones

**"TPM no presente o no listo"**

Causa: El script se ejecuto en PowerShell 32-bit. `Get-Tpm` no existe en `SysWOW64`.

Solucion: Verificar que "Run script in 64-bit PowerShell" este en **Yes** en Intune. Los scripts incluyen un safeguard que detecta el entorno 32-bit y se relanza automaticamente en 64-bit:

```powershell
if ($env:PROCESSOR_ARCHITEW6432 -eq "AMD64") {
    Start-Process "$env:SystemRoot\SysNative\WindowsPowerShell\v1.0\powershell.exe" ...
    exit $LASTEXITCODE
}
```

**"ScriptRequiresElevation"**

Causa: `#Requires -RunAsAdministrator` no es compatible con Intune Management Extension. IME ejecuta como SYSTEM pero sin token UAC elevado explicito.

Solucion: No usar `#Requires -RunAsAdministrator` en ningun script desplegado por Intune. SYSTEM ya tiene todos los privilegios necesarios.

**Caracteres corruptos / "TerminatorExpectedAtEndOfString"**

Causa: IME re-codifica los scripts al descargarlos. Caracteres Unicode (tildes, box-drawing, emojis) se corrompen al pasar de UTF-8 a Windows-1252.

Solucion: Usar exclusivamente caracteres ASCII (0x00-0x7F) en todo el script. Esto incluye comentarios, strings, banners y mensajes de log. Evitar: tildes, enyes, emojis, caracteres box-drawing.

**"Cannot convert System.Object[] to System.Security.SecureString"**

Causa: Contaminacion del pipeline de PowerShell. Comandos como `Add-Type`, `.Focus()`, `.ShowDialog()` producen output que se mezcla con el return value de la funcion.

Solucion: Anteponer `[void]` a todo comando que produzca output dentro de una funcion:

```powershell
[void][System.Reflection.Assembly]::LoadWithPartialName("PresentationFramework")
[void]$txtPin.Focus()
[void]$window.ShowDialog()
```

**"Access denied" en Request-Pin.ps1**

Causa: El script de Request-Pin corre como usuario estandar pero intenta ejecutar cmdlets que requieren admin (`Get-BitLockerVolume`, `Start-ScheduledTask`).

Solucion: Request-Pin.ps1 no debe llamar ningun cmdlet de administrador. Solo muestra el dialogo WPF, valida el PIN y guarda `pin.dat`. La comunicacion con el contexto SYSTEM se hace via archivos:

- `pin.dat` - PIN cifrado con DPAPI (usuario lo crea, SYSTEM lo lee)
- `completed.flag` - Marca de completado (SYSTEM lo crea, usuario lo lee)

**"'x' is an undeclared prefix" en XAML**

Causa: El XAML usa `x:Name` en los controles pero le falta la declaracion del namespace `xmlns:x`.

Solucion: Siempre incluir ambos namespaces en el elemento `<Window>`:

```xml
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        ...>
```

**Ventana de PowerShell visible al solicitar PIN**

Causa: Incluso con `-WindowStyle Hidden`, PowerShell muestra brevemente la ventana de consola antes de ocultarla.

Solucion: Usar un launcher VBS que ejecuta PowerShell con visibilidad 0:

```vbs
Set objShell = CreateObject("WScript.Shell")
objShell.Run "powershell.exe -WindowStyle Hidden -File ""Request-Pin.ps1""", 0, False
```

El scheduled task ejecuta `wscript.exe "Request-Pin.vbs"` en lugar de `powershell.exe` directamente.

**El dialogo de PIN se muestra en loop infinito**

Causa: Apply-Pin nunca se ejecuta correctamente, el PIN nunca se aplica, y Request-Pin sigue pidiendo PIN cada logon.

Solucion: Los scripts usan un sistema de flags basado en archivos:

```
                    +-- completed.flag existe? --> EXIT (ya terminado)
Request-Pin.ps1 ---|
                    +-- pin.dat existe? ---------> EXIT (pendiente de Apply-Pin)
                    |
                    +-- ninguno? ----------------> Mostrar dialogo
```

### Limpiar y reiniciar el despliegue

Si necesitas reiniciar el proceso completo en un dispositivo:

```powershell
# Eliminar tasks
Unregister-ScheduledTask -TaskName "BitLocker_RequestPin" -Confirm:$false -ErrorAction SilentlyContinue
Unregister-ScheduledTask -TaskName "BitLocker_ApplyPin" -Confirm:$false -ErrorAction SilentlyContinue

# Eliminar archivos de trabajo
Remove-Item "C:\ProgramData\BitLockerPIN" -Recurse -Force -ErrorAction SilentlyContinue

# Forzar re-ejecucion del Platform Script
Restart-Service IntuneManagementExtension -Force
```

## Reglas para scripts desplegados por Intune

Estas reglas deben aplicarse a **todo** script de PowerShell desplegado por Intune, no solo a BitLocker:

```powershell
#Requires -Version 5.1
# NO usar: #Requires -RunAsAdministrator (SYSTEM no tiene token UAC)
# NO usar: caracteres fuera de ASCII 0x00-0x7F (encoding se corrompe)
# NO usar: Add-Type sin [void] (contamina pipeline)
# NO usar: Write-Host con emojis ni box-drawing characters

# Safeguard 64-bit (obligatorio si el script usa modulos de sistema)
if ($env:PROCESSOR_ARCHITEW6432 -eq "AMD64") {
    $args64 = "-NoProfile -ExecutionPolicy Bypass -File `"$($MyInvocation.MyCommand.Definition)`""
    Start-Process -FilePath "$env:SystemRoot\SysNative\WindowsPowerShell\v1.0\powershell.exe" `
        -ArgumentList $args64 -Wait -NoNewWindow
    exit $LASTEXITCODE
}
```

## Consideraciones de seguridad empresarial

**Sobre la necesidad real del PIN pre-boot:**

En equipos modernos con TPM 2.0 y Secure Boot, el cifrado con TPM-only ya proporciona un nivel alto de proteccion. El TPM solo libera las claves de cifrado si la configuracion de hardware y software no ha sido alterada. Un atacante necesitaria el equipo completo e intacto para llegar a la pantalla de login de Windows, que esta protegida contra fuerza bruta.

El PIN pre-boot agrega proteccion contra ataques especificos: cold boot (lectura de RAM refrigerada), DMA via puertos Thunderbolt/FireWire, y TPM sniffing (lectura de senales en el bus del TPM). Estos ataques requieren acceso fisico al equipo y herramientas especializadas.

**Recomendaciones para entornos bancarios y de alta seguridad:**

- Configurar BIOS con password de administrador para evitar deshabilitacion de TPM o Secure Boot
- Usar XTS-AES 256 como metodo de cifrado (no AES-CBC)
- Establecer longitud minima de PIN en 8 o mas digitos
- Considerar Enhanced PINs (alfanumericos) solo si la politica lo requiere, ya que el teclado pre-boot puede no soportar layouts internacionales
- Verificar que la recovery key se respalde en Entra ID antes de habilitar el cifrado
- Monitorear compliance via Intune Encryption Report o Proactive Remediations
- Documentar el proceso de recuperacion en caso de PIN olvidado (recovery key via Intune > Device > Recovery Keys)

**Sobre el flujo de PIN temporal vs PIN elegido por usuario:**

Se implemento el flujo de PIN elegido por el usuario por las siguientes razones:

- No se expone informacion del dispositivo (serial number) como PIN temporal
- El usuario establece un PIN que puede memorizar
- No depende de comunicacion adicional para informar el PIN temporal al usuario
- Cumple con politicas de seguridad que prohiben PINs predecibles o basados en datos del dispositivo

## Repositorio y recursos

Los scripts completos estan disponibles en el repositorio del proyecto. La solucion fue desarrollada y probada en entornos con Windows 11 Enterprise 24H2, Intune standalone, y dispositivos unidos a Entra ID con TPM 2.0.

**Inspiracion y referencias de la comunidad:**

- [Oliver Kieselbach - Pre-Boot BitLocker startup PIN with Intune](https://oliverkieselbach.com/2019/08/02/how-to-enable-pre-boot-bitlocker-startup-pin-on-windows-with-intune/) - Solucion original con ServiceUI.exe y Win32 App
- [Katy Nicholson - BitLocker with PIN during Autopilot](https://katystech.blog/mem/bitlocker-with-pin) - Enfoque con serial number como PIN temporal
- [Microsoft Learn - Encrypt devices with BitLocker using Intune](https://learn.microsoft.com/en-us/intune/intune-service/protect/encrypt-devices) - Documentacion oficial

---

*Jefferson Castiblanco - Senior Cybersecurity Consultant | Microsoft Security*