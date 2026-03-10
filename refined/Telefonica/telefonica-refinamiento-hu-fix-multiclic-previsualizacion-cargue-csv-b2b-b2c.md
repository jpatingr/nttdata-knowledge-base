# Fix multiclic en previsualización de cargue CSV B2B/B2C (Portal BI / Portal Transporte)

## 1. Definición de la Historia (Formato estándar)
**Como:** Usuario interno (Delivery – Cargue/Descargue de reportes)  
**Quiero:** que las acciones de **Confirmar** y **Descartar** en la previsualización del cargue CSV (B2B/B2C) sean **resilientes a clics repetidos** y mantengan el estado consistente  
**Para:** evitar bloqueos de UI, prevenir ejecuciones duplicadas y asegurar el cierre/avance correcto del flujo.

---

## 2. Contexto de Negocio
Esta HU pertenece al ecosistema del Portal BI/Portal Transporte (Delivery) y cubre el flujo de **cargue de reportes CSV** para **B2B y B2C**.

Hallazgos evidenciados en pruebas (CP_001 y CP_004):
- Ante clics rápidos repetidos en “Confirmar”/“Descartar”, la UI queda en estado bloqueado dentro de la previsualización (no permite salir).
- Aunque la confirmación llega a base de datos con el primer clic, la interfaz no refleja el resultado y queda inconsistente.
- Hallazgo de copy: mensaje con “archivox” al intentar guardar un archivo vacío.

---

## 3. Alcance / Fuera de Alcance

### Alcance
- Robustecer el comportamiento de los botones **Confirmar** y **Descartar** en la pantalla/modal de **previsualización** del cargue CSV para flujos **B2B** y **B2C**.
- Implementar control anti-multiclic en front-end (y refuerzo de idempotencia si existe un identificador de operación).
- Corregir el texto del modal de validación para archivo vacío (remover “archivox”).
- Asegurar que el flujo post-acción (cerrar modal, redirigir o refrescar grilla) quede consistente con el comportamiento actual.

### Fuera de alcance
- Cambios en el formato/estructura del CSV o reglas de negocio del cargue (mapeos, validaciones funcionales no relacionadas al vacío).
- Rediseño completo de la experiencia de cargue o reemplazo del componente de previsualización.
- Cambios de roles/permisos del módulo (solo se mantiene lo existente).
- Optimización general de performance de endpoints no relacionados a Confirmar/Descartar.

---

## 4. Reglas de Negocio
- **RB1 (Single-flight):** Cada acción (Confirmar/Descartar) debe ejecutarse como **una única operación activa** por instancia de previsualización.
- **RB2 (Inmutabilidad durante procesamiento):** Mientras una acción esté en curso, los botones relacionados deben permanecer deshabilitados para evitar cambios de estado concurrentes.
- **RB3 (Recuperabilidad):** Si la acción falla (timeout/error de backend), la UI debe volver a un estado recuperable (habilitar reintento o permitir volver/cerrar de forma segura).
- **RB4 (Validación archivo vacío):** No se permite avanzar a previsualización si el archivo está vacío; se debe mostrar mensaje correcto y consistente.

---

## 5. Criterios de Aceptación (Estándar NTT DATA) — Funcionales
- [ ] **CA1:** Al hacer el **primer clic** en **Confirmar**, el botón debe **deshabilitarse inmediatamente** y mostrar estado **Procesando**.
- [ ] **CA2:** Los clics subsiguientes en **Confirmar** deben ser **ignorados** (no deben disparar solicitudes duplicadas ni alterar el estado).
- [ ] **CA3:** Al completar la confirmación exitosamente, la UI debe **informar éxito** y **salir de la previsualización** según el flujo definido (cerrar modal/volver/refresh).
- [ ] **CA4:** Al hacer el **primer clic** en **Descartar**, la previsualización/modal debe **cerrarse** y la interfaz debe quedar **liberada**, sin posibilidad de quedar atrapado por clics repetidos.
- [ ] **CA5:** Si hay **latencia** o **error** del backend, la UI no debe congelarse; debe mostrarse mensaje claro y quedar en estado **recuperable** (reintentar o volver de forma segura).
- [ ] **CA6:** Al intentar **guardar un archivo vacío** (B2B/B2C), debe mostrarse un modal con mensaje correcto (sin “archivox”) y **no permitir avanzar** a previsualización.
- [ ] **CA7:** El comportamiento debe ser consistente en los casos de prueba **CP_001 (B2B)** y **CP_004 (B2C)**, evitando el estado de “bloqueo” descrito en evidencias.

