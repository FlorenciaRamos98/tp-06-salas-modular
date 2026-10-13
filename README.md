# Trabajo práctico 06

## Proyecto de partida y cambios

El proyecto parte de una copia del Trabajo Práctico 05, correspondiente al sistema de reservas de salas de estudio.

El objetivo del TP06 es refactorizar la aplicación para separar sus responsabilidades en módulos, conservando el comportamiento funcional del sistema original. No se agregaron nuevas funcionalidades ni una base de datos.

Los principales cambios realizados fueron:

- Se separó el arranque del servidor de la configuración de Express.
- Se incorporó un módulo para leer y validar las variables de entorno.
- Se separaron las rutas, los controladores, los servicios y los middleware.
- Se conservaron las vistas EJS, el layout, los estilos CSS y los datos iniciales.
- Se mantuvieron Morgan, el identificador de solicitud y la medición de duración hasta que finaliza la respuesta.
- Se conservaron la validación de reservas y los estados HTTP correspondientes.
- Se agregaron Prettier y ESLint para comprobar el formato y la calidad estática del código.

La carpeta del TP05 original se mantuvo intacta y el proyecto del TP06 se trabajó de manera independiente.

## Instalación y ejecución

Requisitos: tener instalado Node.js y npm.

Desde la carpeta raíz del proyecto, ejecutar:

```bash
npm install
npm run format
npm run check
npm start
```

El servidor utiliza el puerto 3000 por defecto.

Direcciones principales:

- Inicio: `http://localhost:3000/`
- Estado del servicio: `http://localhost:3000/estado`
- Listado de reservas: `http://localhost:3000/reservas`
- Formulario de reserva: `http://localhost:3000/reservas/nueva`

Para detener el servidor, presionar `Ctrl + C` en la terminal donde se está ejecutando.

## Configuración del entorno

La configuración se encuentra en `src/configuracion.js`.

Las variables de entorno utilizadas son:

- `PORT`: indica el puerto del servidor. Si no se define, se utiliza el puerto 3000. Solo se aceptan números enteros entre 1 y 65535.
- `NODE_ENV`: determina el formato de registro de Morgan. En desarrollo se utiliza `dev` y en producción se utiliza `combined`.

El archivo `.env.example` contiene valores de ejemplo:

```env
PORT=3000
NODE_ENV=development
```

El archivo `.env` es opcional y no debe publicarse. El servidor funciona sin él gracias a los valores predeterminados. El archivo `.gitignore` excluye `.env` y `node_modules/` del repositorio.

Se comprobó la configuración con un puerto alternativo, utilizando el puerto 3001, y con un puerto inválido, 70000. El puerto alternativo permitió iniciar el servidor y el puerto inválido produjo un mensaje de error sin iniciar el servidor.

## Mapa de módulos y dependencias

La aplicación se organiza de la siguiente manera:

- `src/index.js`: crea los datos iniciales, obtiene la configuración, crea una única instancia del servicio, conecta las dependencias e inicia el servidor.
- `src/configuracion.js`: lee y valida las variables de entorno.
- `src/app.js`: configura Express, las vistas EJS, los middleware, las rutas y la respuesta final para las URL inexistentes.
- `src/rutas/reservas.js`: declara las rutas del área de reservas y relaciona cada una con su middleware y controlador.
- `src/controladores/reservas.js`: recibe las solicitudes, consulta el servicio y decide qué vista mostrar, qué respuesta enviar o a qué dirección redirigir.
- `src/servicios/reservas.js`: contiene las operaciones para listar, buscar, contar y crear reservas.
- `src/middleware/solicitudes.js`: identifica cada solicitud y registra el tiempo transcurrido hasta que termina la respuesta.
- `src/middleware/reservas.js`: prepara el área de reservas y valida los datos recibidos antes de crear una reserva.

El servicio no utiliza `req` ni `res` porque su responsabilidad es trabajar con los datos, sin depender de Express, de las vistas ni de los códigos HTTP. Esto permite separar la lógica de las reservas del manejo de las solicitudes.

El controlador se encarga de comunicar el resultado al cliente mediante las respuestas HTTP, las vistas o las redirecciones.

El router declara caminos relativos porque está montado en la aplicación bajo `/reservas`. Por ejemplo, la ruta `GET /` del router corresponde a `GET /reservas` en la aplicación.

