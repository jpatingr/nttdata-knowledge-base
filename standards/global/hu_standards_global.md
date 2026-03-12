# 🚀 Estándar Maestro: Historias de Usuario "AI-Ready"

Este documento define el estándar de oro para la creación de Historias de Usuario (US) que sean perfectamente comprensibles para humanos y altamente eficientes para su procesamiento mediante Inteligencia Artificial.

---

## 1. El Marco de Calidad: INVEST
Toda historia de usuario debe cumplir con el acrónimo **INVEST** para garantizar que sea procesable:

* **I**ndependiente: Evita dependencias directas para facilitar el desarrollo en paralelo.
* **N**egociable: El "cómo" se puede discutir; el "qué" debe ser claro.
* **V**aliosa: Debe entregar un beneficio real al usuario final.
* **E**stimable: La complejidad debe ser clara (Puntos de Historia).
* **S**mall (Pequeña): Si es muy grande, es un Épica. Debe caber en un Sprint.
* **T**estable: Si no se puede probar, no está terminada.

---

## 2. Estructura Semántica (Formato Prompt-Friendly)

### 🧩 Título Descriptivo
`[ID] - [Acción breve] en [Módulo/Funcionalidad]`

### 👤 Definición del Actor
**Como** `[Rol de usuario específico]`,
**Quiero** `[Acción clara y concisa en el sistema]`,
**Para** `[Beneficio o valor de negocio cuantificable]`.

> **Nota para IA:** Evita roles genéricos como "Usuario". Usa "Administrador de Base de Datos", "Cliente Premium" o "Soporte Técnico Nivel 1".

---

## 3. Criterios de Aceptación (Formato Gherkin / BDD)
Este es el punto más crítico para la IA. Usamos la sintaxis **Gherkin** para eliminar la ambigüedad y permitir la generación automática de pruebas unitarias.

**Escenario: [Nombre del caso de uso]**
* **Dado que** `[Contexto inicial / Precondición]`,
* **Cuando** `[Acción ejecutada por el usuario]`,
* **Entonces** `[Resultado esperado / Reacción del sistema]`.

---

## 4. Especificaciones Técnicas y Restricciones
Para que la IA no improvise, debemos delimitar el "tablero de juego":

* **Stack Tecnológico:** (Ej: React + TypeScript, .NET 8, Spring Boot).
* **Seguridad:** Roles requeridos, encriptación, validaciones.
* **Performance:** Tiempos de respuesta (ej: < 200ms).
* **Diseño/UI:** Referencias a componentes de diseño o tokens.

---

## 5. Ejemplo de una Historia de Usuario Perfecta

### US-044: Recuperación de Contraseña mediante Token JWT

**Como** Usuario Registrado,
**Quiero** solicitar el restablecimiento de mi contraseña mediante mi correo electrónico,
**Para** recuperar el acceso a mi cuenta de forma segura en caso de olvido.

#### ✅ Criterios de Aceptación:
**Escenario: Solicitud exitosa con correo existente**
- **Dado que** el usuario está en la pantalla de "Olvide mi contraseña",
- **Cuando** ingresa un correo electrónico registrado y hace clic en "Enviar",
- **Entonces** el sistema debe enviar un email con un link único y el sistema debe mostrar un mensaje de confirmación "Correo enviado".

**Escenario: Validación de seguridad en correo inexistente**
- **Dado que** el usuario ingresa un correo que NO existe en la BD,
- **Cuando** hace clic en "Enviar",
- **Entonces** el sistema debe mostrar el MISMO mensaje de confirmación (para evitar enumeración de usuarios).

#### 🛠️ Detalles Técnicos para IA:
- **Backend:** Endpoint `POST /auth/recovery`.
- **Expiración:** El token debe expirar en 15 minutos.
- **Log:** Registrar el intento en la tabla de auditoría con la IP del solicitante.
- **Formato:** El email debe seguir la plantilla `recovery-template.html`.

---

## 6. Tips para Alineación con IA (Prompt Engineering)
1. **Sin Pronombres:** No digas "él quiere", di "el Administrador quiere".
2. **Tablas de Datos:** Si hay lógica compleja, usa tablas MD dentro de la historia.
3. **Estado Final:** Define siempre cómo debe quedar la base de datos tras la acción.
4. **Validaciones:** Lista explícitamente qué errores debe manejar (400, 401, 404, 500).
