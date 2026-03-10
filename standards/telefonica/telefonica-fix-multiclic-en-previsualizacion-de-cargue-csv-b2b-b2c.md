# Fix multiclic y bloqueo de estado en previsualización de cargue CSV (B2B/B2C) – Portal BI

## Definición (Como / Quiero / Para)
**Como** usuario interno del módulo *Portal Transporte* (Delivery Cargue/Descargue de reportes)  
**Quiero** que al confirmar o descartar la previsualización del cargue CSV (B2B/B2C) el sistema controle los clics repetidos y gestione el estado de la UI de forma idempotente  
**Para** evitar bloqueos de la interfaz, ejecuciones duplicadas y estados inconsistentes durante el flujo de cargue.

## Contexto de Negocio
En ambiente de pruebas (desarrollo), durante la ejecución de los casos:
- **CP_001**: HU-Cliente-interno_Delivery-Carga-CSV_B2B
- **CP_004**: HU-Cliente-interno_Delivery-Carga-CSV_B2C

se identificó que al hacer **doble clic o múltiples clics rápidos** sobre los botones **“Confirmar”** o **“Descartar”** en la pantalla/modal de **previsualización**, la aplicación:
- queda en un **estado de bloqueo** (la previsualización permanece visible),
- **no permite salir** (descartar deja de cerrar),
- “Confirmar” puede quedar **aparentemente activo**,
- pero **la confirmación sí llega a base de datos** con el primer clic (riesgo de duplicidad o de estado desincronizado UI vs backend).

Adicionalmente se encontró un hallazgo de **copy/ortografía** en el mensaje mostrado al intentar cargar un archivo vacío: aparece “**archivox**” en lugar de “archivo/archivos”.

## Alcance / Fuera de Alcance

### Alcance
1. Control de multiclic para acciones **Confirmar** y **Descartar** en la previsualización del cargue CSV para **B2B y B2C**.
2. Gestión correcta de estados de UI durante la operación:
   - loading/procesando,
   - éxito,
   - error,
   - cierre/escape del modal.
3. Asegurar **idempotencia a nivel UI** (y/o soporte backend si aplica) para que solicitudes duplicadas no generen estados inconsistentes.
4. Corrección del texto “archivox” en la modal de validación de archivo vacío.

### Fuera de alcance
- Cambios en la estructura del archivo CSV, reglas de validación de negocio del contenido o mapeos de campos.
- Rediseño completo del flujo de cargue; solo se corrige control de interacción/estado y el copy identificado.
- Optimización de performance del backend no relacionada a la operación de confirmar/descartar.

## Reglas de Negocio
1. **Primer clic efectivo**: el primer clic en **Confirmar** o **Descartar** es el único que dispara la acción.
2. **Bloqueo inmediato**: al primer clic, el botón accionado (y preferiblemente ambos botones) debe:
   - deshabilitarse inmediatamente, y/o
   - ignorar eventos subsecuentes hasta finalizar la operación.
3. **Idempotencia**:
   - Si el usuario dispara la confirmación y por cualquier razón se reintenta (doble clic, latencia, re-render), el sistema no debe generar duplicidad ni inconsistencias.
   - Si existe un identificador de operación (ej. `uploadId`, `processId`, hash del archivo + usuario + timestamp controlado), debe reutilizarse para evitar dobles procesamientos.
4. **Consistencia UI**:
   - Tras **Confirmar** exitoso, la UI debe informar éxito y salir del estado de previsualización según el flujo definido (cerrar modal o navegar a la siguiente vista).
   - Tras **Descartar**, la UI debe cerrar la previsualización sin persistir cambios y restablecer el estado para permitir un nuevo cargue.
5. **Manejo de error**:
   - Si falla Confirmar/Descartar (timeout, error 4xx/5xx), la UI debe mostrar un mensaje claro y permitir reintento seguro (rehabilitar botones o presentar acción “Reintentar”).
6. **Copy de validación**: cuando el archivo esté vacío, el mensaje no debe contener “archivox”; debe usar el término correcto definido por UX/Negocio (ej. “archivo” o “archivos”).

## Criterios de Aceptación

### Funcionales (Formato Given/When/Then)
**CA-F01 – Deshabilitar/ignorar multiclic en Confirmar**  
**Dado** que el usuario ha cargado un CSV (B2B o B2C) y está en la previsualización  
**Cuando** hace clic en “Confirmar” y realiza clics adicionales rápidamente  
**Entonces** solo el primer clic ejecuta la acción de confirmación  
**Y** los clics subsecuentes se ignoran o el botón queda deshabilitado inmediatamente  
**Y** la UI no queda bloqueada en la previsualización.