La instancia del servicio es compartida por el listado, el detalle, el endpoint `/estado` y la creación de reservas. De esta manera, todos consultan y modifican la misma colección.

## Pipeline y contrato de rutas

El recorrido general de las solicitudes es el siguiente:

```text
Morgan
  ↓
Identificador de solicitud
  ↓
Medición de duración
  ↓
Configuración de vistas y layout
  ↓
Archivos estáticos
  ↓
Parser de formularios (urlencoded)
  ↓
Parser JSON
  ↓
Rutas de la aplicación
  ↓
Router de reservas (/reservas)
  ↓
Respuesta 404 para URL inexistentes
```

Morgan registra las solicitudes HTTP. El middleware propio asigna un identificador a cada solicitud y mide el tiempo transcurrido hasta el evento `finish`, registrando el método, la URL, el estado HTTP y la duración.

El contrato de rutas conservado es:

| Método | Ruta | Comportamiento esperado |
|---|---|---|
| GET | `/` | Muestra la página de inicio (200). |
| GET | `/estado` | Devuelve un JSON con el estado, la cantidad de reservas y el identificador de solicitud (200). |
| GET | `/reservas` | Muestra el listado de reservas (200). |
| GET | `/reservas/nueva` | Muestra el formulario de creación (200). |
| GET | `/reservas/:id` | Muestra el detalle de una reserva (200) o una respuesta de no encontrada (404). |
| POST | `/reservas` | Si los datos son válidos, crea la reserva y redirige (302). Si son inválidos, responde 400 sin crearla. |
| GET | `/css/estilos.css` | Entrega los estilos CSS (200; el navegador también puede responder con 304 cuando utiliza la caché). |
| GET | Cualquier URL inexistente | Devuelve la respuesta final de no encontrada (404). |

La ruta `/reservas/nueva` se declara antes de `/reservas/:id` para que el formulario no sea interpretado como un identificador de reserva.

El parser `express.urlencoded` se ejecuta antes del middleware de validación. Si los datos son inválidos, la validación responde con 400 y no continúa hacia el controlador. Si son válidos, prepara `req.reservaValidada` y permite continuar.

## Matriz antes/después

El objetivo de la refactorización fue conservar el contrato funcional del TP05 y separar las responsabilidades sin modificar el comportamiento observable de las rutas.

| Caso | Comportamiento de referencia del TP05 | Resultado comprobado en TP06 |
|---|---|---|
| Inicio y CSS | Inicio y recurso estático disponibles | Inicio 200; CSS 200 en la carga inicial y 304 en una solicitud posterior por caché |
| Estado inicial | JSON con cantidad e identificador | 200; se observaron 4 reservas al iniciar y el identificador de solicitud |
| Listado y detalle existente | Listado y detalle disponibles | Listado y detalle existentes respondieron 200 |
| Detalle inexistente | Respuesta 404 | 404, sin caída del servidor |
| Formulario | Formulario disponible | 200 |
| POST con campos vacíos | 400, sin alta | 400 |
| POST con sala no permitida | 400, sin alta | 400 |
| POST con email sin `@` | 400, sin alta | 400 |
| POST con personas igual a 0 | 400, sin alta | 400 |
| POST con personas igual a 7 | 400, sin alta | 400 |
| POST con personas fraccionarias (`2.5`) | 400, sin alta | 400 |
| POST válido | 302 y luego 200 en el listado | 302; la reserva apareció en el listado con respuesta 200 |
| URL inexistente | 404 | 404 |
| Reinicio del servidor | Regreso a los datos iniciales | Al reiniciar, la colección volvió a las 4 reservas iniciales |
| Puerto alternativo | Debe iniciar en el puerto configurado | El servidor inició correctamente en el puerto 3001 |
| Puerto inválido | No debe iniciar | Con `PORT=70000`, mostró un mensaje de configuración inválida y no inició |

Después de crear una reserva válida, el listado pasó de 4 a 5 reservas. Luego se ejecutaron las cinco pruebas de validación inválida y se comprobó que `/estado` seguía mostrando 5 reservas, por lo que esos intentos no agregaron nuevos registros.

En los registros se comprobaron respuestas 200, 302, 400 y 404 tanto en Morgan como en el registro propio del middleware de solicitudes.

## Formato y análisis estático

Se ejecutaron los siguientes comandos:

