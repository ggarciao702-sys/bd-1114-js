# Base de Datos 1114 - Paso a paso

Vas a construir un solo archivo, `escuela.js`, etapa por etapa. No copies sin entender: cada bloque hace una cosa concreta. Si algo no te cierra, pregunta ANTES de seguir.

## Etapa 0 - Verificar Node

Cada uno trabaja en su propia maquina, asi que lo primero es saber si tenes Node.

Abrí la terminal y ejecuta:

```bash
node --version
```

Tiene que dar `v22.5.0` o superior.

Si da "comando no encontrado", un numero menor a `22.5.0`, o directamente no sale nada, tenes que instalarlo:

1. Entrá a https://nodejs.org
2. Descargá la version LTS (la que dice "Recommended")
3. Instalala con doble clic (todo "Siguiente")
4. Cerrá la terminal y abrila de nuevo
5. Verificá otra vez con `node --version`

Cuando ya tengas Node, crea una carpeta para el taller y ubicate adentro:

```bash
mkdir base-datos-1114
cd base-datos-1114
```

## Etapa 1 - Repaso de JSON

Crea el archivo `escuela.js` y pega esto:

```javascript
const alumnos = [
  { nombre: 'Ana', seccion: '1114', edad: 19 },
  { nombre: 'Luis', seccion: '1114', edad: 21 },
  { nombre: 'Marta', seccion: '1113', edad: 20 },
  { nombre: 'Pedro', seccion: '1114', edad: 18 },
];

const deLa1114 = alumnos.filter(a => a.seccion === '1114');
console.log('Alumnos de 1114:', deLa1114);

const mayores = alumnos.filter(a => a.edad >= 20);
console.log('Mayores de 20:', mayores);
```

Corre con:

```bash
node escuela.js
```

Esto ya lo sabes de la clase pasada: JSON es un array de objetos. `filter` devuelve una lista nueva.
![alt text](image.png)

## Etapa 2 - El problema

JSON funciona, pero tiene dos limites. Agrega esto al final de `escuela.js`:

```javascript
const resultado = alumnos
  .filter(a => a.seccion === '1114')
  .filter(a => a.edad >= 19)
  .filter(a => a.nombre.length > 3)
  .sort((a, b) => a.edad - b.edad);

console.log('Consulta con filtros:', resultado);
```
![alt text](image-1.png)

Preguntas:

1. Que pasa con este codigo si necesito 5 filtros? Y si necesito 10? 

           Si aumentan a 5 o 10 filtros, el código manual se vuelve rígido y complejo. La solución es construir la consulta SQL de forma dinámica, acumulando las condiciones en una cadena de texto y sus valores en un arreglo según lo que el usuario necesite filtrar. En proyectos grandes, la mejor práctica es utilizar un ORM (como Prisma), el cual permite manejar decenas de filtros complejos mediante simples objetos de JavaScript de forma limpia y segura.
2. Si cierro el programa, donde quedaron los datos?

       Los datos quedaron guardados de forma permanente en el disco duro dentro del archivo llamado escuela.db

Respuesta corta: JSON **no consulta** (solo filtra a mano) y **no persiste** (se pierde al cerrar). Para eso existe SQLite.

## Etapa 3 - SQLite entra

Borra el contenido de `escuela.js` y empeza de nuevo. Primero, pedimos el modulo:

```javascript
const { DatabaseSync } = require('node:sqlite');
const db = new DatabaseSync('escuela.db');
```

Esa linea abre (o crea) un archivo llamado `escuela.db`. Ahi van a vivir los datos para siempre.

Ahora creamos la tabla y cargamos datos:

```javascript
db.exec(`
  CREATE TABLE IF NOT EXISTS alumnos (
    id INTEGER PRIMARY KEY,
    nombre TEXT NOT NULL,
    seccion TEXT NOT NULL,
    edad INTEGER
  )
`);

const alumnos = [
  { nombre: 'Ana', seccion: '1114', edad: 19 },
  { nombre: 'Luis', seccion: '1114', edad: 21 },
  { nombre: 'Marta', seccion: '1113', edad: 20 },
  { nombre: 'Pedro', seccion: '1114', edad: 18 },
];

const insert = db.prepare('INSERT INTO alumnos (nombre, seccion, edad) VALUES (?, ?, ?)');
for (const a of alumnos) {
  insert.run(a.nombre, a.seccion, a.edad);
}

console.log('Datos cargados en escuela.db');
```
![alt text](image-2.png)

