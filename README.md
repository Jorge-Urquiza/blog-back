# 📌 block-back

API REST para la gestión de un blog básico, que permite la creación de publicaciones (*Posts*) y comentarios.  
Éste proyecto está desarrollado con **Spring Boot** y utiliza **MySQL** como base de datos.

---

## 🛠️ Tecnologías y Versiones

| Tecnología  | Versión        |
|-------------|----------------|
| Java        | 21             |
| Spring Boot | 3.4.0-SNAPSHOT |
| MySQL       | Última estable |
| Maven       | Última estable |

---

## 📋 Requisitos Previos

Antes de ejecutar el proyecto, asegúrate de tener instalado:

- [Java 21](https://www.oracle.com/java/technologies/downloads/#java21) o superior
- [Maven](https://maven.apache.org/) (versión 3.8+ recomendada)
- [MySQL](https://dev.mysql.com/downloads/) configurado y en ejecución

---

## 🚀 Ejecución en Local

1. **Clonar el repositorio**
   ```bash
   git clone https://github.com/Jorge-Urquiza/blog-back.git
   cd block-back
   ```

2. **Configurar base de datos**  
   Editar el archivo `application.properties` y definir:
   ```properties
   spring.datasource.url=jdbc:mysql://localhost:3306/blog
   spring.datasource.username=<usuario>
   spring.datasource.password=<contraseña>
   ```

3. **Compilar y ejecutar**
   ```bash
   mvn spring-boot:run
   ```

---

## 🏗️ Construcción y Empaquetado

Para construir el proyecto y generar el archivo `.jar` ejecuta:

```bash
mvn clean install
```

El artefacto generado se encontrará en:

```
target/block-back-1.0.0.jar
```

Para ejecutarlo:

```bash
java -jar target/block-back-1.0.0.jar
```

---

## CI con Jenkins (build + artefacto + smoke local)

Este proyecto usa un **Jenkinsfile** en la **raíz** del repo para ejecutar un pipeline CI sencillo en **Windows**.

### Requisitos del agente Jenkins
- **SO**: Windows (el pipeline usa `bat` y `mvnw.cmd`).
- **Java 21** disponible en `PATH` (`java -version`).
- **Git** instalado (`git --version`).
- **MySQL** accesible desde el agente (por defecto `localhost:3306` con DB `blog`, user `root`, pass `root`).
  - Si tu MySQL está en otra máquina, ajusta las variables del Jenkinsfile: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`.

### ¿Qué hace el pipeline?
1. **Checkout** de la rama `master`.
2. **Build prod-like sin tests**: `mvnw clean package -Dmaven.test.skip=true`.
3. **Stamp & Archive**: renombra el artefacto a `blog-back-<BUILD>-<GITSHA>.jar`, genera `SHA-256` y **publica artifacts**.
4. **Smoke run**:
   - Arranca el `.jar` con variables de entorno (URL, creds y `SERVER_PORT=8081`).
   - Tiene **timeout de 40s** y `catchError` para marcar el stage como **UNSTABLE** si expira (el build total queda **SUCCESS**).
   - Esto valida arranque/conexión sin dejar procesos colgados.

> **Nota**: el smoke usa **8081** por defecto para evitar conflictos locales. Cambia `SERVER_PORT` si lo necesitas.

### Cómo conectarlo a tu repo en Jenkins
**Opción A — Pipeline from SCM (recomendada)**
1. Crear **New Item → Pipeline**.
2. En **Definition**, elegir **Pipeline script from SCM**.
3. **SCM**: Git → URL del repo → Branch: `master`.
4. **Script Path**: `Jenkinsfile`.
5. Guardar y **Build Now**.

**Opción B — Pipeline script**
- Copia el contenido del `Jenkinsfile` en el campo **Pipeline script** del job.

### Variables clave (en el Jenkinsfile)
- `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`: conexión a MySQL (default: `localhost:3306`, DB `blog`, user `root`, pass `root`).
- `APP_PROFILE`: perfil activo de Spring (vacío por defecto).
- `SERVER_PORT`: puerto de la app durante el smoke (default `8081`).

> Edita estos valores en el `Jenkinsfile` según tu entorno.

### Artefactos generados
- `blog-back-<BUILD>-<GITSHA>.jar`
- `blog-back-<BUILD>-<GITSHA>.jar.sha256.txt`

Puedes verificar la integridad en Windows:
```bat
certutil -hashfile "blog-back-<BUILD>-<GITSHA>.jar" SHA256
type "blog-back-<BUILD>-<GITSHA>.jar.sha256.txt"

Para ejecutar:

cmd windows:

set JAR=blog-back-5-b7e4f675aa95.jar
set URL=jdbc:mysql://localhost:3306/blog?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
set USER=root
set PASS=root

rem 
start "DEV"  cmd /c java -jar "%JAR%" --server.port=8081 --spring.datasource.url="%URL%" --spring.datasource.username=%USER% --spring.datasource.password=%PASS% --spring.jpa.hibernate.ddl-auto=update
start "QA"   cmd /c java -jar "%JAR%" --server.port=8082 --spring.datasource.url="%URL%" --spring.datasource.username=%USER% --spring.datasource.password=%PASS% --spring.jpa.hibernate.ddl-auto=update
start "PROD" cmd /c java -jar "%JAR%" --server.port=8083 --spring.datasource.url="%URL%" --spring.datasource.username=%USER% --spring.datasource.password=%PASS% --spring.jpa.hibernate.ddl-auto=update


linux:

JAR=blog-back-5-b7e4f675aa95.jar
URL='jdbc:mysql://localhost:3306/blog?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC'
USER=root
PASS=root

nohup java -jar "$JAR" --server.port=8081 --spring.datasource.url="$URL" --spring.datasource.username=$USER --spring.datasource.password=$PASS --spring.jpa.hibernate.ddl-auto=update > dev.log 2>&1 &
nohup java -jar "$JAR" --server.port=8082 --spring.datasource.url="$URL" --spring.datasource.username=$USER --spring.datasource.password=$PASS --spring.jpa.hibernate.ddl-auto=update > qa.log 2>&1 &
nohup java -jar "$JAR" --server.port=8083 --spring.datasource.url="$URL" --spring.datasource.username=$USER --spring.datasource.password=$PASS --spring.jpa.hibernate.ddl-auto=update > prod.log 2>&1 &


## 👥 Equipo

| Nombre             |
|--------------------|
| Al Ivan Calle      |
| Jose Andres Mayser |
| Jorge Urquiza      |
| Daniel Zeballos    |

---
