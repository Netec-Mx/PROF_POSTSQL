# Replicación lógica y particionamiento de tablas

## Objetivo de la práctica:
Al finalizar la práctica, serás capaz de:
- Configurar y mantener un sistema de replicación lógica en PostgreSQL.
- Diseñar e implementar estrategias efectivas de particionamiento de tablas.
## Objetivo Visual 

![diagrama1](../images/cap4/img1.png)

## Duración aproximada:
- 30 minutos.

## Tabla de ayuda:

## Instrucciones 
<!-- Proporciona pasos detallados sobre cómo configurar y administrar sistemas, implementar soluciones de software, realizar pruebas de seguridad, o cualquier otro escenario práctico relevante para el campo de la tecnología de la información -->
### Tarea 1. Realizar la replicación lógica de una tabla entre dos instancias de postgresql.

Paso 1. Crear el usuario replicator con permisos de replicacion en la instancia principal.
```shell
CREATE USER replicator WITH replication password 'curso2024';
```
Paso 2. Habilitar el la comunicación de las instancias a través de la red en el archivo pg_hba.conf
```shell
vi /etc/postgresql/14/main/pg_hba.conf
host    all             all             [ip_servidor]/32            scram-sha-256
```

Paso 3. Habilitar la replicación lógica en la instancia principal (puerto 5432) y reiniciar el servicio de la instancia principal.
```shell
vi /etc/postgresql/14/main/postgresql.conf
wal_level = logical;
```
```shell
service postgresql@14-main restart
```

Paso 4. Crear la tabla que se va a publicar en la instancia principal y dar todos los permisos al usuario replicador sobre la misma.
```shell
CREATE TABLE datos_replicados (id serial PRIMARY KEY, dato text);
GRANT ALL ON datos_replicados to replicator;
```

Paso 5. Crear la publicación.
```shell
CREATE PUBLICATION mi_publicacion FOR TABLE datos_replicados;
```


Paso 6. Crear una nueva instancia de Postgresql. Entrar como usuario postgres
```shell
pg_createcluster 14 replica
service postgresql@14-replica start
```
Revisar que las dos instancias estén activas:
```shell
postgres@curso:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                 Log file
14  main    5432 online postgres /var/lib/postgresql/14/main    /var/log/postgresql/postgresql-14-main.log
14  replica 5433 online postgres /var/lib/postgresql/14/replica /var/log/postgresql/postgresql-14-replica.log
```

Paso 7. Crear la base de datos curso_replica en la instancia réplica.
```shell
psql -p 5433
create database curso_replica;
```

Paso 8. Crear la tabla datos_replicados en la instancia replica.
```shell
CREATE TABLE datos_replicados (id serial PRIMARY KEY, dato text);
```

Paso 9. Crear la suscripción en la instancia réplica:
```shell
 CREATE SUBSCRIPTION mi_suscripcion
CONNECTION 'host=[ip_servidor] port=5432 dbname=curso user=replicator password=curso2024'
PUBLICATION mi_publicacion;
```

Paso 10. Insertar datos en la tabla datos_replicados en la instancia principal y revisar sus cambios en la instancia réplica.
```shell
-- En el publicador puerto 5432
INSERT INTO datos_replicados (dato) VALUES ('Dato de prueba');

-- En el suscriptor puerto 5433
SELECT * FROM datos_replicados;
```

### Resultado esperado
Los datos de la tabla deben ser los mismos en ambas instancias.

### Tarea 1. Realizar el particionamiento de tablas en postgresql.

1. Crear una tabla particionada.
```shell
CREATE TABLE ventas (
    id serial,
    fecha_venta date,
    monto decimal(10,2)
) PARTITION BY RANGE (fecha_venta);
```

2. Crear particiones para diferentes rangos de fechas:

```shell
CREATE TABLE ventas_2023q1 PARTITION OF ventas
    FOR VALUES FROM ('2023-01-01') TO ('2023-04-01');

CREATE TABLE ventas_2023q2 PARTITION OF ventas
    FOR VALUES FROM ('2023-04-01') TO ('2023-07-01');
```
3. Insertar datos de prueba.
```shell
INSERT INTO ventas (fecha_venta, monto)
SELECT 
    '2023-01-01'::date + (random() * 180)::integer,
    random() * 1000
FROM generate_series(1, 100000);
```
4. Analizar la distribución de los datos:

```shell
SELECT tableoid::regclass, count(*) 
FROM ventas 
GROUP BY tableoid;
```

5. Crear un índice en la tabla particionada:
```shell
CREATE INDEX ON ventas (fecha_venta);
```

6. Realizar consultas de prueba y analizar el plan de ejecución.
```shell
EXPLAIN ANALYZE 
SELECT * FROM ventas 
WHERE fecha_venta BETWEEN '2023-02-01' AND '2023-03-31';
```

7. Agregar una nueva partición y mover datos
```shell
CREATE TABLE ventas_2023q3 PARTITION OF ventas
    FOR VALUES FROM ('2023-07-01') TO ('2023-10-01');

-- Insertar datos en la nueva partición
INSERT INTO ventas (fecha_venta, monto)

SELECT 
    '2023-07-01'::date + (random() * 90)::integer,
    random() * 1000
FROM generate_series(1, 50000);
```

8. Eliminar una partición antigua.
```shell
DROP TABLE ventas_2023q1;
```

