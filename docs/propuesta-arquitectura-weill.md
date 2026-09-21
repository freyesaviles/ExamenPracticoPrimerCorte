# Propuesta de arquitectura de servicios web

## Sistema del Test de Inteligencia de Weill

**Asignatura:** Servicios Web  
**Actividad:** Primer Corte Evaluativo — Taller #1  
**Integrantes:** Fernando Reyes, Max Martinez, Jorddy Siezar, Emilio Carranza y Victor Cabrera.

## 1. Proyecto retomado

El proyecto seleccionado es el **Sistema del Test de Inteligencia de Weill**, desarrollado durante el semestre anterior. Su propósito es apoyar el registro y la consulta de los resultados obtenidos por los participantes en la aplicación del test. Para cada resultado se manejan datos como el nombre y correo del participante, edad, fecha de aplicación, puntuación, nivel obtenido y estado de finalización.

La propuesta transforma las funciones del proyecto en una API REST desarrollada con Spring Boot. Como referencia se utilizó el módulo Weill de `Taller2_ServiciosWeb`, que ya define operaciones CRUD y validaciones, pero conserva la información temporalmente en memoria. La nueva arquitectura propone persistencia permanente en PostgreSQL.

## 2. Problema que resuelve

El sistema permite centralizar los resultados del Test de Weill y evita depender de registros dispersos o almacenamiento temporal. Un evaluador podrá registrar, consultar, corregir y eliminar evaluaciones desde una aplicación web. Los resultados permanecerán disponibles aunque el servidor sea reiniciado.

## 3. Cliente y comunicación

El cliente será una **aplicación web** utilizada principalmente por un psicólogo o evaluador. Esta aplicación consumirá la API mediante solicitudes **HTTPS** y enviará o recibirá información en formato **JSON**.

El cliente nunca se conectará directamente con PostgreSQL. Toda operación deberá atravesar la API, lo que permite aplicar validaciones, reglas de negocio y manejo uniforme de errores.

## 4. Arquitectura propuesta

Se propone una arquitectura en capas porque separa responsabilidades y facilita el mantenimiento, las pruebas y futuros cambios tecnológicos.

### 4.1 Cliente

La aplicación web presenta formularios y vistas para registrar, listar, consultar y actualizar los resultados. Convierte las acciones del usuario en solicitudes HTTPS dirigidas a la API REST y presenta las respuestas JSON recibidas.

### 4.2 Capa de presentación o controladores

`ResultadoWeillController` expone los endpoints REST. Recibe solicitudes HTTP, obtiene los parámetros y cuerpos JSON, activa las validaciones de entrada y devuelve códigos HTTP adecuados. Los DTO delimitan la información que entra y sale de la API sin exponer directamente las entidades de persistencia.

### 4.3 Capa de aplicación o servicios

`ResultadoWeillService` coordina los casos de uso del sistema. Decide qué reglas deben ejecutarse y solicita al repositorio las operaciones de persistencia. Esta capa no contiene detalles de HTTP ni de la conexión a PostgreSQL.

### 4.4 Capa de lógica de negocio

Contiene la entidad `ResultadoWeill` y las reglas propias del test. Valida la coherencia de la puntuación, el estado de finalización y el nivel (`BAJO`, `PROMEDIO` o `ALTO`). La clasificación exacta se aplicará de acuerdo con la tabla o baremo empleado por el proyecto original; no debe quedar escrita en el controlador.

### 4.5 Capa de acceso a datos

`ResultadoWeillRepository` será una interfaz de Spring Data JPA. Proporcionará operaciones para guardar, buscar, listar, actualizar y eliminar resultados sin incluir consultas SQL dentro del controlador o del servicio.

### 4.6 Capa de infraestructura

Incluye Spring Boot, Hibernate/JPA, el controlador JDBC de PostgreSQL, la configuración de la conexión, CORS, validaciones y el manejador global de excepciones. Esta capa contiene los detalles técnicos necesarios para que las demás capas funcionen.

### 4.7 Base de datos

PostgreSQL almacenará permanentemente las evaluaciones en una tabla `resultados_weill`. Como mínimo contendrá: identificador, nombre del participante, correo, edad, fecha de aplicación, puntuación, nivel y estado de finalización.

