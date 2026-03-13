# HU-XXX - Control de múltiple clic en confirmación y descarte de cargue CSV en Delivery

## 👤 Definición

**Como** Analista de Cliente Interno del Portal BI,  
**Quiero** que el sistema controle y deshabilite los botones “Confirmar” y “Descartar” durante el proceso de cargue de archivos CSV B2B y B2C,  
**Para** evitar bloqueos de interfaz, estados inconsistentes y registros duplicados en base de datos.

---

## 📌 Contexto de Negocio

En el módulo **Delivery – Cargue/Descargue de reportes** del Portal BI, se identificó mediante pruebas funcionales que al realizar múltiples clics rápidos sobre los botones “Confirmar” o “Descartar” durante la previsualización del archivo CSV, la interfaz queda bloqueada.

Aunque la base de datos procesa el primer clic correctamente, la UI permanece en estado inconsistente, impidiendo al usuario continuar operando con normalidad. Esto afecta la experiencia del cliente interno y genera riesgo de duplicidad o inconsistencias operativas.

---

## 🎯 Alcance

Incluye:
- Control de eventos de múltiple clic en botones “Confirmar” y “Descartar”.
- Deshabilitar botones inmediatamente después del primer clic válido.
- Manejo adecuado del estado de la modal de previsualización.
- Corrección del mensaje ortográfico en validación de archivo vacío.

No incluye:
- Cambios estructurales al proceso de cargue CSV.
- Modificaciones al modelo de datos existente.
- Nuevos tipos de archivo distintos a CSV B2B y B2C.

---

## 📜 Reglas de Negocio

1. El sistema debe aceptar únicamente el primer clic ejecutado sobre los botones “Confirmar” o “Descartar”.
2. Una vez ejecutada la acción, el botón debe deshabilitarse inmediatamente.
3. No deben generarse procesos duplicados en backend ante múltiples clics.
4. El mensaje de validación de archivo vacío debe mostrar el texto correcto (“archivo” o “archivos”).
5. La interfaz nunca debe quedar bloqueada tras una operación válida.

---

## ✅ Criterios de Aceptación Funcionales (BDD)

### Escenario 1: Confirmación exitosa con control de múltiple clic

**Dado que** el usuario ha cargado un archivo CSV válido (B2B o B2C) y se encuentra en la pantalla de previsualización,  
**Cuando** hace clic en el botón “Confirmar”,  
**Entonces** el sistema debe procesar el cargue una única vez,  
**Y** el botón debe deshabilitarse inmediatamente,  
**Y** debe mostrarse un mensaje de éxito,  
**Y** la interfaz debe liberarse correctamente.

---

### Escenario 2: Descarte con control de múltiple clic

**Dado que** el usuario se encuentra en la pantalla de previsualización,  
**Cuando** hace clic en el botón “Descartar”,  
**Entonces** la modal debe cerrarse inmediatamente,  
**Y** la interfaz debe volver al estado inicial,  
**Y** clics posteriores no deben generar comportamiento adicional.

---

### Escenario 3: Prevención de bloqueo por clics repetidos

**Dado que** el usuario hace clic rápido dos o más veces en “Confirmar” o “Descartar”,  
**Cuando** el sistema recibe múltiples eventos consecutivos,  
**Entonces** solo el primer evento debe ejecutarse,  
**Y** los demás deben ser ignorados,  
**Y** la aplicación no debe quedar en estado bloqueado.

---

### Escenario 4: Validación de archivo vacío

**Dado que** el usuario carga un archivo vacío,  
**Cuando** hace clic en “Guardar”,  
**Entonces** el sistema debe mostrar una modal de error con el texto correctamente escrito (“archivo” o “archivos”),  
**Y** no debe permitir continuar con el proceso de cargue.

---

## 🔧 Criterios de Aceptación Técnicos

1. Implementar control de idempotencia en frontend para evitar múltiples ejecuciones del mismo evento.
2. Deshabilitar el botón vía estado reactivo inmediatamente tras el primer clic.
3. Validar en backend que no se procesen solicitudes duplicadas provenientes del mismo archivo en la misma sesión.
4. Garantizar que el estado de la modal se gestione mediante control explícito (open/close state).
5. Corregir literal defectuoso identificado en el mensaje de validación.

---

## 🚦 Requisitos No Funcionales (NFR)

- **Usabilidad:** La respuesta visual del botón debe ser inmediata (< 200ms).
- **Confiabilidad:** No deben generarse registros duplicados ante clics repetidos.
- **Consistencia:** La interfaz no debe quedar en estado bloqueado bajo ninguna circunstancia validada.
- **Mantenibilidad:** El control implementado debe ser reutilizable para otros botones críticos del módulo.

---

## 🎨 Notas UX/UI

- Los botones deben mostrar estado “loading” o deshabilitado tras el primer clic.
- El usuario debe recibir retroalimentación visual clara (spinner o cambio de estado).
- La modal debe cerrarse automáticamente tras éxito o descarte válido.
- El mensaje de error ortográfico debe corregirse sin alterar la estructura visual existente.

---

## ⚠️ Riesgos

- Posibles efectos secundarios en otros flujos si el control de eventos no está correctamente encapsulado.
- Dependencia del manejo correcto de estados en frontend.
- Riesgo de no detectar procesos duplicados si el backend no valida idempotencia.

---

## 🧪 QA / UAT

- Validar comportamiento con múltiples clics manuales y automatizados.
- Probar en B2B y B2C.
- Validar que solo se genere un registro en base de datos por cargue.
- Validar corrección ortográfica del mensaje.
- Ejecutar pruebas regresivas sobre el módulo Delivery.

---

## ✅ Definición de Hecho (DoD)

- Código implementado y revisado por pares.
- Pruebas unitarias y funcionales exitosas.
- Sin bloqueos de interfaz reproducibles.
- Sin duplicidad de registros en base de datos.
- Validación aprobada por QA.
- Documentación actualizada en Portal BI.
