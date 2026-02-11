---
title: "Windows Hello for Business (WHfB) con Cloud Kerberos Trust: implementación y mejores prácticas"
date: 2026-02-10T00:00:00Z
draft: false
tags: ["Microsoft", "Entra ID", "Intune", "Windows Hello for Business", "Kerberos", "Hybrid Identity"]
categories: ["Identidad", "Zero Trust", "Intune"]
summary: "Guía práctica para habilitar WHfB en modo Cloud Kerberos Trust (Microsoft Entra Kerberos) y asegurar SSO a recursos on-premises sin contraseñas."
---

## 5.6.7 Windows Hello for Business (WHfB)

Windows Hello for Business (WHfB) habilita **autenticación sin contraseña** en Windows usando **PIN** y/o **biometría**, respaldado por claves asimétricas (idealmente protegidas por **TPM**). En escenarios híbridos, WHfB puede ofrecer **SSO** tanto a recursos cloud (Microsoft 365, apps SAML/OIDC) como a recursos on-premises que dependan de **Kerberos/NTLM**, siempre que se configure el modelo adecuado de confianza.

En esta publicación del Blog se documenta la implementación de WHfB usando el modelo:

- **Cloud Kerberos Trust** (Microsoft Entra Kerberos)

Este modelo permite a Microsoft Entra ID emitir **TGT parciales** para Active Directory, que luego el cliente “intercambia” con un **Domain Controller** para obtener un TGT completo y acceder a recursos tradicionales on-premises.

> Nota: Este artículo usa “CONTOSO” como nombre de referencia. Ajusta nombres de grupos, políticas y dominios a tu entorno.

---

## 5.6.8 WHfB Cloud Trust configuration (Cloud Kerberos Trust)

### Arquitectura (alto nivel)

El flujo general (simplificado) es:

1. El usuario inicia sesión en Windows con credenciales modernas (WHfB o llave FIDO2) y autentica contra **Microsoft Entra ID**.
2. Entra ID valida que exista una configuración de **Microsoft Entra Kerberos** para el dominio on-premises del usuario.
3. Entra ID emite un **TGT parcial** para el dominio AD.
4. El cliente recibe el **PRT** (token de sesión en Entra) y el TGT parcial.
5. El cliente contacta un **DC on-premises** y canjea el TGT parcial por un **TGT completo**.

![Diagrama de flujo: Entra ID emite un TGT parcial y el cliente lo canjea con un DC on-premises para obtener un TGT completo](./whfb-cloud-trust-diagram.png)

> Recomendación (Hugo): guarda las imágenes en el mismo folder del post (page bundle) y referencia con `./archivo.png`.
>
> Estructura sugerida:
> - `content/posts/whfb-cloud-trust/index.md`
> - `content/posts/whfb-cloud-trust/whfb-cloud-trust-diagram.png`
> - `content/posts/whfb-cloud-trust/whfb-intune-policy.png`
> - `content/posts/whfb-cloud-trust/azureadkerberos-aduc.png`

---

## Prerrequisitos (checklist)

### Sistema y plataforma
- **Windows 10 2004+** (o superior) en los endpoints.
- **Domain Controllers Windows Server 2016+**, con parches al día.
- Política de Kerberos: habilitar **AES256_HMAC_SHA1** si estás restringiendo tipos de cifrado en DCs (GPO “Network security: Configure encryption types allowed for Kerberos”).

### Identidad híbrida
- **Microsoft Entra Connect** sincronizando (mínimo) estos atributos hacia Entra ID:
  - `onPremisesSamAccountName`
  - `onPremisesDomainName`
  - `onPremisesSecurityIdentifier`

### Roles y privilegios (mejor práctica)
- Evitar usar Global Admin para tareas operativas.
- Para la configuración de Kerberos en la nube:
  - **$cloudCred**: usuario con rol **Hybrid Identity Administrator** (o equivalente requerido en tu organización).
  - **$domainCred**: cuenta con permisos **Domain Admin** (y típicamente **Enterprise Admin** si aplica en forest y dominios múltiples).

### Alcance y exclusiones recomendadas
- Iniciar con **piloto por dispositivos** (grupo dedicado).
- Excluir cuentas privilegiadas Tier-0 (Domain Admins/Enterprise Admins) del uso de Cloud Kerberos Trust para acceso a recursos on-premises, salvo que tengas un diseño de administración privilegiada muy controlado.

---

## Paso 1 — Instalar módulo AzureADHybridAuthenticationManagement

Este módulo facilita la administración de escenarios passwordless (FIDO2/WHfB) en híbrido.

Ejecutar PowerShell como **Administrador**:

```powershell
# Recomendado: asegurar TLS 1.2 para acceder a PowerShell Gallery
[Net.ServicePointManager]::SecurityProtocol = `
  [Net.ServicePointManager]::SecurityProtocol -bor `
  [Net.SecurityProtocolType]::Tls12

Install-Module -Name AzureADHybridAuthenticationManagement -AllowClobber
```
## Paso 2 — Crear y publicar el objeto Microsoft Entra Kerberos en AD

