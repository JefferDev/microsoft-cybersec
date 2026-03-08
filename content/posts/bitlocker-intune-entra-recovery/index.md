---
title: "Configurar BitLocker en Intune y recuperar la clave desde Entra ID (Lab completo)"
date: 2026-01-22T10:00:00-05:00
draft: false
categories: ["Intune"]
tags: ["BitLocker", "Entra ID", "Windows 11", "Endpoint Security", "Compliance", "Zero Trust"]
summary: "Laboratorio completo para habilitar BitLocker con Intune (Endpoint Security), validar el cifrado y recuperar las llaves de recuperación desde Microsoft Entra ID."
mermaid: true
ShowToc: true
TocOpen: true
---

> **Nivel:** Intermedio · **Tiempo estimado:** 45–60 min · **Última validación:** Enero 2026

## Objetivo

Implementar BitLocker en dispositivos Windows administrados con Microsoft Intune
(Endpoint Security) y validar el ciclo completo:

- Cifrado silencioso activo en el equipo
- Registro automático de claves de recuperación (escrow)
- Recuperación de la clave desde Microsoft Entra ID
- Troubleshooting con logs y eventos de Windows

## Alcance

**Incluye:**

- Política de BitLocker vía Intune (Endpoint Security → Disk encryption)
- Cifrado silencioso (sin intervención del usuario)
- Validación local en Windows
- Validación en portal (Entra ID)
- Script para forzar backup de recovery key
- Troubleshooting con Event Viewer y comandos

**No incluye:**

- MBAM (Microsoft BitLocker Administration and Monitoring, legado)
- Escenarios de coadministración con SCCM/ConfigMgr
- BitLocker To Go (unidades removibles)

## Arquitectura del Flujo

<pre class="mermaid">
flowchart TD
    A["Administrador Intune"] --> B["Endpoint Security\nDisk Encryption Policy"]
    B -->|"Asignar a grupo"| C["Dispositivo Windows\n(Entra Joined o Hybrid)"]
    C --> D["BitLocker Silent\nEncryption"]
    D --> E["Recovery Key\nEscrow automático"]
    E --> F["Microsoft Entra ID\nDevice Object"]
    F --> G["Admin: Recuperar\nclave en portal"]

    style A fill:#0078d4,color:#fff
    style F fill:#0078d4,color:#fff
</pre>

## Requisitos

### Dispositivo

| Requisito | Detalle |
|---|---|
| Sistema operativo | Windows 10/11 Pro, Enterprise o Education |
| Enrolamiento | Dispositivo inscrito en Intune |
| Join type | Entra Joined (recomendado) o Hybrid Entra Joined |
| TPM | Chip TPM 2.0 disponible y habilitado (recomendado) |
| BIOS/UEFI | Secure Boot habilitado (recomendado) |

### Roles y permisos

**Roles de Entra ID** (cualquiera de estos):

- Intune Administrator
- Security Administrator
- Global Administrator (evitar en producción)

**Roles de Intune** (para operación día a día):

- Endpoint Security Manager — crear y asignar políticas
- Help Desk Operator — ver estado de dispositivos y recuperar claves

### Decisiones previas

Antes de implementar, define el alcance del cifrado:

<pre class="mermaid">
flowchart LR
    A["¿Qué unidades cifrar?"] --> B["Solo SO (C:)\n— Más común"]
    A --> C["SO + Fijas\n— Más restrictivo"]
    A --> D["Removibles\n— BitLocker To Go\n(fuera de alcance)"]

    style B fill:#107c10,color:#fff
</pre>

## Paso a Paso

### Paso 1 — Verificar estado del dispositivo

Abre PowerShell como administrador en el equipo de prueba:

![Ejecutar PowerShell como admin](images/run-pws-admin.png)

Verifica el estado actual de BitLocker:
```powershell
# Estado de BitLocker en todas las unidades
manage-bde -status
```

**Esperado si no está cifrado:** `Conversion Status: Fully Decrypted`

![Estado BitLocker](images/bitlocker-status.png)

Verifica disponibilidad del TPM:
```powershell
# Estado del módulo TPM
Get-Tpm
```

**Esperado:** `TpmReady: True` y `TpmPresent: True`

![Estado TPM](images/get-tpm.png)

> **Nota:** TPM no siempre es obligatorio — depende de la política. Sin TPM, BitLocker
> puede usar contraseña de arranque, pero la experiencia de usuario es peor.

### Paso 2 — Crear política de cifrado en Intune

1. Ingresa a [Microsoft Intune admin center](https://intune.microsoft.com)
2. Navega a: **Endpoint security → Disk encryption**
3. Selecciona **Create policy**
4. Plataforma: **Windows 10 and later**
5. Perfil: **BitLocker**

Configuración base recomendada:

| Setting | Valor recomendado | Notas |
|---|---|---|
| Require device encryption | Yes | Habilita cifrado |
| Allow standard users to enable encryption | Yes | Permite cifrado silencioso |
| Configure encryption method (OS drive) | XTS-AES 256-bit | Máxima seguridad |
| OS drive recovery | Enable | Permite recuperación |
| Save BitLocker recovery info to Entra ID | Yes | Escrow automático de claves |
| Configure pre-boot recovery message | Default recovery message | O personalizar con instrucciones de Service Desk |
| Startup TPM | Require TPM | Para la mejor experiencia |

6. En **Assignments**, asigna a un **grupo de dispositivos piloto** (nunca directamente a todos)

> **Buena práctica:** Crea un grupo dinámico en Entra ID con 3–5 dispositivos de prueba
> para validar antes de masificar.

### Paso 3 — Forzar sincronización en el dispositivo

En el equipo del piloto:

**Opción A** — Desde Settings: