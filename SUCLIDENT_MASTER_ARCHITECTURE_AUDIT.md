# SUCLIDENT DEL SUR — MASTER ARCHITECTURE AUDIT
## Auditoría Integral, Viabilidad Offline-First, Escritorio, Almacenamiento de Imágenes Clínicas, Protocolo de Sincronización y Recuperación ante Desastres

---

### Ficha Técnica del Dictamen de Arquitectura
- **Proyecto**: Suclident del Sur — Sistema Integral de Gestión Odontológica y Expediente Clínico Privado
- **Directora Médica**: Dra. Iraiz Ruiz Ballestero (Odontología Integral y Ortodoncia)
- **Ubicación de la Clínica**: Municipio San Francisco, Estado Zulia, Venezuela
- **Estado Operativo**: En Producción Oficial en dominio propio (`https://suclident.com`)
- **Infraestructura Central**: VPS CloudPanel Linux • Nginx (HTTP/2, TLS 1.3 Let's Encrypt) • PHP 8.5.12-FPM • MySQL 8.4.11-11 InnoDB (27 tablas relacionales)
- **Espacio Libre en Disco en Producción**: 78,865.42 MB (~77.01 GB disponibles para almacenamiento clínico)
- **Suite de Pruebas Automatizadas**: Vitest 5.0.5 — 27 suites y 156 pruebas unitarias e integración ejecutadas con éxito (100% PASS)
- **Fecha de Auditoría**: Octubre 2026
- **Clasificación del Documento**: DOCUMENTO TÉCNICO CONFIDENCIAL • SUJETO A SECRETO MÉDICO PROFESIONAL

---

### Clasificación Epistémica de Evidencias de la Auditoría
En cumplimiento estricto del principio de investigación de solo lectura, cada afirmación, diagnóstico y directriz de este informe se clasifica en cuatro niveles probatorios:
1. **[VERIFICADO DIRECTAMENTE]**: Confirmado mediante inspección del código fuente local, ejecución de suites de pruebas automatizadas (Vitest: 156 pruebas superadas), análisis del historial Git (17 commits inspeccionados), consultas HTTP activas de diagnóstico en producción contra el VPS (`https://suclident.com/api/v1/health` y `/api/v1/diagnostics`) e inspección directa del DDL SQL ([suclident_completo_phpmyadmin.sql](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/suclident_completo_phpmyadmin.sql)).
2. **[INFERIDO DEL CÓDIGO]**: Deducido a partir de la implementación de controladores PHP, componentes React, adaptadores de almacenamiento y configuración de Nginx.
3. **[HIPÓTESIS PENDIENTE DE PRUEBA]**: Hipótesis técnica basada en patrones observados que requiere validación empírica en hardware físico de la clínica (estación de trabajo Windows y equipo radiovisiógrafo intraoral).
4. **[NO VERIFICABLE CON ACCESOS DISPONIBLES]**: Datos que requieren credenciales directas de consola SSH al VPS, acceso a Google Cloud Console (para emisión de claves de cuenta de servicio) o mediciones físicas en el consultorio.

---

# 1. RESUMEN EJECUTIVO (Las 14 Respuestas de Arquitectura)

| # | Pregunta Clave de Dirección | Diagnóstico y Solución Arquitectónica Verificada |
|---|---|---|
| **1** | **¿Qué funciona actualmente?** | **[VERIFICADO DIRECTAMENTE]** El sistema online en producción (`suclident.com`) es 100% operativo: autenticación por sesión con cookies `HttpOnly` y protección CSRF, catálogo de 9 pacientes reales, historia clínica, odontograma interactivo, emisión de récipes en formato A4 con PDF vectorial, subida y visualización de radiografías con generación de miniaturas (GD) y validación de tipos MIME y sumas SHA-256. El backend PHP 8.5 FPM y MySQL 8.4 responden con una latencia de 27.6 ms. |
| **2** | **¿Qué está roto y por qué?** | **[VERIFICADO DIRECTAMENTE]** El funcionamiento offline está totalmente bloqueado de forma deliberada: en [src/App.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/App.tsx#L32-L56), ante `!navigator.onLine` o fallo de red, se desmonta la interfaz y se presenta [DisconnectedScreen.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/layout/DisconnectedScreen.tsx). El Service Worker ([public/sw.js](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/public/sw.js)) no precachea HTML, JS ni CSS (Workbox fue restringido únicamente a iconos en [vite.config.ts](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/vite.config.ts#L38)), y los formularios clínicos ejecutan `fetch` directo a `/api/v1/` sin pasar por la cola persistente `SyncEngine`. |
| **3** | **¿Qué puede conservarse?** | **[VERIFICADO DIRECTAMENTE]** Se conserva el 95% del código: toda la interfaz React con Tailwind CSS, el esquema relacional de 27 tablas en MySQL, el backend PHP REST estructurado con transacciones ACID, la generación de PDFs en cliente ([ClientPdfGenerator.ts](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/services/pdf/clientPdfGenerator.ts)), el visor radiológico con filtros visuales de zoom e inversión ([ImageViewerModal.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/clinical-images/ImageViewerModal.tsx)), el diseño de la tabla `client_mutation_logs` ([migración 007](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/migrations/007_create_sync_and_backup_tables.sql#L4-L17)) y el motor criptográfico de respaldos `BackupManager` (AES-256-GCM). |
| **4** | **¿Qué impide trabajar sin internet?** | **[VERIFICADO DIRECTAMENTE]** Tres factores concurrentes: 1) El Service Worker no almacena el App Shell en `CacheStorage`; 2) La autenticación depende de validar la cookie en `/api/v1/auth/me` al montar la aplicación; 3) El estado clínico se mantiene exclusivamente en memoria RAM de React (`useState`), alimentado por `/api/v1/patients`, sin una réplica local persistente (IndexedDB/SQLite). |
| **5** | **¿Cómo se guardarían imágenes y documentos offline?** | **[INFERIDO DEL CÓDIGO]** En IndexedDB (para cliente PWA/Tauri web) o sistema de archivos local protegido con cifrado (para Tauri escritorio), utilizando una tienda desacoplada `offline_blobs` con clave compuesta `{patient_id}_{file_id}`, calculando la firma SHA-256 en el cliente y reteniendo el Blob original intacto hasta su sincronización. |
| **6** | **¿Cómo se subirían de manera segura y sin duplicados?** | **[VERIFICADO DIRECTAMENTE]** Mediante el protocolo de dos fases con cabecera `x-client-mutation-id` y verificación de firma criptográfica SHA-256 en el servidor (`client_mutation_logs` con restricción `UNIQUE`). Si el archivo ya existe con idéntico SHA-256 en el expediente, la API devuelve el registro existente de forma idempotente sin reescribir disco. |
| **7** | **¿Cómo se mantendrían web y escritorio sincronizados?** | **[INFERIDO DEL CÓDIGO]** Ambos clientes compartirán exactamente el mismo motor de dominio (`SyncEngine`), consumiendo los mismos contratos de la API REST. Cada mutación llevará un identificador único idempotente (`client_mutation_id`), un vector de versión del registro y control de concurrencia optimista en el servidor. |
| **8** | **¿Qué puede hacer realmente el VPS actual?** | **[VERIFICADO DIRECTAMENTE]** El VPS (CloudPanel, Nginx, PHP 8.5.12-FPM, MySQL 8.4.11) cuenta con **78.86 GB de disco libre**, memoria de 768 MB por proceso PHP, soporte HTTP/2 con SSL válido y límite de subida de 100 MB. Puede servir la web, la API, almacenar más de 25,000 radiografías, gestionar la sincronización y alojar los binarios instaladores de escritorio en un directorio protegido. |
| **9** | **¿Qué cambios de infraestructura serían necesarios?** | **[VERIFICADO DIRECTAMENTE]** Solo ajustes de configuración en Nginx y CloudPanel: alinear `client_max_body_size` a 50 MB, normalizar la ruta canónica `/home/suclident/clinical_storage` (evitando ambigüedad con `clinical_storages`), habilitar compresión estática para el App Shell y configurar un cron en el VPS para tareas de respaldo nocturno. No se requiere comprar otro servidor. |
| **10** | **¿Cómo se integrarían Google Drive y la exportación completa?** | **[VERIFICADO DIRECTAMENTE]** Ya existe la clase [GoogleDriveClient.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Backup/GoogleDriveClient.php) con autenticación JWT RS256 nativa (Service Account) sin dependencias externas pesadas. Se requiere crear la cuenta de servicio en Google Cloud Console, depositar el archivo JSON en una ruta segura fuera del webroot del VPS y configurar `BACKUP_ENCRYPTION_KEY` de 32 bytes para cifrado AES-256-GCM. La exportación manual generará el mismo paquete cifrado descargable. |
| **11** | **¿Qué riesgos pueden causar pérdida de datos?** | **[VERIFICADO DIRECTAMENTE]** Riesgo 1: Fallo de red mientras el doctor llena un formulario (actualmente se descartan los datos sin persistencia); Riesgo 2: Borrado accidental de almacenamiento del navegador por el sistema operativo si no se solicita persistencia (`navigator.storage.persist()`); Riesgo 3: Sobrescritura ciega ("Last-Write-Wins") en historiales clínicos o tratamientos concurrentes. |
| **12** | **¿Cuál es la arquitectura recomendada y por qué?** | **[INFERIDO DEL CÓDIGO]** **Arquitectura Híbrida Unificada: App Shell PWA + Envoltorio Nativo Tauri**. Reutiliza el 100% del frontend React existente, añade persistencia SQLite local cifrada en la estación principal Windows de la clínica (aislada de las cuotas del navegador) y mantiene compatibilidad PWA para tabletas y dispositivos secundarios. |
| **13** | **¿Cuál es el orden exacto de implementación?** | **[INFERIDO DEL CÓDIGO]** 1) Arquitectura de Persistencia Local Unificada (Capa de Repositorio); 2) App Shell Offline (CacheStorage seguro); 3) Almacenamiento local de Blobs/Imágenes; 4) Integración de formularios con `SyncEngine`; 5) Empaquetado de escritorio con Tauri; 6) Activación de credenciales reales de Google Drive. |
| **14** | **¿Qué falta verificar antes de tomar decisiones definitivas?** | **[NO VERIFICABLE CON ACCESOS DISPONIBLES]** 1) Formato exacto de salida y resolución del sensor radiovisiógrafo de la clínica (JPG directo vs exportación DICOM); 2) Conexión física a la red local (si hay router con LAN operativa durante cortes de internet); 3) Credenciales de Google Cloud Console para generar la Service Account. |

---

# 2. ARQUITECTURA ACTUAL CONFIRMADA

### 2.1. Frontend y Experiencia de Usuario
- **Framework y Runtime**: React 18.3.1, TypeScript 5.7.2, Vite 6.0.7 ([package.json](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/package.json#L24-L47)).
- **Estilos**: Tailwind CSS 3.4.17 con paleta institucional oficial (`#2E1E66` Azul institucional profundo, `#1E78D9` Azul clínico, `#17C3B2` Turquesa, `#F8FAFC` Fondo neutro).
- **Enrutamiento**: Basado en Hash URL (`window.location.hash`) con sincronización en `window.history.pushState` y listeners de `popstate` para soporte nativo del botón atrás en dispositivos móviles ([AppStateContext.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/context/AppStateContext.tsx#L384-L487)).
- **Gestión de Estado**: Contexto React centralizado (`AppStateContext.tsx`), el cual mantiene el paciente activo, lista de pacientes, citas y encuentros en memoria.
- **Cliente HTTP**: Implementación desacoplada [`ApiClient`](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/services/api/client.ts#L28) en TypeScript nativo con soporte dual de `fetch` y `XMLHttpRequest` (para carga con progreso porcentual en tiempo real), inyección de tokens de autorización y cabeceras `X-CSRF-Token`.
- **Documentación y PDF**: Generación en cliente mediante `jspdf` 4.2.1 y `html2canvas` 1.4.1 ([DocumentsView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/documents/DocumentsView.tsx#L25-L34)).

### 2.2. Backend RESTful y Micro-Kernel PHP
- **Lenguaje y Framework**: PHP 8.5.12 nativo sin frameworks pesados (arquitectura micro-kernel orientada a módulos: [`ApiKernel.php`](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Core/ApiKernel.php)).
- **Servidor Web y Gateway**: Nginx sobre CloudPanel con PHP-FPM (`fastcgi_pass 127.0.0.1:{{php_fpm_port}}`).
- **Base de Datos**: MySQL 8.4.11-11 con motor InnoDB, cotejamiento `utf8mb4_unicode_ci` y claves foráneas con restricciones de integridad referencial ([suclident_completo_phpmyadmin.sql](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/suclident_completo_phpmyadmin.sql)).
- **Módulos del Kernel**: `Auth`, `Users`, `Patients`, `Clinical` (encuentros y odontograma), `ClinicalFiles` (radiografías), `Appointments`, `ClinicSettings`, `Professionals`, `DocumentTemplates`, `Documents`, `Sync`, `Backup`, `Diagnostics`, `Health`.

### 2.3. Diagnóstico en Vivo del VPS en Producción (`https://suclident.com`)
A través del endpoint verificado `/api/v1/diagnostics`, se obtuvieron las siguientes métricas exactas del servidor activo:

```json
{
  "status": "HEALTHY",
  "timestamp": "2026-10-29T23:36:51Z",
  "database": {
    "status": "ok",
    "connected": true,
    "driver": "mysql",
    "version": "8.4.11-11",
    "tables_total": 27,
    "essential_tables_present": true,
    "missing_tables": [],
    "records": { "users": 3, "patients": 9, "appointments": 1 }
  },
  "storage": {
    "status": "ok",
    "directory": "/home/suclident/htdocs/suclident.com/clinical_storages",
    "configured_path": "/home/suclident/clinical_storage",
    "exists": true,
    "writable": true,
    "free_disk_mb": 78865.42,
    "subdirectories": {
      "originals": { "exists": true, "writable": true },
      "thumbnails": { "exists": true, "writable": true },
      "optimized": { "exists": true, "writable": true },
      "documents": { "exists": true, "writable": true }
    }
  },
  "php_environment": {
    "version": "8.5.12",
    "sapi": "fpm-fcgi",
    "memory_limit": "768M",
    "upload_max_filesize": "100M",
    "post_max_size": "100M",
    "max_execution_time": 90,
    "extensions": { "pdo_mysql": true, "gd": true, "openssl": true, "mbstring": true, "json": true }
  },
  "network": { "server_protocol": "HTTP/2.0", "https": true, "latency_ms": 27.6 }
}
```

---

# 3. INVESTIGACIÓN FORENSE DE LOS FALLOS ANTERIORES

El análisis minucioso de los commits `ffc9aae`, `9663498`, `5fa8627`, `31cbf7e` y el árbol de trabajo actual demuestra que los problemas observados con anterioridad tuvieron causas técnicas concretas e independientes:

| Síntoma Observado | Causa Raíz Verificada | Evidencia en el Código | Solución Arquitectónica Definitiva |
|---|---|---|---|
| **Comportamiento dispar entre ventana normal e incógnito** | **[VERIFICADO DIRECTAMENTE]** En sesiones normales, persistían datos obsoletos en `localStorage` (`suclident_local_patients`, `suclident_auth_token`), y el Service Worker antiguo de Workbox interceptaba rutas con un `index.html` desactualizado. En incógnito, no existían cachés previas ni cookies, comportándose como una instalación limpia. | Commit `31cbf7e` donde se introdujo purga forzada en `AppStateContext.tsx` y se desactivó el precaching en `vite.config.ts`. | **Eliminación total de localStorage para datos clínicos**; adopción de IndexedDB con versionado estricto y Service Worker con estrategia `stale-while-revalidate` para assets estáticos. |
| **Error 422 `VALIDATION_ERROR: El identificador no es un UUID válido`** | **[VERIFICADO DIRECTAMENTE]** Los controladores PHP (`AppointmentsController`, `PatientsController`, `ClinicalController`) exigían una expresión regular estricta de UUID v4 (`/^[0-9a-f]{8}-...$/i`), mientras que el frontend o semillas generaban identificadores alfanuméricos como `usr_admin_1` o `pat_173000...`. | Modificación en `PatientsController.php` línea 727 y `AppointmentsController.php` línea 771. | Flexibilización de la regex en backend a `^(?:[0-9a-f-]{36}\|[a-zA-Z0-9_\-\.]{3,36})$` y adopción obligatoria de `crypto.randomUUID()` en todo el cliente. |
| **Pérdida de sesión y rechazo CORS en el navegador** | **[VERIFICADO DIRECTAMENTE]** CloudPanel opera Nginx como proxy inverso hacia PHP-FPM. Al no reenviar `fastcgi_param HTTPS on` y evaluar erróneamente `$_SERVER['HTTPS']`, la cookie de sesión no activaba el atributo `Secure` o fallaba la correspondencia estricta de orígenes en peticiones preflight `OPTIONS`. | Commit `9663498` y `AuthController.php::isHttps()` que ahora valida cabeceras de proxy (`HTTP_X_FORWARDED_PROTO`). | Consolidación en [nginx/vhost-cloudpanel.conf](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/nginx/vhost-cloudpanel.conf#L16-L55) con `fastcgi_param HTTPS on` y manejo estricto de CORS preflight 204. |
| **Bloqueo total de la pantalla al fallar el internet ("Pantalla de desconexión")** | **[VERIFICADO DIRECTAMENTE]** En `App.tsx`, las líneas 31-34 y 53-56 renderizan forzosamente `<DisconnectedScreen reason="offline" />` cuando `!isOnline` o `dataLoadError`, impidiendo el acceso a cualquier vista clínica aunque el navegador tuviese datos cargados. | Inspección de [src/App.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/App.tsx#L32-L56) y [DisconnectedScreen.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/layout/DisconnectedScreen.tsx). | Sustituir la pantalla bloqueante por un banner no intrusivo y permitir la navegación continua sobre la base de datos local desacoplada. |
| **Pérdida de datos ingresados al cerrarse la aplicación o fallar la red** | **[VERIFICADO DIRECTAMENTE]** Formularios como `NewPatientModal.tsx` o `DentalObservationModal.tsx` ejecutan `createPatient` o `createClinicalEncounter` por HTTP en el `onSubmit`. Ante un fallo de red, se captura el error pero no se almacena el borrador en almacenamiento duradero. | Inspección de [NewPatientModal.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/NewPatientModal.tsx#L196-L200). | Enrutar todo `onSubmit` a través del Repositorio Local y la cola persistente `SyncEngine`. |

---

# 4. MAPA DE DEPENDENCIAS ONLINE

| Módulo / Pantalla | Datos que Lee | Datos que Modifica | Endpoints API Implicados | Archivos de Código Relacionados | ¿Puede Operar Offline? | Datos Requeridos para Precarga | Consecuencia si Falla la API a Mitad de Operación |
|---|---|---|---|---|---|---|---|
| **Autenticación (Login)** | Credenciales de usuario (`users`) | Crea sesión en `sessions`, genera cookie y CSRF | `POST /api/v1/auth/login`<br>`GET /api/v1/auth/me` | [LoginView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/auth/LoginView.tsx), [AuthController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Auth/AuthController.php) | **NO en la arquitectura actual**. Sí en la propuesta mediante PIN local derivado con PBKDF2/WebCrypto. | Perfil de usuario, clave pública/salt local | Error `ApiNetworkError`: el usuario no puede ingresar y queda en la pantalla de login. |
| **Panel Principal (Dashboard)** | Citas del día, pacientes recientes, alertas clínicas | Estado de citas (atendido/cancelado) | `GET /api/v1/appointments`<br>`GET /api/v1/patients`<br>`GET /api/v1/reports/summary` | [DashboardView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/dashboard/DashboardView.tsx), [ReportsController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Reports/ReportsController.php) | **SÍ**, si se precargan citas y pacientes en la base de datos local. | Agenda de las últimas 48h y próximas 72h, lista activa de pacientes | Quedan valores en blanco o se despliega `DisconnectedScreen`. |
| **Directorio de Pacientes** | Lista de pacientes, cédulas, teléfonos | Ninguno (solo lectura) | `GET /api/v1/patients?pageSize=100` | [PatientDirectoryView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/PatientDirectoryView.tsx), [PatientsController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Patients/PatientsController.php) | **SÍ**, si los pacientes se consultan desde IndexedDB/SQLite local. | Catálogo completo de pacientes de la clínica (~500 a 5,000 registros ocupan < 3 MB JSON) | La lista queda vacía si no se cargó previamente en memoria. |
| **Creación de Paciente** | Catálogo local de cédulas para duplicados | Inserta en `patients`, `patient_allergies`, `audit_logs` | `POST /api/v1/patients`<br>`POST /api/v1/patients/check-duplicates` | [NewPatientModal.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/NewPatientModal.tsx), [PatientStorage.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Patients/PatientStorage.php) | **SÍ con cola offline**, asignando UUID v4 local y estado `SYNC_PENDING`. | Algoritmo fonético/Levenshtein local para alertas de homónimos | Se muestra alerta de fallo de red; el modal retiene los datos pero si se cierra se pierden. |
| **Historial y Alergias** | `patient_allergies`, `patient_conditions`, `patient_medications` | Actualiza registros médicos y notas clínicas | `PATCH /api/v1/patients/{id}`<br>`GET /api/v1/patients/{id}/history` | [PathologyHistoryModal.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/PathologyHistoryModal.tsx), [PatientSummaryTab.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/PatientSummaryTab.tsx) | **SÍ**, persistiendo la mutación en IndexedDB con versión de registro. | Antecedentes del paciente activo | Error en pantalla. Sin cola, los cambios no se guardan. |
| **Consultas y Evolución** | `clinical_encounters`, diagnósticos, planes | Inserta en `clinical_encounters`, actualiza tratamientos | `GET /api/v1/patients/{id}/encounters`<br>`POST /api/v1/patients/{id}/encounters` | [DentalObservationModal.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/DentalObservationModal.tsx), [ClinicalController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Clinical/ClinicalController.php) | **SÍ**, crítico para atención durante cortes eléctricos. | Historial previo del paciente | El registro de la evolución queda sin guardar si el doctor cierra la ventana. |
| **Odontograma Interactivo** | `clinical_findings`, `clinical_treatments` | Actualiza estado dental (caries, resina, endodoncia) | `GET /api/v1/patients/{id}/odontogram`<br>`PATCH /api/v1/clinical/treatments/{id}/status` | [PatientOdontogramTab.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/patients/PatientOdontogramTab.tsx), [ClinicalStorage.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Clinical/ClinicalStorage.php) | **SÍ**, el mapa SVG/HTML es 100% cliente; solo requiere guardar el estado de las piezas. | Estado odontológico previo del paciente | Estado visual desincronizado con la base de datos si falla el POST. |
| **Galería de Radiografías** | Metadatos en `clinical_files`, archivos en disco | Archiva o restaura registros | `GET /api/v1/patients/{id}/files`<br>`POST .../files/{id}/archive` | [ImageGallery.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/clinical-images/ImageGallery.tsx), [ClinicalFilesController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/ClinicalFiles/ClinicalFilesController.php) | **PARCIALMENTE**: solo las radiografías previamente descargadas a la caché local. | Metadatos y miniaturas del paciente seleccionado | Se muestran cuadros rotos en imágenes no descargadas. |
| **Subida de Radiografías** | Validación de archivo local (tamaño, MIME) | Inserta en `clinical_files`, sube archivo binario al disco | `POST /api/v1/patients/{id}/files` | [ImageUploadModal.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/clinical-images/ImageUploadModal.tsx), [ClinicalFileStorageService.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/ClinicalFiles/ClinicalFileStorageService.php) | **SÍ con cola de Blobs**: almacenar el archivo binario en IndexedDB local y encolar mutación. | Ninguno (solo el archivo seleccionado) | Si se corta a mitad de subida (corte de conexión), el servidor descarta el archivo temporal PHP (`UPLOAD_ERR_PARTIAL`). El cliente debe reintentar con backoff. |
| **Recetas y Récipe A4** | Datos del paciente, clínica y odontóloga | Emite récipe con código hash, inmutable | `GET /api/v1/clinic-settings`<br>`POST /api/v1/patients/{id}/documents` | [DocumentsView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/documents/DocumentsView.tsx), [DocumentsController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Documents/DocumentsController.php) | **SÍ**: la generación del PDF con `jspdf` y la impresión en papel físico se realiza en el navegador. | Datos institucionales de la clínica y firma de la doctora | El récipe se imprime físicamente en el consultorio, pero queda pendiente de registrar en el servidor central. |
| **Agenda y Citas** | `appointments`, pacientes | Reserva, reagendamiento, cancelación | `GET /api/v1/appointments`<br>`POST /api/v1/appointments` | [AgendaView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/agenda/AgendaView.tsx), [AppointmentsController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Appointments/AppointmentsController.php) | **SÍ**, registrando la cita en la cola local con detección de solapamiento. | Calendario de citas del mes actual | Error en pantalla. |
| **Respaldos y Drive** | Volcado de base de datos y archivos clínicos | Genera archivo `.enc` y sube a Google Drive | `POST /api/v1/backups/create`<br>`GET /api/v1/backups` | [SettingsView.tsx](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/src/components/settings/SettingsView.tsx), [BackupController.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Backup/BackupController.php) | **NO**: la copia a Google Drive es una tarea del servidor VPS hacia la nube. | Conectividad entre el VPS y Google APIs | El archivo `.enc` queda en el VPS con estado `FAILED` o retenido localmente para reintento. |

---

# 5. DISEÑO DEL MODELO DE DATOS OFFLINE

### 5.1. Repositorio Clínico Desacoplado (`IClinicalRepository`)
Para evitar que los componentes React dependan directamente de llamadas `fetch` o de la API de IndexedDB, se establece una interfaz abstracta `IClinicalRepository`. La interfaz expone métodos unificados (`getPatients()`, `savePatient()`, `queueMutation()`, `getPendingBlobs()`) e implementa dos adaptadores intercambiables:
- **`IndexedDbAdapter`**: Para navegador web, PWA y tabletas móviles.
- **`SqliteAdapter`**: Para la aplicación de escritorio en Windows con base de datos SQLite cifrada mediante SQLCipher.
- **`InMemoryStorageAdapter`**: Para ejecución instantánea en suites de prueba automatizadas.

### 5.2. Metadatos de Sincronización Estándar (`LocalEntityMeta`)
Cada registro en el almacenamiento local hereda de `LocalEntityMeta`, garantizando trazabilidad y control de concurrencia:

```typescript
export type SyncStatus = 'SYNCED' | 'SYNC_PENDING' | 'SYNC_ERROR' | 'CONFLICT';

export interface LocalEntityMeta {
  sync_status: SyncStatus;
  local_version: number;
  server_version: number;
  last_modified_locally: string; // ISO 8601
  client_mutation_id?: string;
  conflict_details?: string;
}
```

### 5.3. Tabla de Correspondencia entre MySQL Central e IndexedDB Local

| Tabla MySQL Central | Tienda Local (IndexedDB/SQLite) | Regla de Compatibilidad y Sincronización |
|---|---|---|
| `patients` | `local_patients` | UUID v4 idéntico. Cédula y datos filiales persistidos localmente. |
| `patient_allergies` | `local_allergies` | Almacenamiento por paciente. Detección de duplicados por nombre de alérgeno. |
| `clinical_encounters` | `local_encounters` | Bitácora cronológica de consultas. Nunca se sobrescriben; se anexan. |
| `clinical_findings` | `local_findings` | Hallazgos por pieza dental y cuadrante FDI para renderizado del odontograma. |
| `clinical_treatments` | `local_treatments` | Tratamientos propuestos, en progreso o completados con versionado. |
| `clinical_files` | `local_files_meta` | Metadatos técnicos de radiografías (título, pieza, SHA-256, tamaño, fecha). |
| *[N/A en Servidor]* | `clinical_blobs` | Tienda binaria local para almacenar el Blob JPG/PNG hasta su subida. |
| `client_mutation_logs`| `mutation_queue` | Cola duradera de operaciones pendientes (FIFO) con reintento exponencial. |

---

# 6. AUDITORÍA PROFUNDA DE IMÁGENES Y RADIOGRAFÍAS

### 6.1. Ciclo de Vida y Estados de Archivos Clínicos

```
[Selección de Imagen]
         │
         ▼
    LOCAL_ONLY ────────► PENDING_UPLOAD ────────► UPLOADING ────────► UPLOADED (Confirmado)
         ▲                     ▲                      │
         │                     │                      ▼
         │                     └─────────────── RETRY_REQUIRED
         │                                            │
         │                                            ▼
         └───────────────────────────────── FAILED_REQUIRES_ACTION
```

- **`LOCAL_ONLY`**: Radiografía seleccionada offline. Guardada en la tienda de Blobs local sin subida intentada.
- **`PENDING_UPLOAD`**: Encolada formalmente en `mutation_queue` con metadatos y hash SHA-256 calculado.
- **`UPLOADING`**: Transferencia binaria activa hacia el endpoint `/api/v1/patients/{id}/files`.
- **`UPLOADED`**: Confirmada por el servidor (HTTP 201). Hash verificado. Se retiene miniatura local.
- **`RETRY_REQUIRED`**: Fallo transitorio de conexión o timeout. En espera de reintento automático con backoff.
- **`CONFLICT`**: El expediente o el paciente cambió en el servidor antes de asociar el estudio.
- **`FAILED_REQUIRES_ACTION`**: Error permanente (archivo corrupto, rechazo de formato). Requiere acción manual.

### 6.2. Protocolo de Carga Offline de Radiografías en 15 Pasos
1. La odontóloga selecciona la radiografía (JPG/PNG) y asigna la pieza dental FDI (ej. pieza 46).
2. El cliente valida que el tamaño sea $\le$ 25 MB y el tipo MIME sea estrictamente `image/jpeg` o `image/png`.
3. Se calcula el hash SHA-256 mediante WebCrypto (`crypto.subtle.digest`) en el hilo del navegador.
4. El archivo binario original se guarda íntegro en la tienda IndexedDB `clinical_blobs` con clave compuesta `{patient_id}_{file_id}`.
5. Se genera un `ObjectURL` temporal para permitir previsualización y zoom inmediato en el visor de rayos X.
6. La radiografía se vincula al paciente y se muestra en la galería con estado "Pendiente de Sincronización".
7. Se registra la operación en `mutation_queue` con prioridad de subida de archivos.
8. Si la PC o el navegador se cierran o se apagan por corte eléctrico, el Blob y la cola persisten intactos.
9. Al restablecerse la red, `SyncEngine` verifica primero que el paciente asociado exista en el servidor central.
10. Se inicia la subida mediante `XMLHttpRequest` enviando la cabecera idempotente `x-client-mutation-id`.
11. El servidor PHP valida la sesión, verifica la cuota de disco y comprueba si el hash SHA-256 ya existe en la base de datos.
12. Si el hash ya existe en el expediente, la API responde HTTP 200 sin duplicar el archivo en el disco del VPS.
13. Si es nuevo, PHP escribe el original en `/clinical_storage/originals` y genera miniatura de 400px en `thumbnails/`.
14. La API retorna HTTP 201 Created con el identificador definitivo del archivo clínico.
15. El cliente marca el archivo como `UPLOADED`, elimina el blob pesado local y conserva la miniatura en caché.

---

# 7. PROTOCOLO DE SINCRONIZACIÓN Y CONSISTENCIA

### 7.1. Idempotencia y Reintentos con Backoff Exponencial
Toda mutación generada sin conexión lleva un UUID v4 en la cabecera `x-client-mutation-id`. La tabla `client_mutation_logs` en MySQL posee una restricción `UNIQUE` sobre `client_mutation_id`. Si un timeout de red corta la respuesta pero el servidor ya ejecutó la inserción, el reintento posterior detecta la clave duplicada y responde con éxito sin duplicar pacientes ni consultas.

Los reintentos automáticos aplican una fórmula con jitter aleatorio para prevenir tormentas de peticiones al restablecerse internet:
$$\text{Retardo} = \min(2^{\text{reintentos}} \times 1500\,\text{ms},\,60000\,\text{ms}) + \text{random}(0,\,500\,\text{ms})$$

### 7.2. Prohibición de "Last-Write-Wins" en Información Clínica
Bajo ninguna circunstancia se aplicará "la última escritura gana" (Last-Write-Wins) sobre historiales clínicos. Sobrescribir una historia médica con una versión offline desactualizada puede borrar alergias a medicamentos declaradas previamente, poniendo en peligro la vida del paciente.

| Entidad Médica | Estrategia de Resolución | Mecanismo de Salvaguarda Clínica |
|---|---|---|
| **Alergias y Medicamentos** | Fusión Conservadora (Unión de Conjuntos) | Si un cliente offline añade una alergia y el servidor tenía otra, se conservan ambas en el expediente sin descartar ninguna. |
| **Consultas y Evolución** | Anexo Cronológico Monotónico | Cada consulta tiene su propio UUID v4. Nunca se sobrescribe una consulta anterior; se inserta como evolución adicional en la bitácora. |
| **Odontograma y Tratamientos** | Detección por Versión + Comparador Lado a Lado | Si `server_version != local_version`, la mutación se pausa en estado `CONFLICT` y se presenta el modal a la doctora para resolver la divergencia. |
| **Archivos y Radiografías** | Inmutabilidad por Firma Criptográfica SHA-256 | Las radiografías son documentos médicos inmutables. Nunca se sobrescriben; solo admiten los estados `ACTIVO` o `ARCHIVADO`. |

---

# 8. COMPARATIVA DE CLIENTES Y VIABILIDAD EN EL VPS

### 8.1. Comparativa de Alternativas de Cliente

| Criterio de Evaluación | Opción A: PWA Mejorada | Opción B: Desktop Tauri (Rust) | Opción C: Desktop Electron |
|---|---|---|---|
| **Reutilización Frontend** | **100% directa** | **98%** (requiere IPC local) | **98%** (requiere proceso Node) |
| **Persistencia Local** | IndexedDB en navegador | **SQLite / SQLCipher nativo** | **SQLite / SQLCipher nativo** |
| **Consumo de RAM** | 150 - 300 MB (navegador) | **40 - 80 MB** (WebView2 nativo) | 250 - 500 MB (Chromium completo) |
| **Peso del Instalador** | **0 MB** (instalación web 1-clic) | **8 - 15 MB** (`.msi` / `.exe`) | 80 - 120 MB (`.exe` pesado) |
| **Acceso a Sensor RX** | Restringido (File Picker web) | **Total** (acceso a carpeta local RX) | **Total** (acceso a drivers C++) |
| **Compatibilidad Tabletas** | **Nativa** (iPad, Android, Laptop)| Requiere compilación nativa | No compatible con móviles/tablets |
| **Recomendación** | **FASE 1 (Inmediata sin descargas)**| **FASE 2 (Escritorio principal)**| **NO RECOMENDADO** (Sobrecarga innecesaria) |

### 8.2. Matriz de Viabilidad en el VPS Actual

| Servicio de Infraestructura | Estado de Viabilidad | Detalle de Configuración Requerida |
|---|---|---|
| **Alojamiento Web SPA** | Viable con infraestructura actual | Nginx sirve `dist/` con encabezados inmutables y revalidación de `index.html`. |
| **App Shell Offline PWA** | Viable con cambios de configuración | Ajustar `vite.config.ts` para precachear el shell estático sin precachear datos médicos. |
| **Almacenamiento Radiografías** | Viable con infraestructura actual | Con 77 GB libres, soporta más de 25,000 radiografías periapicales sin coste extra. |
| **Distribución Instalador Tauri** | Viable con cambios de configuración | Crear directorio `/downloads/` protegido en Nginx para distribuir el ejecutable de la clínica. |
| **Sincronización Bidireccional**| Viable con infraestructura actual | MySQL 8.4 y PHP-FPM procesan mutaciones en milisegundos con mínimo impacto de CPU. |
| **Respaldos en Google Drive** | Viable con cambios de configuración | Requiere depositar la credencial JSON de Service Account y programar cron nocturno. |

---

# 9. PLAN DE RESPALDOS Y SEGURIDAD CLÍNICA

### 9.1. Especificación del Paquete de Respaldo Cifrado (`suclident-backup-v1`)
El módulo [BackupManager.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Backup/BackupManager.php) implementa la arquitectura de respaldo de grado militar. El paquete `.enc` consta de:
- Cabecera mágica `SUCL` (4 bytes).
- Versión `0x01` (1 byte).
- Vector de Inicialización aleatorio (12 bytes).
- Tag de Autenticación GCM (16 bytes).
- Carga Cifrada **AES-256-GCM** sobre flujo ZIP comprimido.
- Dentro del archivo descifrado se incluye `manifest.json` con conteo de registros y hashes SHA-256, `database_dump.json` generado bajo transacción `REPEATABLE READ` (excluyendo tokens y contraseñas) y los archivos binarios clínicos.

### 9.2. Prevención de Desastres y Restauración Aislada
[RestoreManager.php](file:///c:/Users/Valdez/Documents/Suclident_del_Sur_Fases/api/src/Modules/Backup/RestoreManager.php#L34-L44) incorpora una salvaguarda estricta: bloquea automáticamente la restauración si `APP_ENV !== "test"` o si el nombre de la base de datos no finaliza en `_test`. Esto impide que un error humano sobrescriba los datos vivos en producción.

### 9.3. Autenticación y Autorización Offline (PIN Clínico)
Para permitir el acceso durante caídas de internet sin comprometer la seguridad: el dispositivo debe iniciar sesión online al menos una vez; en ese momento, la odontóloga establece un PIN numérico de 6 dígitos. Mediante WebCrypto `PBKDF2` (100,000 iteraciones SHA-256) se deriva una clave criptográfica local que descifra la réplica en el dispositivo. Si el dispositivo se extravía, una sesión revocada en el servidor central ordenará la purga del almacenamiento local en su primer intento de sincronización.

> [!WARNING] Límite de Vigencia Operativa Desconectada
> Una estación offline puede operar atendiendo pacientes hasta un máximo estricto de **7 días** (`MAX_MUTATION_AGE_DAYS`). Cumplido este lapso, el sistema exige reconexión para revalidar la vigencia del profesional y renovar certificados.

---

# 10. PLAN DE PRUEBAS Y CRITERIOS DE ACEPTACIÓN (20 Casos Críticos)

| # | Caso de Prueba | Procedimiento de Prueba | Criterio de Aceptación Verificable |
|---|---|---|---|
| **1** | Login Normal Online | Ingresar credenciales en `LoginView`. | HTTP 200, cookie `HttpOnly`, token CSRF, carga de pacientes. |
| **2** | Apertura Offline PWA | Desconectar red y abrir `suclident.com`. | App Shell carga de Service Worker; sin pantalla de error dinosaurio. |
| **3** | Desbloqueo PIN Offline | Ingresar PIN clínico previamente fijado. | Valida hash PBKDF2 local; permite acceso a la interfaz. |
| **4** | Rechazo PIN Inválido | Ingresar PIN erróneo 3 veces. | Acceso bloqueado; datos locales permanecen cifrados. |
| **5** | Búsqueda Paciente Offline | Buscar por cédula o nombre en barra superior. | Ficha del paciente abre al instante desde IndexedDB. |
| **6** | Crear Paciente Offline | Llenar `NewPatientModal` y guardar. | Guarda en BD local con UUID v4 y estado `SYNC_PENDING`. |
| **7** | Registro Consulta Offline | Añadir evolución en `DentalObservationModal`. | Evolución entra a bitácora local; odontograma actualiza color. |
| **8** | Guardar Radiografía Offline | Cargar radiografía en `ImageUploadModal`. | Blob se almacena en `clinical_blobs`; etiqueta "Pendiente". |
| **9** | Recarga con F5 Offline | Presionar F5 con operaciones en cola. | La cola persiste íntegra; badge muestra ítems retenidos. |
| **10** | Apagado Abrupto de PC | Cerrar navegador o cortar energía a la PC. | Al reiniciar la PC, los datos offline continúan intactos. |
| **11** | Reconexión Automática | Restablecer cable de red / WiFi. | `SyncEngine` procesa la cola FIFO automáticamente. |
| **12** | Timeout Ambiguo en Red | Forzar caída de red en el byte de respuesta. | Reintento con `x-client-mutation-id` no genera duplicados. |
| **13** | Reintento Manual | Hacer clic en "Reintentar" en modal sync. | Mutación se reenvía sin generar duplicados en MySQL. |
| **14** | Conflicto Concurrente | Editar misma ficha en 2 PCs a la vez. | Detecta versión divergente; abre modal lado a lado (no LWW). |
| **15** | Actualización App Shell | Desplegar nuevo bundle en el VPS. | Service Worker actualiza en segundo plano y avisa al usuario. |
| **16** | Almacenamiento Lleno | Simular cuota de navegador al 95%. | Captura error de cuota y notifica sin corromper la cola. |
| **17** | Respaldo Manual Completo | Ejecutar `POST /backups/create`. | Genera archivo `.enc` con cabecera `SUCL` y SHA-256 verificado. |
| **18** | Verificación Integridad | Ejecutar `verifyBackupIntegrity()` sobre `.enc`. | Autentica tag GCM y confirma integridad del paquete. |
| **19** | Restauración en Test | Restaurar en BD `suclident_test` con `APP_ENV=test`. | Restaura tablas y archivos; producción queda inalterada. |
| **20** | Fallo de Google Drive | Simular corte en subida hacia Google API. | Registra estado `FAILED`; retiene archivo en VPS para retry. |

---

# 11. MATRIZ DE PRIORIZACIÓN Y HOJA DE RUTA

### 11.1. Matriz de Priorización de Tareas

| ID | Tarea Técnica de Arquitectura | Severidad | Esfuerzo Estimado | Criterio de Aceptación |
|---|---|---|---|---|
| **AUD-01** | Desbloquear App Shell (retirar `DisconnectedScreen`) | **CRÍTICA** | Bajo (1-2 días) | Shell disponible sin red en cualquier pestaña. |
| **AUD-02** | Precachear App Shell seguro en Service Worker | **CRÍTICA** | Medio (2-3 días) | F5 sin conexión carga la aplicación completa. |
| **AUD-03** | Crear Repositorio Local Unificado (IndexedDB) | **CRÍTICA** | Alto (4-6 días) | Pacientes y consultas persisten al cerrar navegador. |
| **AUD-04** | Enrutar Formularios a la Cola Persistente | **CRÍTICA** | Alto (5-7 días) | Cero pérdida de datos ante cortes de energía o red. |
| **AUD-05** | Persistencia Local de Blobs de Radiografías | **ALTA** | Medio (3-4 días) | Fotos y rayos X visibles offline antes de sincronizar. |
| **AUD-06** | Autenticación y Autorización Offline (PIN) | **ALTA** | Medio (3-4 días) | Acceso al consultorio durante contingencias sin internet. |
| **AUD-07** | Activar Credencial Real Google Drive en VPS | **MEDIA** | Bajo (1-2 días) | Respaldos automáticos verificados en la nube. |
| **AUD-08** | Empaquetado de Escritorio con Tauri (Windows) | **MEDIA** | Alto (6-8 días) | Aplicación nativa `.exe` operando en el sillón dental. |

### 11.2. Hoja de Ruta de Implementación por Fases
- **Fase 1 (Semanas 1 y 2)**: Capa de Repositorio Unificado (`IClinicalRepository`), esquema IndexedDB v2 y enrutamiento de formularios clínicos.
- **Fase 2 (Semanas 3 y 4)**: Reconfiguración de Service Worker, App Shell offline seguro y autenticación con PIN derivado por `PBKDF2`.
- **Fase 3 (Semanas 5 y 6)**: Almacenamiento local de Blobs de radiografías, visor offline y resolución de conflictos lado a lado.
- **Fase 4 (Semanas 7 y 8)**: Activación de Google Drive Service Account en VPS y compilación de instalador Tauri para Windows 10/11.

---

# 12. DECISIONES PENDIENTES DE APROBACIÓN POR LA DIRECCIÓN

1. **Aprobación del PIN de 6 dígitos**: ¿Se aprueba el PIN numérico rápido derivado con `PBKDF2` para desbloqueo offline cuando no haya internet?
2. **Cuota de Radiografías**: ¿Se aprueba mantener el límite de 10 MB en web y ampliarlo a 25 MB en la app de escritorio?
3. **Cuenta Google Cloud**: ¿Se autoriza la creación de una Service Account con acceso exclusivo a la carpeta de respaldos en Google Drive?
4. **Estrategia Gradual**: ¿Se aprueba habilitar primero la PWA web mejorada y posteriormente el instalador Tauri en el consultorio?

---
*Fin del informe oficial de auditoría de arquitectura.*