La operación crea un objeto de tipo Computer llamado típicamente AzureADKerberos en el dominio, que se comporta conceptualmente como un RODC (sin servidor físico asociado). Este objeto permite que Entra ID genere TGTs para el dominio.


```powershell
# Dominio on-premises (FQDN) donde se creará el objeto Kerberos
$domain = $env:USERDNSDOMAIN

# Credenciales de Entra (rol recomendado: Hybrid Identity Administrator)
$cloudCred = Get-Credential -Message `
  'Usuario de Entra ID con permisos para configurar Hybrid Authentication (ej. Hybrid Identity Administrator).'

# Credenciales on-prem (Domain Admin; y si aplica, Enterprise Admin)
$domainCred = Get-Credential -Message `
  'Usuario de AD miembro de Domain Admins (y Enterprise Admins si aplica).'

# Crea el objeto AzureADKerberos y lo publica a Entra ID
Set-AzureADKerberosServer -Domain $domain -CloudCredential $cloudCred -DomainCredential $domainCred
```

Verificación esperada
Debe aparecer un objeto AzureADKerberos en Active Directory Users and Computers, generalmente en Domain Controllers.
En Entra ID, la configuración queda asociada a la capacidad de emitir TGTs para los dominios configurados.

Advertencias y mejores prácticas de seguridad
No eliminar ni modificar el objeto AzureADKerberos mientras uses Cloud Kerberos Trust.
No relajar la Password Replication Policy (PRP) de AzureADKerberos para permitir cuentas altamente privilegiadas. Mantén controles Tier-0 estrictos.
Mantén patching y hardening de DCs, y monitoreo de eventos Kerberos.

## Paso 3 — Configurar WHfB para Cloud Kerberos Trust en Intune

Para que WHfB use Cloud Kerberos Trust se requieren (mínimo) estas configuraciones:

Use Windows Hello for Business = true

Use Cloud Trust For On Prem Auth = Enabled

Recomendado: Require Security Device = true (usar hardware/TPM cuando aplique)

Opción A: Settings Catalog (recomendado)

Crea una policy en Intune (Settings catalog) y configura:

Categoría	Setting	Valor
Windows Hello for Business	Use Windows Hello For Business	true
Windows Hello for Business	Use Cloud Trust For On Prem Auth	Enabled
Windows Hello for Business	Require Security Device	true

Asigna la policy a un grupo de dispositivos (piloto primero).

Opción B: Custom policy (CSP PassportForWork)

Si prefieres OMA-URI:

OMA-URI: ./Device/Vendor/MSFT/PassportForWork/{TenantId}/Policies/UsePassportForWork
Data type: bool
Value: True

OMA-URI: ./Device/Vendor/MSFT/PassportForWork/{TenantId}/Policies/UseCloudTrustForOnPremAuth
Data type: bool
Value: True

OMA-URI: ./Device/Vendor/MSFT/PassportForWork/{TenantId}/Policies/RequireSecurityDevice
Data type: bool
Value: True


Importante: si aplicas GPO + Intune para WHfB, normalmente GPO tiene precedencia y puede “anular” Intune. Elige una fuente principal para evitar conflictos.


## Recomendación de baseline WHfB (PIN/TPM/Biometría)

Ejemplo de configuración observada en un piloto (Intune profile):

WHfB: Enable

Minimum PIN length: 6

Maximum PIN length: 8

PIN expiration: 60 días

PIN history: 4

PIN recovery: Enable

TPM: Enable

Biometrics: Enable

Enhanced anti-spoofing (si disponible): Enable

Ajustes sugeridos (mejor práctica)

Permitir PIN más largo: mantener mínimo 6–8, pero subir el máximo (8 suele ser bajo; permite que usuarios adopten 10–12+ si quieren).

Evitar expiración de PIN salvo requerimiento normativo: un PIN WHfB no es una contraseña reutilizable; forzar rotación puede empeorar la experiencia y no siempre mejora seguridad.

Mantener TPM requerido y anti-spoofing habilitado cuando exista soporte.

Considerar controles adicionales: Conditional Access, MFA fuerte para el registro inicial, y protección de endpoints (Defender for Endpoint).


## Paso 4 — Experiencia de enrolamiento (qué verá el usuario)

Después del inicio de sesión, si los prerrequisitos pasan:

Si hay biometría compatible, se ofrece configurar rostro/huella (opcional).

Se solicita confirmar uso de Windows Hello con la cuenta corporativa.

Se realiza verificación MFA (para completar el enrolamiento).

Se crea y valida el PIN (sujeto a políticas).

Se genera un par de claves asimétricas (idealmente en TPM) y se registra la pública.

El usuario puede iniciar sesión con PIN/biometría y tener SSO a cloud + on-premises.

Monitoreo y troubleshooting rápido
Validar estado en el endpoint

Ejecuta:

```powershell
dsregcmd /status
```
Revisa estado de:

Entra join / Hybrid join

PRT

Señales de SSO

Logs relevantes

Event Viewer:

Applications and Services Logs > Microsoft > Windows > User Device Registration

Busca eventos relacionados con prerequisitos y provisioning de WHfB.

