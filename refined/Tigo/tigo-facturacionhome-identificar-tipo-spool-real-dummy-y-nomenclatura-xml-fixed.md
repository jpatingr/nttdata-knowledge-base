# HU — Identificar tipo de spool (REAL vs DUMMY) y aplicar nomenclatura de archivos para facturación fija B2C Hogar

**Como** Sistema generador/orquestador de facturación fija (CBS Fijo) integrado al Gestor Documental,  
**Quiero** identificar si un spool de entrada corresponde a facturas **DUMMY** o **REAL** a partir del campo `ROUTE_ID` y generar/validar el nombre del archivo del spool con la nomenclatura definida,  
**Para** enrutar correctamente los archivos a sus buckets/rutas (pruebas vs ciclo real), soportar impresión/distribución, y asegurar consistencia operativa y cumplimiento regulatorio.

---

## 1. Contexto de Negocio

El programa **“Nueva Factura en Gestor Documental”** busca unificar y parametrizar las plantillas actuales en un nuevo gestor documental para soportar **B2C Hogar (CBS Fijo)** y preparar **B2B** mediante `business_unit` y `network_teletype`, garantizando consistencia visual/regulatoria y mantenibilidad.

Para iniciar el procesamiento del ciclo de facturación, el sistema recibe spools (tar.gz) con XMLs provenientes de CBS Fijo. Es crítico distinguir entre:
- **Spool REAL**: facturas que consumen numeración DIAN (`ROUTE_ID = FIX`).
- **Spool DUMMY**: facturas de prueba/internas (`ROUTE_ID = FIX_INTERNAL`).

Sin esta clasificación y una nomenclatura consistente, el pipeline de ingestión y generación de PDFs puede enrutar mal archivos, mezclar pruebas con ciclo real y generar resultados no auditables.

---

## 2. Alcance / Fuera de Alcance

### Alcance
- Determinar el tipo de spool con base en `BILL_INFO/BILL_RUN/ACCT_INFO/ACCT_PROP/ROUTE_ID`.
- Aplicar/validar nomenclatura de archivo para spools **REAL**.
- Generar una salida de clasificación (metadato/flag) por XML del ciclo: `REAL` o `DUMMY`.
- Definir (como resultado de esta HU) el insumo necesario para enrutar a:
  - Ruta/bucket de **pruebas**.
  - Ruta/bucket de **ciclo real**.
- Registrar evidencia de clasificación por lote (lista de control).

### Fuera de Alcance
- Definición detallada de tópicos/buckets (se asume tarea técnica paralela de arquitectura; esta HU solo deja requerimientos/criterios de entrada/salida).
- Implementación de plantillas de factura, reglas de DIAN, o renderizado PDF.
- Reglas de B2B (solo queda “preparación” como contexto; no se implementa aquí).

---

## 3. Reglas de Negocio

1. **Clasificación por ROUTE_ID**
   - Si `ROUTE_ID = FIX_INTERNAL` → el XML corresponde a factura **DUMMY**.
   - Si `ROUTE_ID = FIX` → el XML corresponde a factura **REAL** (consume numeración DIAN).
2. **Nomenclatura obligatoria para spool REAL**
   - Formato: `XML_FIXED_REAL_BILL_$<BillCycleID>_$<DATETIME_YYYYMMDDHHMMSS>.tar.gz`
   - Ejemplo: `XML_FIXED_REAL_BILL_20240341_20240418211228.tar.gz`
3. **Regla del BillCycleID para fijo**
   - Los ciclos de fijo comienzan desde el número **41**.
4. **Consistencia operativa**
   - La clasificación debe aplicarse a **todos** los XML del ciclo de facturación.
5. **Comportamiento de factura DUMMY**
   - El comportamiento de la “descripción” para factura DUMMY debe ser equivalente al comportamiento vigente de “Móvil palabra (test)” (se mantiene como referencia funcional; los detalles se validarán en pruebas).

---

## 4. Criterios de Aceptación Funcionales (BDD/Gherkin)