---

## 5.1 Criterios de Aceptación (Gherkin)

### Escenario: Confirmar ignora multiclic y no bloquea UI
**Dado** que cargué un CSV (B2B o B2C) y estoy en la previsualización  
**Cuando** hago clic en “Confirmar”  
**Entonces** el botón “Confirmar” se deshabilita inmediatamente y se muestra “Procesando”  
**Y** los clics posteriores se ignoran  
**Y** al finalizar, se muestra mensaje de éxito y se sale de la previsualización.

### Escenario: Descartar cierra previsualización y no deja estado zombie
**Dado** que estoy en la previsualización  
**Cuando** hago clic en “Descartar” repetidamente  
**Entonces** el primer clic cierra la previsualización y libera la interfaz  
**Y** no queda la UI bloqueada ni atrapada en el modal.

### Escenario: Error/timeout permite recuperación segura
**Dado** que estoy en la previsualización  
**Y** el backend presenta latencia o error al confirmar/descartar  
**Cuando** hago clic en “Confirmar” o “Descartar”  
**Entonces** se muestra un mensaje claro de error/timeout  
**Y** la pantalla queda en un estado recuperable (reintentar o volver/cerrar sin bloqueo)  
**Y** no se envían solicitudes duplicadas por multiclic.

### Escenario: Archivo vacío muestra mensaje correcto
**Dado** que selecciono un archivo vacío para B2B o B2C  
**Cuando** hago clic en “Guardar”  
**Entonces** se muestra un modal de validación con mensaje correcto (sin “archivox”)  
**Y** no se permite avanzar a previsualización.

---

## 5.2 Criterios de Aceptación — Técnicos (alineado a estándar)
- [ ] **Arquitectura:** El control anti-multiclic debe implementarse en front (single-flight / lock por acción) y, si aplica, reforzarse con idempotencia en backend por `uploadId` o equivalente.
- [ ] **Rendimiento:** El cambio no debe degradar la interacción; la UI no debe bloquear el hilo principal ni generar listeners duplicados.
- [ ] **Seguridad:** Mantener controles existentes de acceso/roles del módulo (no exponer información adicional por mensajes).
- [ ] **Observabilidad:** Registrar eventos básicos (confirm/descartar, error/timeout, duplicidad ignorada) para troubleshooting.

---

## 6. NFR / Requisitos No Funcionales
- **Usabilidad:** feedback visual inmediato al iniciar Confirmar/Descartar (disabled + spinner/estado “Procesando”).
- **Confiabilidad:** no deben existir estados “zombie” donde el usuario no pueda salir de previsualización.
- **Compatibilidad:** aplicar en B2B y B2C sin cambios en contratos existentes (si los hay).
- **Tiempo de respuesta UI:** deshabilitado/feedback debe ocurrir en < 200ms desde el primer clic (sin esperar respuesta del backend).

---

## 7. Notas UX/UI
- Cambiar el texto “Procesando”/loader de manera consistente con el design system del portal.
- Asegurar que el botón deshabilitado tenga un estado visual claro.
- Mensajes de error: incluir acción sugerida (ej. “Intente nuevamente” o “Vuelva y reintente el cargue”).

---

## 8. Riesgos
- Condiciones de carrera si existen múltiples listeners/eventos duplicados en el componente de previsualización.
- Doble-submit por re-render o re-montaje del componente si no se controla el estado de “in flight”.
- Impacto colateral en el flujo post-éxito (refresh/redirección) si está acoplado a callbacks actuales.

---

## 9. QA / UAT
- Ejecutar y evidenciar que **CP_001** y **CP_004** pasan: multiclic no bloquea la UI.
- Prueba de regresión:
  - Confirmar éxito (primer clic) y cierre/avance correcto.
  - Descartar cierre correcto.
  - Error/timeout: estado recuperable.
  - Archivo vacío: modal con texto correcto, sin avance.
- Validar en ambiente de pruebas desarrollo con usuarios/roles existentes.

---

## 10. Definición de Hecho (DoD)
- Código revisado por par (peer review).
- Pruebas unitarias y/o de componente para el control anti-multiclic (mínimo donde aplique).
- Pruebas QA: reproducción de CP_001 y CP_004 ya no ocurre; validación B2B/B2C.
- Mensaje “archivox” corregido y validado.
- Documentación técnica/nota de cambio actualizada si el equipo la exige.
