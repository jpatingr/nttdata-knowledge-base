# HU — Prevenir bloqueo por multi-clic en Confirmar/Descartar durante el cargue de reportes CSV (B2B/B2C)

**Como** Analista/Usuario interno que realiza el cargue de reportes (B2B o B2C),  
**Quiero** que el sistema ignore o deshabilite inmediatamente acciones repetidas sobre los botones **“Confirmar”** y **“Descartar”** en la pantalla de previsualización,  
**Para** evitar bloqueos de interfaz, estados inconsistentes y ejecuciones duplicadas de la operación de cargue.

---

## 1. Contexto de Negocio

Durante la ejecución de pruebas del delivery **“Cargue/Descargue de reportes”** se evidenció que, al hacer clic repetido (rápido) sobre **Confirmar** o **Descartar** en la previsualización del cargue (para B2B y B2C), la interfaz queda en un estado de bloqueo: permanece la previsualización visible, el botón **Descartar** deja de cerrar, y **Confirmar** permanece activo.  
Adicionalmente, se observó que **la confirmación sí llega a base de datos con el primer clic**, pero la UI no se recupera, lo que genera reprocesos, confusión del usuario y riesgo de duplicidad.

Este comportamiento afecta la operatividad del proceso de cargue, genera retrabajo y puede producir inconsistencias si existen efectos secundarios asociados a la confirmación/descartar.

---

## 2. Alcance / Fuera de Alcance

### Alcance
- Aplicar control anti-doble-clic / anti-multi-clic para:
  - Botón **Confirmar** en previsualización de cargue.
  - Botón **Descartar** en previsualización de cargue.
- Considerar los dos flujos:
  - Cargue **B2B** (CSV)
  - Cargue **B2C** (CSV)
- Asegurar que, tras el primer clic válido:
  - La acción se ejecute una sola vez.
  - El usuario reciba feedback claro (cargando / éxito / error).
  - La pantalla quede en un estado consistente y recuperable.

### Fuera de Alcance
- Cambios funcionales al proceso de negocio del cargue (validaciones del archivo, reglas del contenido).
- Rediseño completo de la pantalla de previsualización.
- Corrección del hallazgo ortográfico “archivox” (se gestionará en una HU independiente).

---

## 3. Reglas de Negocio

1. **Unicidad de acción por interacción:** Para una misma previsualización, una acción de **Confirmar** o **Descartar** debe procesarse **una sola vez** por intento del usuario.
2. **Prioridad del primer clic:** El primer clic válido dispara la operación; clics posteriores deben ser ignorados mientras la operación esté en curso.
3. **Recuperación de estado:** Al finalizar la operación (éxito o error), la UI debe:
   - Mostrar el resultado, y
   - Habilitar nuevamente acciones permitidas según el estado final.

---

## 4. Criterios de Aceptación Funcionales (BDD/Gherkin)

**Escenario: Confirmación de cargue ignora multi-clic en B2B**  
- **Dado que** el usuario cargó un archivo CSV para **B2B** y está en la pantalla de previsualización  
- **Cuando** hace clic dos o más veces rápidamente en el botón **“Confirmar”**  
- **Entonces** el sistema debe ejecutar **una sola** confirmación del cargue  
- **Y** la interfaz debe mostrar feedback de procesamiento (por ejemplo, estado “Cargando”)  
- **Y** al finalizar debe mostrar un mensaje de resultado (éxito o error) sin quedar bloqueada  

**Escenario: Confirmación de cargue ignora multi-clic en B2C**  
- **Dado que** el usuario cargó un archivo CSV para **B2C** y está en la pantalla de previsualización  
- **Cuando** hace clic dos o más veces rápidamente en el botón **“Confirmar”**  
- **Entonces** el sistema debe ejecutar **una sola** confirmación del cargue  
- **Y** la interfaz debe mostrar feedback de procesamiento  
- **Y** al finalizar debe mostrar el resultado sin quedar bloqueada  

