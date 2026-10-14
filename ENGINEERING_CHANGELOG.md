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

## Remediación candidata de vulnerabilidades HIGH — 2026-10-13
- Fuente: Trivy JSON artifact 11865160575, ejecución AWX 38498157470. 21 HIGH y 0 CRITICAL.
- Se identificaron vulnerabilidades en Twisted, azure-core, cryptography, dynaconf, jaraco.context (dos ubicaciones/versiones), msgpack, protobuf, pyOpenSSL, pyasn1, urllib3 y wheel. El hallazgo de wheel apunta a 0.45.1 aunque el lock Python apunta a 0.46.3: revisar rutas y capas instaladas.
- Candidatos de prueba en `requirements/requirements.txt`: dynaconf 3.2.13, msgpack 1.2.1, pyasn1 0.6.4, azure-core 1.38.0, jaraco-context 6.1.0, urllib3 2.8.0.
- Cambios SIN validar todavía: resolver dependencias, compilación, pruebas funcionales/API y Trivy posterior. No tocar `prd` ni publicar la imagen.
- Pendientes de análisis de compatibilidad: cryptography (saltos mayores), protobuf, pyOpenSSL, Twisted (versión corregida reportada como release candidate); además cobertura OS CentOS Stream no admitida por Trivy.

## CI run 38500130796: fallo de validación del intérprete Python
- GitHub Actions: build SUCCESS; `pip check` inicial SUCCESS; validación de versiones FAILED; Trivy SKIPPED y sin artifact.
- Error observado: `PackageNotFoundError: No package metadata was found for PyJWT`, porque el workflow usaba el Python del sistema (`python3.12`) y las dependencias AWX se instalan en `/var/lib/awx/venv/awx`.
- Corrección: ejecutar `pip check` y validación de versiones usando `/var/lib/awx/venv/awx/bin/python` dentro de la imagen. Validación aún pendiente de nueva ejecución. `prd` sin cambios.

## Resultado verificado — AWX CI 38501718152 (2026-10-13)
- Compilación Docker, `pip check`, verificación de versiones, Trivy y upload del artifact: **SUCCESS**.
- Informe `awx-trivy-report`, artifact **11865873282**, consultado: **CRITICAL=0, HIGH=9** (frente al lote anterior de 21 HIGH).
- CVE pendientes: Twisted 25.5.0 / CVE-2026-42304 (Trivy indica 26.4.0rc2: pre-release; no actualizar sin probar); cryptography 46.0.3 / CVE-2026-26007 (46.0.5), CVE-2026-69247 (50.0.0), CVE-2026-69249 (49.0.0), GHSA-537c-gmf6-5ccf (48.0.1); jaraco.context 5.3.0 / CVE-2026-23949 (6.1.0) **en otra ubicación pese al pin 6.1.0**; protobuf 6.32.1 / CVE-2026-0994 (6.33.5); pyOpenSSL 25.3.0 / CVE-2026-27459 (26.0.0); wheel 0.45.1 / CVE-2026-24049 (0.46.2) **en otra ubicación pese al pin 0.46.3**.
- Próximos pasos: localizar las capas/intérpretes afectados en el build multietapa, evaluar compatibilidad de actualizaciones mayores, verificar pruebas funcionales y de autenticación antes de publicar; reportar cobertura OS faltante (`centos-stream`).
- No promover a `prd` ni publicar aún.

## Remediación candidata adicional — 2026-10-13
- Tras el run 38501718152 (0 CRITICAL, 9 HIGH), se proponen las actualizaciones **protobuf 6.32.1 → 6.33.5** (CVE-2026-0994) y **pyOpenSSL 25.3.0 → 26.0.0** (CVE-2026-27459).
- Modificados en rama `dev`: `requirements/requirements.in`, `requirements/requirements.txt` y verificación de versiones en `.github/workflows/awx-build.yml`.
- **Estado: no validado**. El nuevo build debe pasar dependencias, `pip check`, versiones efectivas, Trivy y pruebas funcionales. No inferir que ya se resolvieron las CVE.
- Quedan en análisis `cryptography`, `Twisted`, y copias antiguas de `jaraco.context`/`wheel`; Trivy sigue sin cubrir OS CentOS Stream. No promover a `prd` ni publicar.

## CI AWX 38503644183 — validación protobuf y pyOpenSSL (2026-10-13)
- Build, `pip check`, comprobación de versiones, Trivy y artefacto: **SUCCESS**.
- Artefacto `awx-trivy-report`, ID **11867775621**: **0 CRITICAL, 7 HIGH**. Informe JSON descargado y analizado.
- Restantes: Twisted 25.5.0 CVE-2026-42304 (fix indicado 26.4.0rc2); cryptography 46.0.3 CVE-2026-26007 (46.0.5), CVE-2026-69247 (50.0.0), CVE-2026-69249 (49.0.0), GHSA-537c-gmf6-5ccf (48.0.1); jaraco.context 5.3.0 CVE-2026-23949 (6.1.0); wheel 0.45.1 CVE-2026-24049 (0.46.2).
- `jaraco.context` 5.3.0 y `wheel` 0.45.1 coexisten con versiones corregidas declaradas por el lock en otros entornos/capas; identificar ubicaciones reales antes de intervenir.
- WARNING persistente Trivy: `Unsupported os family=centos-stream`. Sin evaluación completa de vulnerabilidades OS.
- Pendiente: priorizar actualización compatible de cryptography; mapear instalaciones antiguas y evaluar pruebas de autenticación/función. **No publicar ni promover a `prd`.**