## 5. Flujo de comunicación

### Solicitud

1. El evaluador realiza una acción en la aplicación web.
2. El cliente envía una solicitud HTTPS con datos JSON.
3. `ResultadoWeillController` recibe y valida la estructura de la solicitud.
4. `ResultadoWeillService` coordina el caso de uso.
5. La capa de negocio aplica las reglas del Test de Weill.
6. `ResultadoWeillRepository` utiliza JPA/Hibernate y JDBC para acceder a PostgreSQL.

### Respuesta

1. PostgreSQL devuelve el resultado de la operación a la capa de acceso a datos.
2. El repositorio devuelve la entidad a la capa de servicios.
3. El servicio prepara el resultado del caso de uso.
4. El controlador construye una respuesta JSON con el código HTTP correspondiente.
5. La aplicación web interpreta la respuesta y la presenta al evaluador.

Flujo resumido:

`Cliente web → Controller → Service → Reglas de negocio → Repository → PostgreSQL`

`PostgreSQL → Repository → Service → Controller → Respuesta JSON → Cliente web`

## 6. Servicios REST propuestos

| Funcionalidad | Método HTTP | Endpoint | Descripción | Respuesta esperada |
|---|---|---|---|---|
| Registrar resultado | `POST` | `/api/resultados-weill` | Registra una nueva evaluación después de validar sus datos y reglas de negocio. | `201 Created` |
| Listar resultados | `GET` | `/api/resultados-weill` | Devuelve todos los resultados registrados. | `200 OK` |
| Consultar resultado | `GET` | `/api/resultados-weill/{id}` | Busca una evaluación mediante su identificador. | `200 OK` o `404 Not Found` |
| Actualizar resultado | `PUT` | `/api/resultados-weill/{id}` | Reemplaza los datos editables de una evaluación existente. | `200 OK`, `400 Bad Request` o `404 Not Found` |
| Eliminar resultado | `DELETE` | `/api/resultados-weill/{id}` | Elimina una evaluación identificada por su ID. | `204 No Content` o `404 Not Found` |

Los cinco endpoints corresponden a funcionalidades reales presentes en el módulo Weill de referencia y superan el mínimo de tres servicios solicitado. En una implementación posterior también podrían agregarse filtros por participante, fecha o nivel sin alterar la separación de capas.

### Ejemplo de solicitud JSON

```json
{
  "nombreParticipante": "Ana López",
  "correo": "ana@example.com",
  "edad": 24,
  "fechaAplicacion": "2026-09-10",
  "puntuacion": 82,
  "nivel": "ALTO",
  "finalizado": true
}
```

## 7. Códigos HTTP y errores

- `200 OK`: consulta o actualización exitosa.
- `201 Created`: resultado registrado correctamente.
- `204 No Content`: eliminación exitosa.
- `400 Bad Request`: datos incompletos, inválidos o inconsistentes.
- `404 Not Found`: no existe un resultado con el identificador solicitado.
- `500 Internal Server Error`: error inesperado del servidor.

Los errores también se devolverán como JSON, con un mensaje claro y, cuando corresponda, el detalle de los campos inválidos.

## 8. Justificación de las decisiones

- **Spring Boot** simplifica la creación de controladores REST, validaciones e integración con la base de datos.
- **Arquitectura en capas** evita mezclar la comunicación HTTP, las reglas del test y el acceso a datos.
- **PostgreSQL** ofrece persistencia relacional confiable y es una tecnología conocida por el equipo.
- **Spring Data JPA** reduce código repetitivo y mantiene aislados los detalles de las consultas.
- **HTTPS y JSON** permiten una comunicación segura y ampliamente compatible con clientes web.
- **DTOs** controlan la información expuesta y permiten validar solicitudes sin acoplar el contrato REST a la tabla de la base de datos.

## 9. Alcance

Esta actividad presenta el diseño de la solución. No exige implementar en este momento todos los endpoints ni la base de datos. El diagrama sirve como guía para desarrollar posteriormente la API REST de Weill con Spring Boot y PostgreSQL.
