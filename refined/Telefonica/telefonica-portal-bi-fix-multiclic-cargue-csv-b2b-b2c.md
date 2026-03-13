# Fix multiclic y corrección de validaciones en cargue CSV B2B/B2C – Portal BI

## 1. Definición (Como / Quiero / Para)

**Como:** Usuario interno del módulo Delivery (Cargue/Descargue de reportes)  
**Quiero:** que las acciones de Confirmar y Descartar en la previsualización del cargue CSV (B2B/B2C) gestionen correctamente los clics repetidos y que los mensajes de validación sean correctos  
**Para:** evitar bloqueos de la interfaz, prevenir ejecuciones duplicadas y asegurar un flujo consistente y confiable de cargue.

---

## 2. Contexto de Negocio

Durante la ejecución de pruebas funcionales en ambiente de desarrollo (08/01/2026) para la HU de cliente interno – Delivery, se identificaron fallos en los casos:

- CP_001 – Carga CSV B2B  
- CP_004 – Carga CSV B2C  

Se evidenció que, al realizar clics rápidos y repetidos sobre los botones “Confirmar” o “Descartar” en la pantalla de previsualización:

- La interfaz pierde capacidad de respuesta.
- El usuario queda atrapado en la previsualización.
- Aunque el primer clic sí ejecuta la operación en base de datos, la UI queda en estado inconsistente.
- Existe un error ortográfico en el modal al cargar un archivo vacío (“archivox”).

Esta HU corresponde a un evolutivo correctivo sobre el Portal BI (módulo Delivery).

---

## 3. Alcance / Fuera de Alcance

### Alcance
- Control anti-multiclic en botones “Confirmar” y “Descartar”.
- Manejo adecuado de estados de procesamiento.
- Corrección del mensaje mostrado al cargar archivo vacío.
- Asegurar consistencia entre estado backend y estado UI.

### Fuera de Alcance
- Cambios en la estructura del archivo CSV.
- Modificaciones en reglas de negocio de validación del contenido del archivo.
- Rediseño visual completo del flujo de cargue.

---

## 4. Reglas de Negocio

- El cargue debe ejecutarse una sola vez por archivo.
- La confirmación exitosa debe reflejarse inmediatamente en la interfaz.
- No se deben generar procesos duplicados por acciones repetidas del usuario.
- No se debe permitir avanzar a previsualización si el archivo está vacío.

---

## 5. Criterios de Aceptación Funcionales

- [ ] **CA1:** Al hacer el primer clic en “Confirmar”, el botón debe deshabilitarse inmediatamente y mostrar estado “Procesando”.
- [ ] **CA2:** Los clics posteriores sobre “Confirmar” deben ser ignorados y no deben generar solicitudes adicionales al backend.
- [ ] **CA3:** Al completarse exitosamente la confirmación, se debe mostrar mensaje de éxito y cerrar la previsualización según flujo definido.
- [ ] **CA4:** Al hacer el primer clic en “Descartar”, la previsualización debe cerrarse y liberar la interfaz.
- [ ] **CA5:** Los clics repetidos sobre “Descartar” no deben generar estados inconsistentes ni bloquear la UI.
- [ ] **CA6:** Si ocurre error o latencia del backend, la UI debe mostrar mensaje claro y permitir recuperación controlada.
- [ ] **CA7:** Al intentar cargar un archivo vacío, se debe mostrar un modal con mensaje correcto (sin la palabra “archivox”) y no permitir avanzar.

---

## 6. Criterios de Aceptación Técnicos

- [ ] **Arquitectura:** Implementar control anti-multiclic en front (lock por acción / single-flight) y validar idempotencia en backend cuando aplique.
- [ ] **Gestión de Estado:** Garantizar sincronización entre estado del backend y estado visual del modal.
- [ ] **Observabilidad:** Registrar eventos básicos (confirmación, descarte, error, duplicidad ignorada).
- [ ] **Calidad de Código:** No generar listeners duplicados ni bloqueos del hilo principal.

---

## 7. NFR / Requisitos No Funcionales

- **Rendimiento:** El cambio no debe incrementar el tiempo de respuesta perceptible para el usuario.
- **Usabilidad:** La interfaz debe reflejar claramente estados: Procesando, Éxito, Error.
- **Confiabilidad:** No deben generarse confirmaciones duplicadas en base de datos.
- **Seguridad:** Mantener controles actuales de roles y permisos del módulo Delivery.

---

## 8. Notas UX/UI

- El botón presionado debe mostrar feedback visual inmediato (disabled + estado Procesando).
- El cierre del modal debe ser inmediato en caso de “Descartar”.
- Los mensajes de validación deben ser claros, sin errores ortográficos.
- Se recomienda aplicar debounce/throttle adicional para evitar condiciones de carrera.

---

## 9. Riesgos

- Condiciones de carrera si no se implementa correctamente el control de estado.
- Desalineación entre respuesta backend y renderizado del front.
- Impacto en flujos B2B y B2C si la solución no se valida en ambos escenarios.

---

## 10. QA / UAT

Se debe validar:

- Reproducción de CP_001 y CP_004 ya no ocurre.
- Confirmar que múltiples clics no bloquean la UI.
- Verificar que solo se registra una confirmación en base de datos.
- Validar corrección del mensaje de archivo vacío.
- Probar en B2B y B2C en ambiente de pruebas.

---

## 11. Definición de Hecho (DoD)

- Código revisado por par (Peer Review).
- Pruebas unitarias o de componente para control anti-multiclic.
- Validación QA exitosa en B2B y B2C.
- Mensaje ortográfico corregido y aprobado por negocio.
- Documentación técnica actualizada si aplica.
