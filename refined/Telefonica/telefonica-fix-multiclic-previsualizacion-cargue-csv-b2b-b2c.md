# Fix multiclic en previsualización de cargue CSV B2B/B2C (Portal BI / Portal Transporte)

## 1. Definición de la Historia (Como / Quiero / Para)
**Como:** Usuario interno (Delivery – Cargue/Descargue de reportes)  
**Quiero:** que las acciones de **Confirmar** y **Descartar** en la previsualización del cargue CSV (B2B/B2C) sean **resilientes a clics repetidos**, manteniendo el estado consistente y evitando ejecuciones duplicadas  
**Para:** prevenir bloqueos de UI, evitar confirmaciones/operaciones repetidas y asegurar el cierre/avance correcto del flujo de cargue.

---

## 2. Contexto de Negocio
Esta HU pertenece al ecosistema del **Portal Transporte / Portal BI (módulo Delivery – Cargue/Descargue de reportes)** y cubre el flujo de **cargue de reportes CSV** para **B2B** y **B2C**.

En ejecución de pruebas (casos **CP_001** y **CP_004**) se evidenció que ante clics rápidos repetidos en los botones **“Confirmar”** o **“Descartar”** dentro de la **previsualización**, la aplicación:
- queda atrapada dentro del modal/pantalla de previsualización,
- pierde capacidad de respuesta (UI bloqueada),
- aunque el backend/BD registra la confirmación con el primer clic, la UI no refleja el resultado y queda en estado inconsistente.

Adicionalmente, se detectó un **hallazgo de copy**: al intentar guardar un archivo vacío (B2B/B2C), el mensaje muestra **“archivox”** en lugar de **“archivo/archivos”**.

---

## 3. Alcance / Fuera de Alcance
**Alcance**
- Implementar control anti-multiclic para las acciones **Confirmar** y **Descartar** en la previsualización del cargue CSV (B2B/B2C).
- Asegurar que el flujo finalice correctamente (cierre de previsualización y liberación de la UI) ante éxito, error o latencia.
- Corregir el texto del modal de validación para archivo vacío (eliminar “archivox”).

**Fuera de alcance**
- Cambios en el formato/estructura del CSV.
- Cambios funcionales al proceso de negocio del cargue (más allá de anti-multiclic, recuperación y copy).
- Re-diseño visual del módulo (solo ajustes de estados/disabled/mensajería necesarios).

---

## 4. Reglas de Negocio
- La previsualización debe permitir una única ejecución efectiva por acción (**Confirmar** o **Descartar**) por intento de cargue.
- Si se inicia el procesamiento de **Confirmar**, la UI debe impedir la repetición de la acción hasta finalizar (éxito o fallo controlado).
- Si el usuario ejecuta **Descartar**, el flujo debe cerrarse y liberar el estado de la interfaz de forma inmediata con el primer clic.
- No se debe permitir avanzar a previsualización si el archivo seleccionado está vacío; se debe informar con mensaje correcto.

---

## 5. Criterios de Aceptación Funcionales
- [ ] **CA1:** En previsualización, al hacer el **primer clic** en **“Confirmar”**, el sistema debe **deshabilitar inmediatamente** el botón y mostrar un estado visible de **Procesando** (o equivalente).  
- [ ] **CA2:** Cualquier clic posterior en **“Confirmar”** mientras exista una operación en curso debe ser **ignorado** (no dispara solicitudes duplicadas ni cambia el estado).  
- [ ] **CA3:** Al finalizar la confirmación de forma **exitosa**, la UI debe **notificar éxito** y **salir/cerrar** la previsualización según el flujo actual del portal.  
- [ ] **CA4:** En previsualización, al hacer el **primer clic** en **“Descartar”**, la previsualización/modal debe **cerrarse** y la interfaz debe quedar **liberada**.  
- [ ] **CA5:** Clics subsiguientes en **“Descartar”** no deben provocar estados inconsistentes ni reabrir/bloquear la UI.  
- [ ] **CA6:** Si ocurre **latencia** o **error** en backend durante “Confirmar”, la UI no debe congelarse; debe mostrar un mensaje claro y quedar en un estado **recuperable** (permitir reintento o volver de forma segura).  
- [ ] **CA7:** Al intentar **Guardar** un archivo vacío (B2B/B2C), el sistema debe mostrar un modal de validación con mensaje correcto (sin “archivox”) y **no permitir** avanzar a la previsualización.

---

## 6. Criterios de Aceptación Técnicos
- [ ] **Arquitectura:** Implementar control anti-multiclic en front bajo patrón **single-flight** / **lock por acción** (p. ej. bandera “inProgress” por operación y por intento de cargue).  
- [ ] **Idempotencia (si aplica):** Si el backend soporta un identificador de cargue (p. ej. `uploadId`), reforzar idempotencia para que múltiples solicitudes equivalentes no generen efectos duplicados.  
- [ ] **Rendimiento:** El control anti-multiclic no debe degradar la interacción; no debe bloquear el hilo principal ni generar listeners duplicados.  
- [ ] **Manejo de errores:** Para timeouts/errores, asegurar limpieza de estado (re-habilitar acciones según corresponda) y mensajería consistente.  
- [ ] **Observabilidad:** Registrar eventos mínimos (confirmar/descartar, error/timeout, multiclic ignorado) para troubleshooting, sin exponer información sensible.  
- [ ] **Seguridad:** Mantener controles existentes de acceso/roles del módulo; no exponer información adicional por mensajes o logs.

---

## 7. NFR / Requisitos No Funcionales
- **Usabilidad:** La UI debe reflejar de forma inmediata el cambio de estado (disabled/loading) para reducir la posibilidad de clics repetidos.  
- **Confiabilidad:** El flujo no debe quedar en estado “zombie” atrapado en previsualización; debe existir salida controlada ante éxito, error o cancelación.  
- **Compatibilidad:** El cambio debe aplicar de forma homogénea para B2B y B2C en los puntos de previsualización del módulo Delivery.  

---

## 8. Riesgos
- Riesgo de condiciones de carrera si existen múltiples puntos que disparan “Confirmar/Descartar” (duplicidad de handlers).  
- Si el backend no es idempotente, múltiples clics podrían generar efectos duplicados; por ello el control de UI debe ser inmediato y robusto.  
- Cambios en estados de UI pueden afectar el flujo actual (cierre modal, refresco de grilla) si no se respeta la secuencia existente.

---

## 9. QA / UAT
- [ ] Reproducir y validar corrección de los casos **CP_001 (B2B)** y **CP_004 (B2C)**: clics rápidos repetidos en “Confirmar/Descartar” no bloquean la interfaz ni dejan atrapado al usuario.  
- [ ] Validar que ante éxito de confirmación se muestra mensaje y se sale de previsualización (sin doble envío).  
- [ ] Validar escenario de error/latencia: UI recuperable, sin congelamiento.  
- [ ] Validar carga de **archivo vacío**: se muestra modal con copy correcto (sin “archivox”) y no avanza a previsualización.  
- [ ] Validación cross-browser según matriz del proyecto (si aplica).

---

## 10. Definición de Hecho (DoD)
- Código revisado por par (Peer Review).  
- Pruebas unitarias y/o de componente para el control anti-multiclic (mínimo donde aplique).  
- Evidencia QA: reproducción de CP_001 y CP_004 ya no ocurre; validación B2B/B2C.  
- Copy “archivox” corregido y validado.  
- Documentación técnica/nota de cambio actualizada si el equipo la exige.
