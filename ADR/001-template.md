# ADR-001: Refactorización del módulo de autenticación para eliminar vulnerabilidades críticas y aplicar principios SOLID y Clean Code

## Contexto

El sistema actual implementa un módulo básico de autenticación que permite registrar usuarios y realizar login mediante acceso directo a una base de datos PostgreSQL usando JDBC. La arquitectura es sencilla: `AuthController` recibe las peticiones HTTP, delega en `AuthService`, y este utiliza `UserRepository` para ejecutar consultas SQL construidas manualmente.

Durante la auditoría se identificaron vulnerabilidades críticas y múltiples violaciones a principios de Clean Code y SOLID. Entre los hallazgos más graves se encuentran: vulnerabilidad a SQL Injection por concatenación directa de parámetros en consultas, uso de `Statement` en lugar de `PreparedStatement`, credenciales de base de datos hardcodeadas en el código fuente, almacenamiento de contraseñas utilizando MD5 (algoritmo inseguro), exposición del hash de la contraseña en la respuesta del login y ausencia de cierre de conexiones JDBC.

Además, se encontraron problemas de diseño como clases con demasiadas responsabilidades (violación del principio de Responsabilidad Única), atributos públicos en el modelo `User`, uso de `Map<String,Object>` como respuesta en lugar de DTOs tipados y nombres poco descriptivos que afectan la legibilidad y mantenibilidad del código.

Estos problemas son urgentes porque comprometen directamente la seguridad de los usuarios y del negocio. En un entorno productivo, las vulnerabilidades detectadas podrían permitir acceso no autorizado, filtración de información sensible y compromiso total de la base de datos. También afectan al equipo de desarrollo, ya que el código actual es difícil de mantener, probar y escalar.

---

## Decisión

Se decide realizar una refactorización estructurada del módulo de autenticación aplicando principios de seguridad por diseño, Clean Code y SOLID.

### 1. Eliminación de SQL Injection

Se reemplazará el uso de `Statement` por `PreparedStatement` en `UserRepository`, eliminando la concatenación de strings en las consultas SQL.  
Esta decisión aplica el principio de separación entre datos y estructura de consulta, mitigando completamente el riesgo de inyección SQL.

Adicionalmente, se externalizarán las credenciales de base de datos a `application.properties` o variables de entorno, eliminando información sensible del código fuente.

---

### 2. Sustitución de MD5 por BCrypt

Se reemplazará el método `md5()` por un algoritmo de hashing seguro como BCrypt, el cual incluye salt automático y está diseñado específicamente para almacenamiento de contraseñas.

Se eliminará completamente el retorno del hash en las respuestas del login, aplicando el principio de mínima exposición de datos sensibles.

---

### 3. Aplicación del principio SRP y mejora de arquitectura

Se reorganizará el módulo en capas claramente definidas:

- `AuthController` → Manejo exclusivo de peticiones HTTP y validación básica de entrada.
- `AuthService` → Lógica de negocio relacionada con autenticación.
- `UserRepository` → Acceso a datos exclusivamente.
- `PasswordEncoder` (nuevo componente) → Encapsular lógica de hashing.

También se introducirán DTOs tipados (`LoginResponse`, `RegisterResponse`) en lugar de `Map<String,Object>`, mejorando legibilidad, seguridad de tipos y mantenibilidad.

El modelo `User` será refactorizado para tener atributos privados con getters y setters, respetando el principio de encapsulamiento.

---

### 4. Mejora en gestión de recursos y logging

Se implementará manejo adecuado de recursos utilizando `try-with-resources` para cerrar conexiones, statements y result sets automáticamente.

Se reemplazará `System.out.println` por un framework de logging estándar (por ejemplo, SLF4J), evitando exposición innecesaria de información sensible en consola.

---

## Consecuencias

### Consecuencias positivas

- Eliminación de la vulnerabilidad SQL Injection.
- Protección adecuada de contraseñas mediante hashing seguro.
- Eliminación de exposición de datos sensibles en respuestas HTTP.
- Mejor separación de responsabilidades y cumplimiento de principios SOLID.
- Código más legible, mantenible y testeable.
- Reducción significativa del riesgo de incidentes de seguridad en producción.

---

### Consecuencias negativas o riesgos

- Incremento en el tiempo de desarrollo debido a la refactorización.
- Posibles regresiones si no se implementan pruebas adecuadas.
- Necesidad de migrar contraseñas existentes al nuevo esquema de hashing.
- Mayor complejidad inicial comparada con la implementación original simplificada.

---

## Alternativas consideradas

### 1. Reescribir completamente el módulo desde cero

Se consideró reescribir el módulo de autenticación desde cero para resolver todos los problemas estructurales. Sin embargo, fue descartado debido al alto costo en tiempo, mayor riesgo de introducir nuevos errores y la posibilidad de perder funcionalidades existentes. Se prefirió una refactorización incremental y controlada.

---

### 2. Corregir únicamente las vulnerabilidades críticas sin mejorar el diseño

Se evaluó limitar el cambio a reemplazar `Statement` y MD5 sin modificar la arquitectura ni mejorar nombres o encapsulamiento.  
Esta alternativa fue descartada porque dejaría intactas violaciones importantes a SRP, encapsulamiento y diseño general, manteniendo una deuda técnica significativa que afectaría la mantenibilidad futura del sistema.