**Escenario: Descartar previsualización ignora multi-clic en B2B**  
- **Dado que** el usuario está en la pantalla de previsualización de cargue **B2B**  
- **Cuando** hace clic dos o más veces rápidamente en el botón **“Descartar”**  
- **Entonces** el sistema debe cerrar el modal/previsualización con el **primer** clic  
- **Y** los clics posteriores deben ser ignorados  
- **Y** el usuario debe quedar fuera de la previsualización en un estado navegable  

**Escenario: Descartar previsualización ignora multi-clic en B2C**  
- **Dado que** el usuario está en la pantalla de previsualización de cargue **B2C**  
- **Cuando** hace clic dos o más veces rápidamente en el botón **“Descartar”**  
- **Entonces** el sistema debe cerrar el modal/previsualización con el **primer** clic  
- **Y** los clics posteriores deben ser ignorados  
- **Y** el usuario debe quedar fuera de la previsualización en un estado navegable  

**Escenario: Manejo de error sin bloqueo**  
- **Dado que** el usuario está en previsualización y ejecuta **Confirmar** o **Descartar**  
- **Cuando** ocurre un error en la operación  
- **Entonces** el sistema debe informar el error al usuario  
- **Y** la interfaz no debe quedar bloqueada  
- **Y** el usuario debe poder reintentar o salir según corresponda  

---

## 5. Criterios de Aceptación Técnicos

1. **Bloqueo de doble ejecución (UI):** tras el primer clic, el botón correspondiente debe quedar deshabilitado (o la acción debe quedar “debounced/throttled”) hasta la finalización.
2. **Idempotencia (recomendado):** si por latencia/red llega más de una solicitud, el backend debe manejarla de forma idempotente o rechazar duplicadas según aplique (sin generar duplicidad de registros/efectos).
3. **Gestión de estados:** el front debe manejar estados explícitos: `idle | processing | success | error` (o equivalente) para evitar estados “zombies”.
4. **Trazabilidad mínima:** registrar en logs/aplicación el evento de “acción repetida ignorada” (si existe mecanismo de logging) para análisis de UX.

---

## 6. NFR / Requisitos No Funcionales

- **Usabilidad:** el usuario debe ver un indicador claro de que la acción fue recibida (spinner, estado deshabilitado, mensaje).
- **Performance percibida:** el sistema debe responder al primer clic con un cambio de estado visible en **< 300 ms** (deshabilitar/feedback), independiente de la respuesta del backend.
- **Confiabilidad:** no deben producirse bloqueos de pantalla que requieran recargar la página para continuar.

---

## 7. Riesgos

- Si solo se corrige en front sin control en backend, aún podría haber duplicidad si el usuario dispara solicitudes por otros medios (reintentos automáticos, latencia, etc.).
- Cambios en gestión de estados podrían afectar otros flujos del cargue (validaciones, navegación post-éxito).

---

## 8. QA / UAT

### Pruebas sugeridas (mínimo)
- Ejecutar casos para **B2B** y **B2C**:
  - Multi-clic rápido (2–10 clics) en **Confirmar**.
  - Multi-clic rápido (2–10 clics) en **Descartar**.
- Validar que:
  - No hay bloqueo de interfaz.
  - La previsualización se cierra correctamente al descartar.
  - La confirmación se ejecuta una sola vez (validar trazas/logs o evidencia de no duplicidad).
  - Se muestra feedback y mensajes de resultado.

---

## 9. Definición de Hecho (DoD)

- La HU cumple INVEST y está implementada para B2B y B2C.
- Criterios de aceptación funcionales y técnicos pasan en ambiente de pruebas.
- Evidencia de pruebas adjunta (capturas/logs) demostrando:
  - Deshabilitación/ignorancia de multi-clic.
  - No bloqueo de interfaz.
- Código revisado (peer review) y sin regresiones detectadas en el flujo de cargue.
