# Lenguaje SQL ConsultaDatos. DML. RA3
Judith Viñas Tribaldo

---

## Escenario Técnico: CineGest 2.0

Esquema relacional para gestionar la información de una cadena de cines:

- **CINES** (id_cine, nombre, ciudad)
- **PELICULAS** (id_pelicula, titulo, director, genero)
- **PROYECCIONES** (id_proyeccion, id_cine, id_pelicula, fecha, hora, recaudacion) — *id_cine e id_pelicula son claves ajenas*

---

## Creación del esquema e inserción de datos

```sql
CREATE TABLE CINES (
    id_cine  NUMBER PRIMARY KEY,
    nombre   VARCHAR2(100),
    ciudad   VARCHAR2(100)
);

CREATE TABLE PELICULAS (
    id_pelicula  NUMBER PRIMARY KEY,
    titulo       VARCHAR2(200),
    director     VARCHAR2(100),
    genero       VARCHAR2(50)
);

CREATE TABLE PROYECCIONES (
    id_proyeccion  NUMBER PRIMARY KEY,
    id_cine        NUMBER REFERENCES CINES(id_cine),
    id_pelicula    NUMBER REFERENCES PELICULAS(id_pelicula),
    fecha          DATE,
    hora           VARCHAR2(5),
    recaudacion    NUMBER
);
```

```sql
INSERT INTO CINES VALUES (1, 'Cine Sur', 'Cordoba');
INSERT INTO CINES VALUES (2, 'Cine Axion', 'Sevilla');
INSERT INTO CINES VALUES (3, 'Cine Plaza', 'Malaga');

INSERT INTO PELICULAS VALUES (1, 'Interestelar', 'Christopher Nolan', 'Ciencia Ficcion');
INSERT INTO PELICULAS VALUES (2, 'Dune', 'Denis Villeneuve', 'Ciencia Ficcion');
INSERT INTO PELICULAS VALUES (3, 'La La Land', 'Damien Chazelle', 'Musical');

INSERT INTO PROYECCIONES VALUES (1, 1, 1, DATE '2023-01-10', '20:00', 3000);
INSERT INTO PROYECCIONES VALUES (2, 1, 2, DATE '2023-02-15', '22:00', 4500);
INSERT INTO PROYECCIONES VALUES (3, 2, 3, DATE '2023-03-01', '19:00', 2000);
```

---

## EJERCICIO 1: Consultas Básicas y Composiciones

### 1.1 Películas de Ciencia Ficción ordenadas por título

```sql
SELECT titulo,
       genero
FROM   PELICULAS
WHERE  genero = 'Ciencia Ficcion'
ORDER  BY titulo;
```

---

### 1.2 Nombre del cine y título de la película para todas las proyecciones (INNER JOIN)

```sql
SELECT c.nombre AS nombre_cine,
       p.titulo AS titulo_pelicula
FROM   PROYECCIONES pr
       INNER JOIN CINES     c ON pr.id_cine     = c.id_cine
       INNER JOIN PELICULAS p ON pr.id_pelicula = p.id_pelicula;
```

---

### 1.3 Cines que no han proyectado películas todavía (LEFT OUTER JOIN)

```sql
SELECT c.nombre AS nombre_cine,
       pr.id_pelicula
FROM   CINES c
       LEFT JOIN PROYECCIONES pr ON c.id_cine = pr.id_cine
ORDER  BY c.nombre;
```

> El cine **Cine Plaza** aparece con `null` porque no tiene ninguna proyección registrada.

---

## EJERCICIO 2: Resumen y Agrupación

### 2.1 Recaudación total acumulada por cada cine

```sql
SELECT c.nombre            AS nombre_cine,
       SUM(pr.recaudacion) AS recaudacion_total
FROM   CINES c
       INNER JOIN PROYECCIONES pr ON c.id_cine = pr.id_cine
GROUP  BY c.nombre;
```

---

### 2.2 Solo cines con recaudación total > 5000€ (usar HAVING)