- `npm install`: instaló las dependencias del proyecto; en la última ejecución informó que no había vulnerabilidades.
- `npm run format`: aplicó el formato de Prettier a los archivos de `src`.
- `npm run check`: ejecutó `format:check` y ESLint sin errores.
- `npm start`: inició el servidor para las pruebas manuales.

Los scripts configurados en `package.json` son:

```json
{
  "start": "node src/index.js",
  "format": "prettier --write src",
  "format:check": "prettier --check src",
  "lint": "eslint src",
  "check": "npm run format:check && npm run lint"
}
```

Las comprobaciones de formato y análisis estático no reemplazan las pruebas manuales de las rutas HTTP.

## Persistencia temporal y límites

Las reservas se almacenan en memoria, dentro de la colección privada del servicio creado en `src/servicios/reservas.js`. El único arreglo mutable con las reservas se encuentra en ese servicio y se inicializa a partir de los datos definidos en `src/index.js`.

El servicio permite listar, buscar por identificador, contar y crear reservas. La colección es compartida por los controladores y por el endpoint `/estado`, por lo que los cambios realizados mediante una reserva válida se reflejan en el listado y en el estado del servicio.

Como no se utiliza una base de datos ni otro mecanismo de persistencia, las nuevas reservas se pierden cuando se detiene o reinicia el servidor. Al volver a iniciarlo, se cargan nuevamente las cuatro reservas iniciales.

El alcance del trabajo es la refactorización modular de la aplicación existente. No se incorporaron bases de datos, APIs nuevas, autenticación ni persistencia permanente.

### Explicación de las decisiones de diseño

**¿Por qué el servicio no utiliza `res`?**

El servicio se encarga únicamente de trabajar con las reservas: listarlas, buscarlas, contarlas y crearlas. No utiliza `res` porque no debe encargarse de responder las solicitudes HTTP. Esa tarea corresponde al controlador, que decide si debe mostrar una vista, enviar una respuesta o redirigir al usuario.

**¿Qué hace el controlador?**

El controlador recibe las solicitudes, obtiene los datos necesarios y utiliza el servicio para realizar las operaciones sobre las reservas. Después, según el resultado, decide qué respuesta HTTP enviar, qué vista mostrar o a qué ruta redirigir.

**¿Por qué el router declara caminos relativos?**

El router de reservas está montado en la aplicación bajo `/reservas`. Por eso declara caminos relativos, como `/` y `/:id`, sin repetir el prefijo. Por ejemplo, `GET /` dentro del router corresponde a `GET /reservas` en la aplicación.

**¿Dónde vive el único arreglo de reservas y qué ocurre al reiniciar?**

El arreglo mutable de reservas vive dentro de `src/servicios/reservas.js`, en la instancia del servicio creada desde `src/index.js`. Los datos iniciales se definen en `src/index.js` y se utilizan para inicializar la colección. El listado, el detalle, el endpoint `/estado` y la creación utilizan la misma instancia del servicio.

Como las reservas se almacenan únicamente en memoria, las nuevas reservas no se guardan de forma permanente. Al detener y volver a iniciar el servidor, los cambios se pierden y se recuperan las cuatro reservas iniciales.

### Comandos ejecutados y resultados

| Comando o prueba | Resultado observado |
|---|---|
| `npm install` | Dependencias instaladas; en la última ejecución se informó que no había vulnerabilidades. |
| `npm run format` | Prettier aplicó el formato a los archivos de `src`. |
| `npm run check` | La comprobación de formato y ESLint terminó sin errores. |
| `npm start` | El servidor inició correctamente en el puerto 3000. |
| Solicitudes GET a las rutas principales | Se comprobaron respuestas 200 y 404 según la ruta solicitada. |
| POST con una reserva válida | Respondió 302 y luego 200 al consultar el listado. |
| POST con datos inválidos | Las cinco pruebas realizadas respondieron 400. |
| Consulta de `/estado` después de los POST inválidos | La cantidad permaneció en 5 reservas. |
| Reinicio del servidor | La colección volvió a las cuatro reservas iniciales. |
| Configuración con `PORT=3001` | El servidor inició correctamente en el puerto alternativo. |
| Configuración con `PORT=70000` | Se mostró un mensaje de configuración inválida y el servidor no inició. |

No se incluyen valores ni credenciales del archivo `.env` local.