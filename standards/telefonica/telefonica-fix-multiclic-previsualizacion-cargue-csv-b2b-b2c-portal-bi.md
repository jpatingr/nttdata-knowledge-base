# Fix multiclic y consistencia de estado en previsualización de cargue CSV B2B/B2C (Portal BI)

## 1. Definición
**Como:** Usuario interno (Delivery – Cargue/Descargue de reportes)  
**Quiero:** que las acciones de **Confirmar** y **Descartar** en la previsualización del cargue CSV (B2B/B2C) sean **resilientes a clics repetidos**, gestionen correctamente el estado de carga y reflejen el resultado real del proceso  
**Para:** evitar bloqueos de UI, prevenir ejecuciones duplicadas, asegurar un flujo consistente (confirmación/descartar) y reducir reprocesos/incidencias operativas.

---

## 2. Contexto de Negocio
En el Portal BI (Delivery: cargue/descargue de reportes), durante la ejecución de pruebas (CP_001 B2B y CP_004 B2C) se evidenció que ante clics rápidos repetidos sobre **Confirmar** o **Descartar** dentro de la previsualización del cargue CSV:

- La UI puede quedar en un estado inconsistente (“bloqueada”/“zombie”), atrapando al usuario en la pantalla de previsualización.
- Se observa que **la confirmación puede llegar a base de datos con el primer clic**, pero la UI no presenta el resultado ni avanza/cierra correctamente.
- Existe un hallazgo de calidad de copy: al intentar guardar un archivo vacío, el modal muestra “**archivox**” en vez de “archivo/archivos”.

Este evolutivo/correctivo busca asegurar que el front gestione el flujo como **single-flight** (una sola acción efectiva), con señales claras al usuario, manejo recuperable de error/latencia, y copy validado.

---

## 3. Alcance / Fuera de Alcance

### Alcance
- Flujo: **Cargue CSV** → **Previsualización** → acciones **Confirmar** y **Descartar**.
- Aplicabilidad: **B2B y B2C**.
- Prevención de multiclic/doble envío y consistencia de estado UI.
- Manejo de estados: *idle*, *processing*, *success*, *error/timeout*.
- Corrección del mensaje “archivox” en el modal de validación de archivo vacío.

### Fuera de Alcance
- Cambios en reglas de negocio del procesamiento del CSV (validaciones del contenido/estructura) diferentes a la validación de archivo vacío.
- Cambios de modelo de datos, transformación de archivos o rediseño del flujo completo.
- Cambios de permisos/roles existentes del módulo.

---

## 4. Reglas de Negocio
1. **Single-flight por acción (front):** una vez el usuario dispare **Confirmar** o **Descartar**, se debe bloquear la posibilidad de disparar la misma acción nuevamente hasta resolver el estado (éxito/error/cierre).
2. **Deshabilitado inmediato:** el botón accionado debe deshabilitarse inmediatamente (sin esperar respuesta del backend) y reflejar estado **Procesando**.
3. **Ignorar multiclic:** clics posteriores deben ser ignorados (sin duplicar requests ni alterar el estado).
4. **Consistencia UI ↔ resultado:** la UI debe reflejar el resultado real del proceso (éxito/error) y retornar a un estado navegable.
5. **Recuperabilidad:** ante error/timeout/latencia, la UI no debe congelarse; debe mostrar un mensaje claro y permitir reintentar o volver de forma segura.
6. **Validación archivo vacío:** si el archivo es vacío, debe mostrarse validación y no permitir avanzar a previsualización/confirmación. El texto debe ser correcto (sin “archivox”).

---

## 5. Criterios de Aceptación (Estándar NTT DATA) — Funcionales
- [ ] **CA1:** En previsualización (B2B/B2C), al hacer el **primer clic** en **Confirmar**, el botón debe **deshabilitarse inmediatamente** y mostrar estado **Procesando** (spinner o label).
- [ ] **CA2:** Los clics subsiguientes en **Confirmar** deben ser **ignorados** y **no** deben generar solicitudes duplicadas ni cambios de estado inconsistentes.
- [ ] **CA3:** Al finalizar la confirmación exitosamente, la UI debe **informar éxito** (mensaje/modal/toast según estándar visual del portal) y **salir de la previsualización** siguiendo el flujo vigente (cierre de modal / navegación / refresco de grilla).
- [ ] **CA4:** En previsualización, al hacer el **primer clic** en **Descartar**, la previsualización/modal debe **cerrarse** y la interfaz debe quedar **liberada**.
- [ ] **CA5:** Los clics repetidos en **Descartar** no deben dejar la UI en estado bloqueado ni impedir el cierre/retorno a la pantalla anterior.
- [ ] **CA6:** Ante **latencia**, **timeout** o **error** del backend durante Confirmar, la UI debe:
  - mostrar mensaje claro (sin términos técnicos),
  - quedar en estado **recuperable** (habilitar reintentar o permitir volver),
  - y evitar duplicidad de confirmaciones.
