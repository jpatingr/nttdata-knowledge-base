---
empresa: Telefonica
proyecto: Portal BI
fuente_input: Telefonica_reporte-hallazgos_carga.pdf
fuente_estandar: empresa
estado: Awaiting_Approval
---

# Fix multiclic en previsualización de cargue CSV B2B/B2C (Portal BI / Portal Transporte)

## 1. Definición de la Historia (Formato estándar)
**Como:** Usuario interno (Delivery – Cargue/Descargue de reportes)  
**Quiero:** que las acciones de **Confirmar** y **Descartar** en la previsualización del cargue CSV (B2B/B2C) sean **resilientes a clics repetidos** y mantengan el estado consistente  
**Para:** evitar bloqueos de UI, prevenir ejecuciones duplicadas y asegurar el cierre/avance correcto del flujo.

---

## 2. Contexto de Negocio
Esta HU pertenece al ecosistema del Portal BI / Portal Transporte (Delivery) y cubre el flujo de **cargue de reportes CSV** para **B2B y B2C**.

Hallazgos evidenciados en pruebas (ambiente desarrollo):
- **CP_001 (B2B) / CP_004 (B2C):** ante clics rápidos repetidos en “Confirmar”/“Descartar”, la UI queda en estado bloqueado dentro de la previsualización (no permite salir).
- Aunque la **confirmación llega a base de datos con el primer clic**, la interfaz no refleja el resultado y queda inconsistente.
- **Hallazgo de copy:** mensaje con “archivox” al intentar guardar un archivo vacío.

---

## 3. Criterios de Aceptación (Estándar NTT DATA) — Funcionales
- [ ] **CA1:** Al hacer el **primer clic** en **Confirmar**, el botón debe **deshabilitarse inmediatamente** y mostrar estado **Procesando**.
- [ ] **CA2:** Los clics subsiguientes en **Confirmar** deben ser **ignorados** (no deben disparar solicitudes duplicadas ni alterar el estado).
- [ ] **CA3:** Al completar la confirmación exitosamente, la UI debe **informar éxito** y **salir de la previsualización** según el flujo definido.
- [ ] **CA4:** Al hacer el **primer clic** en **Descartar**, la previsualización/modal debe **cerrarse** y la interfaz debe quedar **liberada**, sin posibilidad de quedar atrapada por clics repetidos.
- [ ] **CA5:** Si hay **latencia** o **error** del backend, la UI no debe congelarse; debe mostrarse mensaje claro y quedar en estado **recuperable** (reintentar o volver de forma segura).
- [ ] **CA6:** Al intentar **guardar un archivo vacío** (B2B/B2C), debe mostrarse un modal con mensaje correcto (sin “archivox”) y **no permitir avanzar** a previsualización.

---

## 3.1 Criterios de Aceptación (Gherkin)

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

### Escenario: Archivo vacío muestra mensaje correcto
**Dado** que selecciono un archivo vacío para B2B o B2C  
**Cuando** hago clic en “Guardar”  
**Entonces** se muestra un modal de validación con mensaje correcto (sin “archivox”)  
**Y** no se permite avanzar a previsualización.

---

## 3.2 Criterios de Aceptación — Técnicos (alineado a estándar)
- [ ] **Arquitectura:** El control anti-multiclic debe implementarse en front (single-flight / lock por acción) y, si aplica, reforzarse con idempotencia en backend por `uploadId` o equivalente.
- [ ] **Rendimiento:** El cambio no debe degradar la interacción; la UI no debe bloquear el hilo principal ni generar listeners duplicados.
- [ ] **Seguridad:** Mantener controles existentes de acceso/roles del módulo (no exponer información adicional por mensajes).
- [ ] **Observabilidad:** Registrar eventos básicos (confirm/descartar, error/timeout, duplicidad ignorada) para troubleshooting.

---

## 4. Definición de Hecho (DoD)
- Código revisado por par (peer review).
- Pruebas unitarias y/o de componente para el control anti-multiclic (mínimo donde aplique).
- Pruebas QA: reproducción de CP_001 y CP_004 ya no ocurre; validación B2B/B2C.
- Mensaje “archivox” corregido y validado.
- Documentación técnica/nota de cambio actualizada si el equipo la exige.

---

## 5. Notas adicionales
- El comportamiento post-éxito (redirigir, cerrar modal y refrescar grilla, etc.) debe mantenerse consistente con el flujo actual del portal.
- Recomendación: además del disabled, aplicar **debounce/throttle** y/o **guardas de estado** para evitar condiciones de carrera.
