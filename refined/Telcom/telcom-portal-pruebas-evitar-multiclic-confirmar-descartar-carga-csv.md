# [HU] Evitar bloqueo por multi‑clic en Confirmar/Descartar durante la carga de CSV (B2B/B2C)

**Como** Analista/Usuario interno que realiza la carga de archivos CSV en el portal,  
**Quiero** que los botones **“Confirmar”** y **“Descartar”** del flujo de previsualización sean idempotentes y se deshabiliten al primer clic (o ignoren clics repetidos),  
**Para** evitar bloqueos de interfaz, confirmaciones duplicadas y estados inconsistentes al cargar reportes para **B2B** y **B2C**.

---

## Contexto de Negocio
Durante la ejecución de pruebas del delivery de **Cargue/Descargue de reportes** en el módulo/aplicación **Portal Transporte**, se evidenció que, al hacer clic rápidamente dos o más veces sobre **Confirmar** o **Descartar** en la pantalla/modal de previsualización de la carga de CSV (B2B/B2C), la aplicación puede quedar “bloqueada” (la previsualización permanece visible, no permite salir ni descartar/confirmar) aunque la base de datos sí recibe la confirmación del primer clic.

Esto afecta:
- La continuidad operativa del usuario interno (pérdida de capacidad de respuesta ante clics repetidos).
- La confiabilidad del proceso de cargue (riesgo de duplicidad o inconsistencias).
- La experiencia de usuario y la percepción de calidad del portal.

---

## Alcance / Fuera de Alcance

### Alcance
- Aplicar control de **multi‑clic** (debounce/lock) en los botones **Confirmar** y **Descartar** en el flujo de **previsualización** de carga de CSV.
- Comportamiento aplicable a **B2B y B2C**.
- Asegurar que el sistema:
  - Procese **solo una vez** la acción de Confirmar o Descartar por intento de carga.
  - Mantenga la UI consistente (sin quedar bloqueada) ante clics repetidos.
- Ajuste de texto del mensaje mostrado al intentar cargar un archivo vacío (hallazgo ortográfico: “archivox” → “archivo/archivos”), si corresponde al mismo flujo de carga.

### Fuera de Alcance
- Cambios al formato/estructura del CSV o validaciones de negocio adicionales del contenido.
- Rediseño completo del flujo de carga o de la pantalla de previsualización.
- Cambios a infraestructura, autenticación, o roles/perfiles (salvo que sea requisito para esta corrección).

---

## Reglas de Negocio
1. **Idempotencia UI**: Una vez el usuario ejecute **Confirmar** o **Descartar**, el sistema debe considerar la acción como “en curso” y no permitir re-ejecutarla hasta completar.
2. **Acción única**: Si el usuario realiza múltiples clics sobre el mismo botón, el sistema debe:
   - Ignorar clics subsiguientes, o
   - Deshabilitar el botón de manera inmediata tras el primer clic.
3. **Consistencia de estado**:
   - Si la acción es **Confirmar** y el backend responde éxito, se debe informar el éxito del cargue y cerrar/avanzar el flujo según corresponda.
   - Si la acción es **Descartar**, se debe cerrar el modal/previsualización y liberar la interfaz.
4. **Mensajería correcta**: En caso de archivo vacío, el mensaje debe ser ortográficamente correcto (no debe aparecer “archivox”).

---

## Criterios de Aceptación Funcionales

**Escenario: Confirmación exitosa con un solo clic**
- **Dado que** el usuario cargó un archivo CSV válido (B2B o B2C) y se encuentra en la previsualización,
- **Cuando** hace clic una vez en **“Confirmar”**,
- **Entonces** el sistema debe procesar la confirmación una única vez y mostrar un mensaje de éxito (o feedback equivalente del flujo).

**Escenario: Multi‑clic en Confirmar no bloquea la interfaz**
- **Dado que** el usuario está en la previsualización de una carga de CSV (B2B o B2C),
- **Cuando** hace dos o más clics rápidos sobre **“Confirmar”**,
- **Entonces** el sistema debe ignorar los clics subsiguientes o deshabilitar el botón inmediatamente,
- **Y** la interfaz no debe quedar bloqueada,
- **Y** no debe ser posible generar una segunda ejecución de confirmación desde la UI para el mismo intento.