Fijate dos cosas:

- `?` es un comodin. `run` lo reemplaza por el valor que le pases, en orden. Asi evitamos meter datos a mano en el SQL.
- `id INTEGER PRIMARY KEY` le da a cada alumno un numero unico automatico.

Corre con `node escuela.js`. Si lo corres dos veces, se duplican los datos. Eso es normal: estamos insertando, no reemplazando. Mas adelante lo arreglamos.

## Etapa 4 - Consultas con SELECT

Agrega al final de `escuela.js`:

```javascript
const selSeccion = db.prepare('SELECT * FROM alumnos WHERE seccion = ?');
console.log('Alumnos de 1114:', selSeccion.all('1114'));

const selMayores = db.prepare('SELECT nombre, edad FROM alumnos WHERE edad >= ? ORDER BY edad');
console.log('Mayores de 20:', selMayores.all(20));

const selPrimero = db.prepare('SELECT * FROM alumnos ORDER BY edad DESC LIMIT 1');
console.log('El mas grande:', selPrimero.get());

const selCuenta = db.prepare('SELECT COUNT(*) AS total FROM alumnos');
console.log('Total de alumnos:', selCuenta.get());
```
![alt text](image-3.png)

Compara esto con el `filter` de la Etapa 1. Es lo mismo, pero:

- El SQL se lee como una pregunta en ingles ("trae los alumnos donde la seccion sea 1114").
- Los datos viven en disco, no en memoria.

Tres metodos que tenes que distinguir:

| Metodo | Devuelve |
|---|---|
| `.run()` | Resultado de insertar/actualizar/borrar (cuantos cambios) |
| `.get()` | La primera fila que matchea (un solo objeto) |
| `.all()` | Todas las filas que matchean (un array) |

## Etapa 5 - Modificar y borrar

Agrega:

```javascript
const actualizar = db.prepare('UPDATE alumnos SET edad = ? WHERE nombre = ?');
const cambio = actualizar.run(22, 'Ana');
console.log('Filas actualizadas:', cambio.changes);

const borrar = db.prepare('DELETE FROM alumnos WHERE nombre = ?');
const borrado = borrar.run('Marta');
console.log('Filas borradas:', borrado.changes);
```

![alt text](image-4.png)

Dato importante: `UPDATE` y `DELETE` sin `WHERE` afectan a TODA la tabla. El `WHERE` es tu freno de mano.

## Etapa 6 - El puente JSON <-> SQL

Este es el momento. Cierra el circulo: consulta con SQL y devuelve el resultado como JSON:

```javascript
const rows = selSeccion.all('1114');
const json = JSON.stringify(rows, null, 2);
console.log('Como JSON:');
console.log(json);
```

![alt text](image-5.png)

Ahora los datos salen de la base en disco y vuelven al mundo JavaScript como JSON, listos para mandarlos por internet, guardarlos en un archivo o mostrarlos en una pantalla.

## Etapa 7 - Desafio

Agrega dos tablas mas: `cursos` y `inscripciones`. La idea: un alumno puede inscribirse a varios cursos.

```javascript
db.exec(`
  CREATE TABLE IF NOT EXISTS cursos (
    id INTEGER PRIMARY KEY,
    nombre TEXT NOT NULL
  )
`);

db.exec(`
  CREATE TABLE IF NOT EXISTS inscripciones (
    alumno_id INTEGER,
    curso_id INTEGER,
    FOREIGN KEY (alumno_id) REFERENCES alumnos(id),
    FOREIGN KEY (curso_id) REFERENCES cursos(id)
  )
`);
```

Despues cargá cursos e inscripciones, y resolvé estas consultas:

1. Que alumnos se inscribieron a "Base de Datos"?
2. Cuantos cursos tiene cada alumno?

Pista: para la primera necesitas un `JOIN` entre las tres tablas. Para la segunda, `GROUP BY`.

## Entregable

Cada alumno entrega:

1. `escuela.js` completo y funcionando.
2. El archivo `escuela.db` generado.
3. Respuesta escrita a las preguntas de las Etapas 2 y 7.
4. Una frase explicando, con sus palabras, para que usa JSON y para que usa SQLite.