```sql
SELECT c.nombre            AS nombre_cine,
       SUM(pr.recaudacion) AS recaudacion_total
FROM   CINES c
       INNER JOIN PROYECCIONES pr ON c.id_cine = pr.id_cine
GROUP  BY c.nombre
HAVING SUM(pr.recaudacion) > 5000;
```

#### Explicación HAVING vs WHERE

| Cláusula | Cuándo filtra       | Puede usar funciones de grupo |
|----------|---------------------|-------------------------------|
| `WHERE`  | Antes de GROUP BY   | No                            |
| `HAVING` | Después de GROUP BY | Sí (SUM, AVG, COUNT...)       |

> Se usa `HAVING` porque la condición se aplica sobre el resultado de `SUM(recaudacion)`, que es una función de agrupación. `WHERE` no puede usarse con agregados.

---

## EJERCICIO 3: Subconsultas y Herramientas

### 3.1 Películas con recaudación en una sesión > recaudación media de todas las sesiones

```sql
SELECT p.titulo
FROM   PELICULAS p
       INNER JOIN PROYECCIONES pr ON p.id_pelicula = pr.id_pelicula
WHERE  pr.recaudacion >
       (SELECT AVG(recaudacion)
        FROM   PROYECCIONES);
```

---

### 3.2 DML usado por Workbench y Consola, y diferencias

**Pregunta:** ¿Qué comando DML es el motor de ambas herramientas para mostrar resultados? ¿Existe alguna diferencia según la herramienta?

**Respuesta:**

El comando que se ejecuta en ambos casos es `SELECT`, utilizado para consultar datos en la base de datos.

Los datos obtenidos son los mismos, porque ambas herramientas ejecutan la misma consulta sobre la misma base de datos. Lo que puede cambiar es:

- La **forma en que se muestran** los datos (tabla, colores, paginación, exportación, etc.).
- Las **opciones adicionales** de cada herramienta (gráficos, plan de ejecución, estadísticas, etc.).

> En resumen: los datos son los mismos, pero la forma de mostrarlos puede variar.

---

## EJERCICIO 4: Optimización de Consultas

### 4.1 Crear un índice para acelerar búsquedas por fecha

```sql
CREATE INDEX idx_proyecciones_fecha ON PROYECCIONES (fecha);
```

**Justificación:**

- **Sin índice:** el motor lee toda la tabla para encontrar las filas por fecha (full table scan).
- **Con índice:** el motor accede directamente a las entradas que coinciden con la fecha, reduciendo lecturas de disco y tiempo de respuesta.

---

### 4.2 Consultas para películas proyectadas en 2023

#### Versión 1 — Ineficiente

```sql
SELECT DISTINCT p.titulo
FROM   PELICULAS p
       INNER JOIN PROYECCIONES pr ON p.id_pelicula = pr.id_pelicula
WHERE  TO_CHAR(pr.fecha, 'YYYY') = '2023';
```

**Por qué es ineficiente:**
- `TO_CHAR(pr.fecha, 'YYYY')` aplica una función sobre la columna indexable, lo que impide usar el índice eficientemente.
- Obliga al motor a evaluar la función para cada fila de la tabla.

---

#### Versión 2 — Optimizada

```sql
SELECT DISTINCT p.titulo
FROM   PELICULAS p
       INNER JOIN PROYECCIONES pr ON p.id_pelicula = pr.id_pelicula
WHERE  pr.fecha >= DATE '2023-01-01'
AND    pr.fecha <  DATE '2024-01-01';
```

**Por qué es eficiente:**
- La condición es un rango directo sobre `fecha` sin funciones, lo que permite usar el índice `idx_proyecciones_fecha`.
- Se reduce el número de filas leídas y comparadas.

#### Comparación de eficiencia

| Aspecto        | Versión 1 (ineficiente)                     | Versión 2 (optimizada)                       |
|----------------|---------------------------------------------|----------------------------------------------|
| Tráfico de red | Revisa más filas, proceso más lento         | Filtra desde el inicio, devuelve antes       |
| Uso de memoria | Mayor, aplica funciones a cada fila         | Menor, trabaja solo con las filas necesarias |
| Uso de índices | No lo aprovecha (función sobre la columna)  | Aprovecha índices B-tree sobre fecha         |