## Diagnóstico de capas y prueba de cryptography 46.0.5
- Se añadió al workflow AWX un diagnóstico de solo lectura de versiones y ubicaciones *.dist-info de jaraco.context, wheel, cryptography y Twisted dentro de la imagen. Se guardará como artifact awx-python-installations.
- Se cambió cryptography de 46.0.3 a 46.0.5 en requirements.in y requirements.txt; el workflow comprobará la versión efectiva.
- Estado: cambios candidatos SIN validar. Deben aprobar build, pip check, versiones, Trivy y pruebas funcionales antes de publicar o promover. Las vulnerabilidades que requieren cryptography 48/49/50 siguen pendientes.
- Trivy no cubre completamente el OS CentOS Stream. Se mantiene prd sin cambios.

## Validación CI 38504713333 — imagen AWX
- Todos los steps completaron SUCCESS: build, pip check, verificación de versiones, inventario Python, Trivy y subida de artefactos.
- Trivy: 0 CRITICAL, 6 HIGH; aviso persistente `Unsupported os family=centos-stream`. Los seis HIGH aún requieren revisión CVE por CVE.
- Artefactos: `awx-python-installations` ID 11867945854; `awx-trivy-report` ID 11866744977.
- cryptography 46.0.5 instalado y verificado; se redujo un hallazgo HIGH frente a 46.0.3.
- Copias vendorizadas detectadas: `/var/lib/awx/venv/awx/lib/python3.12/site-packages/setuptools/_vendor/wheel-0.45.1.dist-info` y `setuptools/_vendor/jaraco.context-5.3.0.dist-info`; el entorno AWX principal tiene wheel 0.46.3.
- Prohibido eliminar manualmente los archivos vendorizados. Investigar actualización compatible de setuptools, revisar si las copias quedan incluidas en imagen final y evaluar dependencias afectadas. Sin publicación ni promoción a `prd`.

## Prueba controlada de cryptography 50 y pyOpenSSL 26.4 — 2026-10-13
- En `dev` se aplicaron versiones candidatas: `cryptography==50.0.0` y `pyOpenSSL==26.4.0` en requirements.txt; límites correspondientes en requirements.in.
- Motivo: el análisis previo mantenía 6 HIGH y varias CVE de cryptography solo se corrigen en ramas mayores. El changelog oficial de pyOpenSSL 26.4.0 indica soporte para cryptography 50.x; la actualización cambia APIs y requiere pruebas de regresión.
- Workflow ampliado con comprobación de versiones, `pip check`, importación TLS/X509 y firma Ed25519 de prueba, previo a Trivy.
- CANDIDATO NO VALIDADO: esperar nuevo GitHub Actions y resultados Trivy. Compilación e importación no reemplazan pruebas reales de autenticación, certificados, credenciales, receptor, jobs y APIs.
- Persisten los hallazgos por dependencias vendorizadas en setuptools y Twisted; investigar solución sin borrados manuales.
- No modificar `prd`, no publicar imágenes ni desplegar Kubernetes hasta aprobación.

## CI 38505724509 — bloqueo de actualización cryptography 50
- Build FAILED durante resolución de dependencias: `msal==1.34.0` exige `cryptography>=2.5,<49`; prueba `cryptography==50.0.0` causa `ResolutionImpossible` (pip).
- Etapas `pip check`, pruebas TLS/firmas y Trivy se omitieron porque falló el build. Sin nuevo reporte de vulnerabilidades.
- Reversión de versiones candidatas a último conjunto que compiló: `cryptography==46.0.5`, `pyOpenSSL==26.0.0`, y restricciones equivalentes en `requirements.in`; verificación CI adaptada. Se conservaron las pruebas TLS y de firmas en el workflow para validar en la siguiente ejecución.
- Riesgo residual: CVE de cryptography que requieren ramas 48/49/50 no resueltas. Evaluar actualización de MSAL y resto de dependencias de autenticación **separadamente**, con pruebas de regresión. No usar `--no-deps` ni ignorar el resolver.
- Último estado Trivy confirmado sigue siendo 0 CRITICAL / 6 HIGH (run 38504713333), con advertencia `Unsupported os family=centos-stream`. `prd` sin modificaciones.

## CI 38506467521 — fallo del smoke test por quoting
- Build SUCCESS, pip check SUCCESS y control de versiones SUCCESS después del rollback a cryptography 46.0.5 / pyOpenSSL 26.0.0.
- Smoke test TLS/Ed25519 FAILED con `SyntaxError: unexpected character after line continuation character`: escape `\\x27` en la invocación de `python -c`, no fallo demostrado de cryptography.
- Corregido el comando usando el intérprete del virtualenv como entrypoint de Docker y comillas seguras para la cadena de prueba. Aún requiere nueva ejecución para validar.
- Trivy SKIPPED en este run, por tanto último escaneo válido: 0 CRITICAL y 6 HIGH. Sin cambio a prd ni publicación de imágenes.
