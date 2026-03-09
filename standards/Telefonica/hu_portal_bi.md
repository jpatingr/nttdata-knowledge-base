# Historia de Usuario: Visualización de Indicadores de Negocio (BI) - Portal Telefónica

## 1. Definición de la Historia
**Como:** Analista de Operaciones de Telefónica  
**Quiero:** Visualizar un tablero con los indicadores de rendimiento de red en tiempo real  
**Para:** Tomar decisiones proactivas sobre el mantenimiento y evitar caídas de servicio.

---

## 2. Contexto de Negocio
Esta HU pertenece al ecosistema del Portal BI. Los datos provienen de la API de monitoreo de red y deben presentarse de forma gráfica y tabular.

---

## 3. Criterios de Aceptación (Estándar NTT DATA)

### Funcionales
- [ ] **CA1:** El sistema debe cargar los datos del último turno (8 horas) por defecto.
- [ ] **CA2:** Se debe incluir un gráfico de líneas para la latencia y un gráfico de barras para el volumen de tráfico.
- [ ] **CA3:** Los indicadores deben actualizarse automáticamente cada 5 minutos sin necesidad de recargar la página.
- [ ] **CA4:** Debe permitir la descarga de los datos en formato CSV.

### Técnicos
- [ ] **Arquitectura:** La conexión debe realizarse mediante el servicio de integración de .NET Core existente.
- [ ] **Rendimiento:** El tiempo de respuesta de la consulta no debe superar los 2 segundos.
- [ ] **Seguridad:** Solo los usuarios con el rol `BI_ADMIN` pueden ver los detalles de infraestructura crítica.

---

## 4. Definición de Hecho (DoD)
- Código revisado por un par (Peer Review).
- Pruebas unitarias con cobertura mínima del 80%.
- Documentación técnica actualizada en el Wiki del proyecto.
- Cumplimiento de estándares de accesibilidad (WCAG 2.1).

---

## 5. Notas Adicionales
*Referencia para el desarrollador:* Utilizar la librería de componentes compartidos de NTT DATA para los gráficos para mantener la identidad visual de Telefónica.
