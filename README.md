# INFORME TÉCNICO: MIGRACIÓN DE MARIADB A POSTGRESQL 18

**Asignatura:** Tecnología de Base de Datos I

**Docente:** Jared López Leaños

**Estudiante:** Ricardo Maldonado

**Repositorio GitHub:** https://github.com/ricardo0202/richi.git

**Fecha de Entrega:** 29 de Septiembre de 2026

---

## 1. RESUMEN EJECUTIVO

El presente documento detalla el procedimiento completo de migración de la base de datos relacional `employees` desde un motor MariaDB 11.8.9 hacia PostgreSQL 18. La arquitectura de origen y destino fue desplegada mediante un entorno de contenedores en Docker Compose.

El proceso incluyó la ejecución de scripts de carga automatizados con `pgloader`, la extracción y adaptación de la sintaxis de vistas SQL, la validación de integridad de los datos (conteos de registros y verificación de huérfanos), y la resolución de restricciones de peso de GitHub mediante la compresión del respaldo `.sql` a `.sql.gz`.

---

## 2. CONFIGURACIÓN DEL ENTORNO (DOCKER COMPOSE)

Ubicación del archivo: `~/tecBD1/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:18
    container_name: postgresql
    restart: unless-stopped

    environment:
      POSTGRES_DB: db_ricardo
      POSTGRES_USER: ricardo
      POSTGRES_PASSWORD: 123123
      TZ: 'America/La_Paz'
      PGTZ: 'America/La_Paz'
    command: ["postgres", "-c", "timezone=America/La_Paz"]

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ricardo -d db_ricardo"]
      interval: 10s
      timeout: 5s
      retries: 5

  mariadb:
    image: mariadb:11.8.9-ubi9
    container_name: mariadb
    restart: always
    environment:
      MARIADB_ROOT_PASSWORD: 123456
      MARIADB_DATABASE: ricardo
      MARIADB_USER: ricardo
      MARIADB_PASSWORD: 123123
    volumes:
      - mariadb_data:/var/lib/mysql
    ports:
      - "3306:3306"

  adminer:
    image: adminer
    container_name: adminer
    restart: always
    ports:
      - 8081:8080

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin4
    restart: always
    ports:
      - "8080:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@upds.com
      PGADMIN_DEFAULT_PASSWORD: 123123
    volumes:
      - pgadmin-data:/var/lib/pgadmin

volumes:
  mariadb_data:
    driver: local
  postgres_data:
    driver: local
  pgadmin-data:
    driver: local
```

---

## 3. DESARROLLO DE LA MIGRACIÓN Y CÓDIGO UTILIZADO

### 3.1. Punto 1: Migración de Tablas y Datos (15 pts)

#### 1. Archivo de Configuración de `pgloader` (`migracion.load`)

Se creó el script de carga automatizado para realizar el mapeo entre MariaDB y PostgreSQL:

```sql
LOAD DATABASE
     FROM mysql://ricardo:123123@mariadb:3306/employees
     INTO postgresql://ricardo:123123@postgresql:5432/pdb_employees

 WITH include drop, create tables, create indexes, reset sequences,
      workers = 8, concurrency = 1,
      single transaction

  SET PostgreSQL PARAMETERS
      maintenance_work_mem to '128MB',
      work_mem to '12MB';
```

#### 2. Comando de Ejecución de Migración

```bash
pgloader migracion.load
```

### 3.2. Punto 2: Migración y Adaptación de Vistas (10 pts)

#### 1. Extracción de la Vista Original en MariaDB

Comando de consulta:

```bash
docker exec -it mariadb mariadb -u ricardo -p123123 --ssl=0 -e "SHOW CREATE VIEW employees.v_full_employees;" > vista_latest_mariadb.txt
```

Definición obtenida en MariaDB:

```sql
CREATE ALGORITHM=UNDEFINED DEFINER=`ricardo`@`%` SQL SECURITY DEFINER VIEW `v_full_employees` AS 
SELECT `employees`.`emp_no` AS `emp_no`,
       CONCAT(`employees`.`first_name`,' ',`employees`.`last_name`) AS `full_name`,
       `employees`.`gender` AS `gender`,
       `employees`.`hire_date` AS `hire_date` 
FROM `employees`;
```

#### 2. Creación y Adaptación de la Vista en PostgreSQL

Se reemplazó la función `CONCAT()` por el operador de concatenación de PostgreSQL (`||`) y se eliminaron las cláusulas específicas de MariaDB:

```bash
docker exec -i postgresql psql -U ricardo -d pdb_employees -c "CREATE OR REPLACE VIEW v_full_employees AS SELECT emp_no, first_name || ' ' || last_name AS full_name, gender, hire_date FROM employees;"
```

Código DDL resultante en PostgreSQL:

```sql
CREATE OR REPLACE VIEW v_full_employees AS
SELECT 
    emp_no, 
    first_name || ' ' || last_name AS full_name,
    gender,
    hire_date
FROM employees;
```