**Escenario: Identificación de spool DUMMY por ROUTE_ID**  
- **Dado que** el sistema recibe un XML de facturación fija desde CBS Fijo  
- **Cuando** se lee el valor de `BILL_INFO/BILL_RUN/ACCT_INFO/ACCT_PROP/ROUTE_ID`  
- **Entonces** si el valor es `FIX_INTERNAL` el XML debe clasificarse como **DUMMY**  
- **Y** el resultado de clasificación debe estar disponible para el proceso de ruteo (pruebas)  

**Escenario: Identificación de spool REAL por ROUTE_ID**  
- **Dado que** el sistema recibe un XML de facturación fija desde CBS Fijo  
- **Cuando** se lee el valor de `BILL_INFO/BILL_RUN/ACCT_INFO/ACCT_PROP/ROUTE_ID`  
- **Entonces** si el valor es `FIX` el XML debe clasificarse como **REAL**  
- **Y** el resultado de clasificación debe estar disponible para el proceso de ruteo (ciclo real)  

**Escenario: Validación de nomenclatura del spool REAL**  
- **Dado que** el sistema procesa un spool clasificado como **REAL**  
- **Cuando** se determine el `BillCycleID` y la marca de tiempo del archivo  
- **Entonces** el nombre del archivo debe cumplir el formato `XML_FIXED_REAL_BILL_$<BillCycleID>_$<YYYYMMDDHHMMSS>.tar.gz`  
- **Y** el `BillCycleID` debe ser un valor coherente para fijo (>= 41)  

**Escenario: Clasificación completa del ciclo**  
- **Dado que** el sistema recibe un conjunto de XML correspondientes a un ciclo de facturación  
- **Cuando** se procese el lote completo  
- **Entonces** el sistema debe producir una **clasificación correcta para todos los XML del ciclo**  
- **Y** debe generarse/actualizarse una **lista de control por lote** con el resumen de la clasificación  

---

## 5. Criterios de Aceptación Técnicos

1. La lectura de `ROUTE_ID` debe ser robusta ante:
   - Nodo ausente
   - Valor vacío
   - Valor diferente a `FIX` o `FIX_INTERNAL`
2. Si `ROUTE_ID` es inválido o no se puede leer:
   - El sistema debe marcar el XML como **Error de clasificación**
   - Debe registrarse el detalle para investigación (sin detener el procesamiento del lote completo, salvo que se defina lo contrario por arquitectura).
3. La validación de nombre del spool REAL debe incluir:
   - Validación del patrón (regex o equivalente)
   - Validación de longitud/estructura del timestamp `YYYYMMDDHHMMSS`
4. La salida de la clasificación debe ser consumible por el componente de ruteo (por ejemplo: metadato en memoria, archivo de control, registro en tabla, o evento).  
   *Nota:* el mecanismo exacto se definirá por el equipo técnico, pero debe existir una interfaz clara.

---

## 6. NFR / Requisitos No Funcionales

- **Trazabilidad/Auditoría:** el proceso debe permitir auditoría por lote (qué XML se clasificó como REAL/DUMMY y por qué).
- **Rendimiento:** clasificación por XML debe ejecutarse en tiempo constante y sin bloquear el procesamiento del lote.
- **Mantenibilidad:** las reglas `FIX`/`FIX_INTERNAL` deben ser parametrizables (configurable) para permitir cambios futuros.

---

## 7. Riesgos

- Si se depende únicamente de `ROUTE_ID`, cambios en CBS podrían romper la clasificación sin aviso.
- Definición incompleta de rutas/buckets puede retrasar la integración operativa del flujo (necesita coordinación con arquitectura).

---

## 8. QA / UAT

- Probar con XMLs que contengan:
  - `ROUTE_ID = FIX_INTERNAL`
  - `ROUTE_ID = FIX`
  - `ROUTE_ID` ausente
  - `ROUTE_ID` con valor inesperado
- Validar que:
  - La clasificación coincide con lo esperado.
  - Se genera la lista de control por lote.
  - La nomenclatura del spool REAL se valida correctamente (casos OK y KO).

---

## 9. Definición de Hecho (DoD)

- Implementación completa de la clasificación REAL/DUMMY.
- Evidencias de pruebas (unitarias/integración) para los escenarios definidos.
- Lista de control por lote disponible y verificable.
- Documentación mínima de la interfaz de salida de clasificación (para el componente de ruteo).
