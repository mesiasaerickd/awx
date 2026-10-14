# Registro de ingeniería y seguridad — AWX

Fecha de apertura: 2026-10-13
Estado: diagnóstico inicial; **sin cambios de código de aplicación ni imágenes construidas**.

## Política de ramas
- `devel`: referencia heredada del proyecto upstream, no modificar.
- `dev`: análisis y cambios sujetos a pruebas.
- `prd`: solo código validado, por ahora idéntico al punto inicial de `devel`.

## Hallazgos iniciales
Se verificó `tools/ansible/roles/dockerfile/templates/Dockerfile.j2`: usa CentOS Stream 9 y Python 3.12 en el builder. `requirements/requirements.in` ya declara Django >=5.2.17,<5.3, GitPython >=3.1.60, Jinja2 >=3.1.6 y sqlparse >=0.6.0. Estas declaraciones son evidencia de intención de actualización, no de una imagen compilada o libre de CVE.

## Decisiones
1. Comparar el código `devel` con la imagen desplegada AWX 24.6.1 antes de cambiar imagen base.
2. No cambiar imágenes de Kubernetes productivo ni tocar el servidor actual.
3. Mantener cambios en PR desde `dev` hacia `prd`, con revisión, SBOM, análisis Trivy, pruebas de construcción y pruebas funcionales.
4. Construir imágenes en CI/runner aislado y publicarlas en un registro OCI, identificadas por SHA256 y versión propia. Ninguna imagen queda aprobada sin pruebas.
5. Evaluar CentOS Stream 9 y UBI 9 por separado: sustituir el sistema base no corrige por sí mismo dependencias Python.

## Historial
- 2026-10-13: creadas ramas `dev` y `prd` desde `devel`; inspección inicial de fuentes y apertura del registro de ingeniería.

## Pendiente
- Verificar árbol de dependencias resueltas, compilar, obtener SBOM y ejecutar Trivy.
- Establecer runners y condiciones de publicación de imágenes.
- Verificar tests, compatibilidad AWX/Operator/EE y recuperación de PostgreSQL.
- Decidir controles y riesgos aceptables antes de solicitar despliegue en producción.

- 2026-10-13: se agregó el workflow de GitHub Actions para compilación de imagen en `dev`, sin despliegue ni publicación al registro. Estado: ejecución y validación pendientes; fallo de build no autoriza cambios automáticos a `prd`.

- 2026-10-13: se extendió GitHub Actions para escanear la imagen compilada con Trivy 0.75.0 (CRITICAL y HIGH), generar un JSON y subirlo como artifact durante 30 días. Los hallazgos no bloquean todavía el build: se requiere establecer baseline, validar límites de cobertura y definir umbrales antes de habilitar gates. Sin push de imágenes ni modificaciones en `prd`.

## Verificación de CI y seguridad — 2026-10-13
Run 38493345027: build y Trivy completados. CRITICAL=1, HIGH=25. Artifact `awx-trivy-report`, ID 11862846497. Referencia original AWX 24.6.1: CRITICAL=6, HIGH=76; no equivale a validación CVE por CVE. Trivy advierte que CentOS Stream 9 no está cubierto como sistema operativo.

Decisión: mantener `prd` sin cambios; no publicar a GHCR hasta identificar CVE remanentes, comprobar cobertura OS y validar la integración Kubernetes. La ejecución de Trivy exitosa significa que el escáner pudo correr, no que no existan vulnerabilidades.

## Corrección candidata — PyJWT (2026-10-13)
- Fuente: artifact `awx-trivy-report` de GitHub Actions run `38493345027`: CVE-2026-102268 CRITICAL, PyJWT 2.10.1, fixed version reportada 2.14.0. Otros hallazgos: 25 HIGH.
- Cambio en `dev`: requisito explícito `pyjwt[crypto]>=2.14.0` en `requirements/requirements.in` y pin `pyjwt[crypto]==2.14.0` en `requirements/requirements.txt`.
- Advertencia: esta actualización es una **candidata**, todavía no se ha comprobado disponibilidad del paquete, resolución del resto de dependencias, build, pruebas de JWT/autenticación, compatibilidad de AWX, ni escaneo de la nueva imagen.
- Aceptación: compilar correctamente en GitHub Actions, verificar la versión efectiva de PyJWT, comprobar que desaparece la CVE, ejecutar pruebas de autenticación/API y someter cambio a PR; prohibida la promoción a `prd` hasta entonces.
- La base de CentOS Stream 9 sigue sin cobertura de CVE del sistema operativo en Trivy, por lo que un escaneo limpio en Python no constituye aprobación de seguridad.

## Verificación corrección PyJWT — GitHub Actions 38498157470
- Job build 115639674023: **SUCCESS**. Instalación observada en logs: `pyjwt-2.14.0`.
- Escaneo Trivy 0.75.0: **CRITICAL=0, HIGH=21**; comparativa inicial 6 CRITICAL / 76 HIGH y previa de desarrollo 1 CRITICAL / 25 HIGH.
- Artifact de informe `awx-trivy-report`, ID 11865160575.
- La CVE-2026-102268 ya no figura como crítica en el conteo; se necesita cotejo detallado del JSON para atribuir todas las diferencias de hallazgos.
- Cobertura incompleta: Trivy muestra `Unsupported os family="centos-stream"`.
- PENDIENTE: pruebas de autenticación/API, integración AWX-Operator-EE, revisión de hallazgos HIGH y examen de paquetes del sistema operativo. No publicar ni promover a `prd` todavía.