**CA-F02 – Deshabilitar/ignorar multiclic en Descartar**  
**Dado** que el usuario está en la previsualización del cargue  
**Cuando** hace clic en “Descartar” y realiza clics adicionales rápidamente  
**Entonces** el modal/pantalla de previsualización se cierra con el primer clic  
**Y** la interfaz queda disponible para continuar operando (sin bloqueo).

**CA-F03 – Estado de carga visible y consistente**  
**Dado** que el usuario presiona “Confirmar” o “Descartar”  
**Cuando** la operación está en curso  
**Entonces** se muestra un estado “procesando” (spinner/disabled/overlay según UX)  
**Y** el usuario no puede disparar la misma acción múltiples veces.

**CA-F04 – Recuperación ante error**  
**Dado** que ocurre un error al confirmar o descartar (ej. respuesta 500 o timeout)  
**Cuando** la UI recibe el error  
**Entonces** se informa el error con un mensaje entendible  
**Y** se permite reintentar sin duplicar operaciones ni dejar el flujo en estado bloqueado.

**CA-F05 – Corrección de copy en archivo vacío**  
**Dado** que el usuario intenta guardar/cargar un archivo vacío (B2B/B2C)  
**Cuando** el sistema valida el archivo  
**Entonces** se muestra un mensaje sin errores ortográficos  
**Y** no se presenta la cadena “archivox”.

### Técnicos
**CA-T01 – Prevención de doble submit**  
- Los handlers de “Confirmar/Descartar” deben tener control de *single-flight* (bandera `isSubmitting/isProcessing` o mecanismo equivalente).
- Se debe asegurar que el estado no se resetee por re-render hasta que finalice la promesa/llamada.

**CA-T02 – Idempotencia (si aplica backend)**  
- Si el endpoint de confirmación procesa una operación, debe manejar reintentos con el mismo `operationId` sin duplicar registros.
- Registro de auditoría/log debe permitir trazar `operationId`, usuario, tipo (B2B/B2C) y resultado.

**CA-T03 – Manejo de concurrencia y navegación**  
- Si el usuario intenta cerrar/navegar durante “procesando”, el sistema debe prevenirlo o confirmar la acción, evitando estados intermedios.

## NFR / Requisitos No Funcionales
- **Usabilidad:** feedback inmediato al clic; no debe percibirse “congelamiento”.
- **Confiabilidad:** el flujo no debe quedar en estado no recuperable; siempre debe existir salida (éxito, error con reintento, o cancelación controlada).
- **Observabilidad:** logs/trazas para detectar doble submit, reintentos y errores (front y/o back).
- **Seguridad:** no exponer información sensible en mensajes de error; mantener controles de autorización existentes para el módulo.

## Notas UX/UI
- Recomendado: al primer clic, deshabilitar ambos botones y mostrar “Procesando…” en el botón accionado.
- Evitar que la previsualización permanezca “atrapada” en pantalla: garantizar cierre/navegación post-acción.
- Texto validación archivo vacío: definir string final con UX/Negocio (singular/plural) y homogeneizar para B2B/B2C.

## Riesgos
- Desincronización UI vs backend si el backend confirma y la UI queda bloqueada (riesgo actual).
- Duplicidad de operaciones si el backend no es idempotente y la UI permite reintentos sin control.
- Re-render o refresh parcial puede resetear la bandera `isProcessing` si no se maneja correctamente.

## QA / UAT
### Casos de prueba recomendados
1. **Multiclic Confirmar** (B2B): doble/triple clic rápido → 1 sola confirmación, UI ok.
2. **Multiclic Confirmar** (B2C): idem.
3. **Multiclic Descartar** (B2B/B2C): cierra al primer clic, UI libera.
4. **Latencia simulada** (throttling): confirmar con demora → botones deshabilitados; al finalizar, éxito.
5. **Error 500/timeout**: mostrar error, permitir reintentar sin bloqueo.
6. **Archivo vacío**: modal con texto correcto (sin “archivox”).
7. **Accesibilidad básica**: foco/teclado en modal, estado disabled anunciado (si aplica).

### Evidencia mínima
- Capturas del estado “procesando”.
- Log/traza de una sola llamada por acción ante multiclic.
- Registro de que no se crean duplicados (si aplica backend).

## Definición de Hecho (DoD)
- Criterios de aceptación funcionales y técnicos cumplidos (B2B y B2C).
- Pruebas QA ejecutadas y evidencias adjuntas.
- No existen bloqueos de UI ante multiclic.
- Mensajería/copy corregido y validado.
- Cambio desplegado en ambiente de pruebas y validado por UAT/negocio (cuando aplique).
- Se cuenta con logs/trazas suficientes para diagnosticar reintentos/doble submit.

---

**status:** Awaiting_Approval