#### 3. Consulta de Prueba en PostgreSQL

Comando de prueba:

```bash
docker exec -i postgresql psql -U ricardo -d pdb_employees -c "SELECT * FROM v_full_employees LIMIT 5;" > prueba_vista_latest.txt
```

Resultado obtenido:

```text
 emp_no |     full_name     | gender | hire_date  
--------+-------------------+--------+------------
  10001 | Georgi Facello    | M      | 1986-06-26
  10002 | Bezalel Simmel    | F      | 1985-11-21
  10003 | Particio Bamford  | M      | 1986-08-28
  10004 | Chirstian Koblick | M      | 1986-12-01
  10005 | Kyoichi Maliniak  | M      | 1989-09-12
(5 rows)
```

### 3.3. Punto 3: Consultas de Verificación e Integridad (25 pts)

#### 1. Verificación de Conteo de Filas

Comando ejecutado para MariaDB:

```bash
docker exec -i mariadb mariadb -u ricardo -p123123 --ssl=0 -e "SELECT 'employees' AS tabla, COUNT(*) FROM employees.employees UNION ALL SELECT 'departments', COUNT(*) FROM employees.departments;" > final_conteos_mariadb.txt
```

Comando ejecutado para PostgreSQL:

```bash
docker exec -i postgresql psql -U ricardo -d pdb_employees -c "SELECT 'employees' AS tabla, COUNT(*) FROM employees UNION ALL SELECT 'departments', COUNT(*) FROM departments;"
```

| Tabla | Conteo MariaDB | Conteo PostgreSQL | Coincidencia |
| --- | --- | --- | --- |
| `employees` | 300,024 | 300,024 | Exacta |
| `departments` | 9 | 9 | Exacta |

#### 2. Verificación de Integridad Referencial (Búsqueda de Huérfanos)

Se evaluó la relación entre las tablas `dept_emp` y `departments`:

* **MariaDB:**
```bash
docker exec -i mariadb mariadb -u ricardo -p123123 --ssl=0 -e "SELECT COUNT(*) AS huerfanos FROM employees.dept_emp de LEFT JOIN employees.departments d ON de.dept_no = d.dept_no WHERE d.dept_no IS NULL;" > huerfanos_mariadb.txt
```

* **PostgreSQL:**
```bash
docker exec -i postgresql psql -U ricardo -d pdb_employees -c "SELECT COUNT(*) AS huerfanos FROM dept_emp de LEFT JOIN departments d ON de.dept_no = d.dept_no WHERE d.dept_no IS NULL;" > huerfanos_postgresql.txt
```

* **Resultado:** `0` registros huérfanos reportados en ambos motores, garantizando la integridad de las claves foráneas.

---

## 4. RESPALDO Y COMANDOS DE VERSIONADO EN GITHUB

### 4.1. Script Auxiliar de Verificación (`ac06.sh`)

```bash
cat << 'EOF' > ac06.sh
#!/bin/bash
echo "Verificando estado de contenedores..."
docker ps
EOF
chmod +x ac06.sh
```

### 4.2. Generación y Compresión del Respaldo

```bash
# Exportación de la base de datos PostgreSQL
docker exec -i postgresql pg_dump -U ricardo -d pdb_employees > backup_pdb_employees.sql

# Compresión para evitar el límite de 100 MB de GitHub
gzip -f backup_pdb_employees.sql
```

### 4.3. Configuración y Subida a GitHub

```bash
# Configuración del perfil de Git
git config --global user.email "ricardo@ejemplo.com"
git config --global user.name "Ricardo Maldonado"

# Inicialización y vinculación remota
git init
git branch -M main
git remote add origin https://github.com/ricardo0202/richi.git

# Selección de los archivos requeridos
git add README.md \
        docker-compose.yml \
        migracion.load \
        backup_pdb_employees.sql.gz \
        ac06.sh \
        final_conteos_mariadb.txt \
        huerfanos_mariadb.txt \
        huerfanos_postgresql.txt \
        prueba_vista_latest.txt \
        vista_latest_mariadb.txt

# Commit y push forzado a la rama main
git commit -m "Entrega Final TBDI - Informe, backup comprimido y evidencias de migracion"
git push -u origin main -f
```

---

## 5. ARCHIVOS ENTREGADOS EN EL REPOSITORIO

El repositorio `https://github.com/ricardo0202/richi.git` contiene los siguientes artefactos:

* `README.md`: Informe general formateado.

* `docker-compose.yml`: Infraestructura de contenedores.

* `backup_pdb_employees.sql.gz`: Copia de respaldo comprimida de PostgreSQL.

* `migracion.load`: Script de migración de pgloader.

* `ac06.sh`: Script ejecutable de soporte.

* `final_conteos_mariadb.txt`: Resultado de verificación de filas.

* `huerfanos_mariadb.txt` / `huerfanos_postgresql.txt`: Reporte de integridad de datos.

* `vista_latest_mariadb.txt` / `prueba_vista_latest.txt`: Definición y pruebas de la vista adaptada.