- [ ] **CA7:** Al intentar **guardar un archivo vacío** (B2B/B2C), el sistema debe impedir avanzar y mostrar un modal/mensaje con copy correcto (sin “archivox”).
- [ ] **CA8:** En ningún caso el usuario debe quedar “atrapado” en la previsualización sin posibilidad de salir (por Confirmar/Descartar o por cierre controlado del modal/pantalla).

---

## 6. Criterios de Aceptación (Gherkin)

### Escenario 1: Confirmar ignora multiclic y no bloquea UI
**Dado** que cargué un CSV (B2B o B2C) y estoy en la previsualización  
**Cuando** hago clic en “Confirmar”  
**Entonces** el botón “Confirmar” se deshabilita inmediatamente y se muestra “Procesando”  
**Y** los clics posteriores en “Confirmar” se ignoran  
**Y** al finalizar, se muestra mensaje de éxito y se sale de la previsualización según el flujo definido.

### Escenario 2: Descartar cierra previsualización y no deja estado zombie
**Dado** que estoy en la previsualización  
**Cuando** hago clic en “Descartar” repetidamente  
**Entonces** el primer clic cierra la previsualización y libera la interfaz  
**Y** la UI no queda bloqueada ni atrapada en el modal/pantalla.

### Escenario 3: Error o timeout en Confirmar no congela la interfaz
**Dado** que estoy en la previsualización  
**Y** el servicio de confirmación presenta error o demora excediendo el tiempo configurado  
**Cuando** hago clic en “Confirmar”  
**Entonces** la UI muestra un mensaje de error/tiempo de espera entendible  
**Y** permite reintentar o volver de forma segura  
**Y** no se ejecutan confirmaciones duplicadas por multiclic.

### Escenario 4: Archivo vacío muestra mensaje correcto y bloquea el avance
**Dado** que selecciono un archivo vacío para B2B o B2C  
**Cuando** hago clic en “Guardar”  
**Entonces** se muestra un modal de validación con mensaje correcto (sin “archivox”)  
**Y** no se permite avanzar a previsualización.

---

## 7. NFR / Requisitos No Funcionales
- **Rendimiento:** la solución no debe degradar la interacción del usuario; el control de estado no debe bloquear el hilo principal ni generar listeners/handlers duplicados.
- **Confiabilidad:** la UI debe conservar consistencia ante reintentos, recargas o fallas de red; no debe quedar en estado intermedio indefinido.
- **Accesibilidad (WCAG 2.1):** cambios de estado (Procesando/Éxito/Error) deben ser perceptibles (texto + feedback visual) y accesibles para lector de pantalla si aplica.
- **Observabilidad:** registrar eventos mínimos para troubleshooting:
  - acción (confirm/descartar),
  - inicio/fin,
  - error/timeout,
  - multiclic ignorado (cuando aplique).

---

## 8. Notas UX/UI
- Deshabilitar el botón accionado en el **primer clic** y mostrar “Procesando…” con feedback visual consistente con la librería de componentes del portal.
- Evitar que el usuario pueda disparar confirmación/descartar desde estados intermedios.
- Mensajes:
  - Éxito: confirmación realizada.
  - Error/timeout: explicación breve + acción sugerida (reintentar o volver).
  - Validación archivo vacío: copy correcto (“El archivo está vacío” / “Selecciona un archivo con contenido”, según estándar del portal).

---

## 9. Riesgos
- Condiciones de carrera entre estado front y respuesta backend (especialmente si el backend procesa rápido y el front no actualiza correctamente).
- Falta de idempotencia en backend podría permitir duplicidad si existe reintento legítimo (mitigar con identificador de operación si aplica).
- Diferencias de implementación entre flujo B2B y B2C (asegurar comportamiento equivalente).

---

## 10. QA / UAT
- Re-ejecutar casos: **CP_001 (B2B)** y **CP_004 (B2C)**, validando que:
  - no existe bloqueo al hacer multiclic,
  - la UI cierra/avanza correctamente,
  - y el estado es recuperable ante error/latencia.
- Prueba específica de archivo vacío (B2B/B2C): se bloquea avance y copy sin “archivox”.
- Pruebas regresión del flujo de cargue CSV completo (happy path y error path).

---

## 11. Definición de Hecho (DoD)
- Código revisado por par (peer review).
- Pruebas unitarias y/o de componente para el control anti-multiclic (mínimo donde aplique).
- Pruebas QA: reproducción de CP_001 y CP_004 ya no ocurre; validación B2B/B2C.
- Mensaje “archivox” corregido y validado.
- Documentación técnica/nota de cambio actualizada si el equipo la exige.

---

status: Awaiting_Approval