**Escenario: Descartar cierra el modal al primer clic**
- **Dado que** el usuario está en la previsualización de una carga de CSV (B2B o B2C),
- **Cuando** hace clic una vez en **“Descartar”**,
- **Entonces** el sistema debe cerrar la previsualización/modal y liberar la interfaz para continuar navegando o realizar una nueva carga.

**Escenario: Multi‑clic en Descartar no bloquea la interfaz**
- **Dado que** el usuario está en la previsualización de una carga de CSV (B2B o B2C),
- **Cuando** hace dos o más clics rápidos sobre **“Descartar”**,
- **Entonces** el sistema debe ignorar los clics subsiguientes o deshabilitar el botón inmediatamente,
- **Y** la previsualización/modal debe cerrarse correctamente (o mantenerse en un estado consistente si ya se está cerrando),
- **Y** la interfaz no debe quedar bloqueada.

**Escenario: Mensaje correcto al cargar archivo vacío**
- **Dado que** el usuario selecciona un archivo vacío para B2B o B2C,
- **Cuando** hace clic en **“Guardar”**,
- **Entonces** el sistema debe mostrar un mensaje de validación con ortografía correcta,
- **Y** el mensaje no debe contener la palabra “archivox”.

---

## Criterios de Aceptación Técnicos
- La acción de **Confirmar** y **Descartar** debe implementarse con un mecanismo de control de concurrencia a nivel UI (por ejemplo, “isSubmitting/isProcessing”):
  - Setear el estado en `true` al primer clic.
  - Deshabilitar el botón mientras `isProcessing = true`.
  - Rehabilitar al finalizar el proceso (success/error) o al cerrar el modal.
- Debe existir protección para evitar:
  - Doble disparo de eventos.
  - Doble invocación de la misma operación por un mismo intento (al menos desde la UI).
- La UI debe manejar correctamente estados de error (si la confirmación falla):
  - Mostrar feedback de error.
  - Rehabilitar la acción de manera controlada (sin quedar bloqueada).

---

## NFR / Requisitos No Funcionales
- **Usabilidad:** respuesta inmediata visual al primer clic (deshabilitado/spinner/estado “procesando”).
- **Confiabilidad:** no debe presentarse bloqueo de interfaz ante multi‑clic.
- **Testabilidad:** el comportamiento debe ser verificable mediante pruebas funcionales y/o automatizadas.
- **Compatibilidad:** comportamiento consistente para B2B y B2C.

---

## Notas UX/UI
- Al primer clic en **Confirmar** o **Descartar**, se recomienda:
  - Deshabilitar el botón presionado y, si aplica, ambos botones mientras se procesa.
  - Mostrar indicador de carga (spinner) o estado “Procesando...”.
- Evitar que el usuario quede “atrapado” en la previsualización; siempre debe existir un estado consistente al finalizar la acción.

---

## Riesgos
- Implementar solo bloqueo en UI sin contemplar reintentos controlados ante error podría impedir al usuario completar el flujo si ocurre un fallo transitorio.
- Si existen listeners duplicados o re-renderizaciones, podría persistir el doble disparo si no se controla correctamente el estado.
- Si el backend no es idempotente, el riesgo de duplicidad puede permanecer si existen reintentos no controlados; se debe validar el comportamiento end-to-end.

---

## QA / UAT
- Ejecutar pruebas para:
  - B2B: carga CSV → previsualización → multi‑clic en Confirmar → validar que no se bloquee y que se procese una sola vez.
  - B2C: carga CSV → previsualización → multi‑clic en Confirmar → validar que no se bloquee y que se procese una sola vez.
  - Multi‑clic en Descartar (B2B/B2C) → validar cierre del modal y liberación de interfaz.
  - Validación de archivo vacío (B2B/B2C) → verificar mensaje sin “archivox”.
- Evidenciar que después del flujo el usuario puede continuar operando (nueva carga, navegación, etc.).

---

## Definición de Hecho (DoD)
- La HU cumple INVEST y cuenta con criterios de aceptación verificables.
- Se implementa el control de multi‑clic (deshabilitado/ignorar) en Confirmar/Descartar.
- No se reproduce el bloqueo reportado en pruebas (B2B/B2C).
- Se corrige el mensaje ortográfico asociado al archivo vacío (si aplica al alcance acordado).
- Pruebas funcionales ejecutadas y evidencias adjuntas.
- Sin regresiones en el flujo de carga/descarga de reportes.
