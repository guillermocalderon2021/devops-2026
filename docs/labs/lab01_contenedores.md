# Laboratorio 1. Construcción y optimización de contenedores

## Información general

| Elemento | Detalle |
|---|---|
| Modalidad | Parejas |
| Ponderación | 5% de la calificación final |
| Fecha límite | Lunes 21 de septiembre de 2026 |
| Repositorio inicial | [devops-lab01-starter](https://github.com/guillermocalderon2021/devops-lab01-starter) |
| Entrega | Enlace al repositorio privado de la pareja en Moodle |

El laboratorio parte de una API desarrollada con Express y TypeScript que utiliza PostgreSQL como base de datos. El repositorio suministrado contiene una implementación funcional de la aplicación y una configuración inicial de Docker y Docker Compose que presenta decisiones técnicas que deben ser analizadas, corregidas y verificadas.

El objetivo no es modificar la lógica de negocio de la aplicación. El trabajo debe concentrarse en la construcción de la imagen, la configuración de ejecución, la coordinación entre servicios y la reproducibilidad del entorno.

---

## Resultados de aprendizaje

Al finalizar el laboratorio, los estudiantes deberán ser capaces de:

1. analizar una configuración de contenedores e identificar decisiones que afectan el tamaño de la imagen, la reproducibilidad, el uso de caché, la seguridad y la configuración;
2. construir una imagen de contenedor utilizando apropiadamente el contexto de construcción, `.dockerignore`, caché y multi-stage builds;
3. separar las dependencias y herramientas requeridas durante la construcción de aquellas necesarias durante la ejecución;
4. ejecutar la aplicación con un usuario no privilegiado y sin incorporar credenciales ni configuración sensible dentro de la imagen;
5. coordinar una aplicación y una base de datos mediante Docker Compose, considerando disponibilidad, inicialización y persistencia;
6. verificar mediante evidencia técnica que la solución final puede reconstruirse y ejecutarse de forma reproducible.

---

# 1. Preparación del repositorio

El repositorio inicial del laboratorio es un **template repository**. No debe realizarse un fork.

Uno de los integrantes de la pareja deberá:

1. abrir el repositorio inicial;
2. seleccionar **Use this template**;
3. crear un nuevo repositorio **privado**;
4. agregar al otro integrante de la pareja como colaborador;
5. agregar al docente como colaborador;
6. compartir con su pareja el enlace del nuevo repositorio;
7. clonar el repositorio privado en el equipo de trabajo.

El repositorio privado creado a partir del template constituirá la entrega oficial de la pareja.

No debe trabajarse directamente sobre el repositorio público suministrado.

---

# 2. Sistema suministrado

El repositorio contiene los siguientes componentes principales:

```text
src/
scripts/
Dockerfile
compose.yaml
.dockerignore
.env.example
.gitignore
package.json
package-lock.json
tsconfig.json
README.md
```

La aplicación expone los siguientes endpoints:

```text
GET  /health
GET  /products
GET  /products/:id
POST /products
```

La base de datos utiliza PostgreSQL y debe contener una tabla `products` con datos iniciales.

El repositorio también incluye un script que permite crear la estructura de la base de datos y cargar los datos iniciales.

La lógica de Express, las consultas SQL, el esquema de la base de datos y el script de inicialización se consideran parte del sistema suministrado y no constituyen el objeto principal del laboratorio.

---

# 3. Análisis de la implementación inicial

Antes de realizar modificaciones, la pareja deberá construir y ejecutar la configuración suministrada.

Como mínimo, deberá analizarse:

- el contexto enviado al proceso de construcción;
- el orden de las instrucciones del `Dockerfile`;
- el mecanismo utilizado para instalar dependencias;
- la utilización de caché durante reconstrucciones;
- el contenido que permanece en la imagen final;
- el usuario configurado para ejecutar la aplicación;
- las variables incorporadas a la imagen;
- la comunicación entre la API y PostgreSQL;
- el orden y las condiciones de inicio de los servicios;
- el procedimiento de inicialización de la base de datos;
- la persistencia de los datos.

El comportamiento inicial puede presentar fallas. Estas forman parte del escenario del laboratorio y deben analizarse utilizando evidencia técnica.

La pareja deberá documentar en `EVIDENCIAS.md` al menos **seis problemas o decisiones técnicas que deban corregirse**, utilizando la siguiente estructura:

| Problema identificado | Evidencia observada | Consecuencia técnica | Corrección realizada |
|---|---|---|---|
| | | | |

No se evaluarán afirmaciones genéricas como “no es una buena práctica”. Cada problema deberá relacionarse con un comportamiento, riesgo o efecto técnico concreto.

---

# 4. Construcción de la imagen

La solución final deberá producir una imagen apta para ejecución que cumpla los siguientes requisitos.

## 4.1 Contexto de construcción

El contexto de construcción deberá excluir archivos y directorios locales que no sean necesarios para construir la aplicación.

Como mínimo, deberá analizarse si deben formar parte del contexto:

- dependencias instaladas localmente;
- artefactos compilados localmente;
- archivos de control de versiones;
- archivos de configuración local;
- archivos de log.

La pareja deberá justificar en `EVIDENCIAS.md` las exclusiones realizadas.

## 4.2 Instalación reproducible de dependencias

La construcción deberá utilizar el archivo `package-lock.json` para realizar una instalación reproducible.

La pareja deberá justificar el comando utilizado para instalar las dependencias durante el proceso de construcción.

## 4.3 Uso de caché

El `Dockerfile` deberá organizarse de manera que cambios frecuentes en el código fuente no obliguen innecesariamente a reinstalar las dependencias cuando los archivos que las definen no han cambiado.

Deberá realizarse el siguiente experimento:

1. construir la imagen;
2. modificar únicamente un archivo de código fuente;
3. reconstruir la imagen;
4. identificar qué pasos fueron reutilizados desde caché;
5. documentar la evidencia.

## 4.4 Separación entre construcción y ejecución

La imagen utilizada para ejecutar la aplicación no deberá contener herramientas requeridas exclusivamente durante la compilación.

La solución deberá utilizar un **multi-stage build** que separe al menos:

- una etapa de construcción;
- una etapa de ejecución.

La etapa final deberá contener únicamente los artefactos y dependencias requeridos para ejecutar la aplicación y el proceso de inicialización de la base de datos.

## 4.5 Imagen base

La pareja deberá seleccionar una imagen base apropiada para la etapa de ejecución.

La selección deberá justificarse brevemente considerando, como mínimo:

- compatibilidad con la aplicación;
- contenido necesario en ejecución;
- tamaño de la imagen;
- implicaciones operativas de la variante seleccionada.

No se asignará una reducción porcentual mínima de tamaño. Se evaluará la corrección técnica de la solución y la justificación de la decisión.

---

# 5. Seguridad y configuración

## 5.1 Usuario de ejecución

La aplicación no deberá ejecutarse como `root`.

La pareja deberá comprobar mediante un comando o inspección de la imagen cuál es el usuario configurado para la etapa de ejecución e incluir la evidencia en `EVIDENCIAS.md`.

## 5.2 Configuración de entorno

La configuración necesaria para conectar y ejecutar los servicios deberá proporcionarse en tiempo de ejecución y no quedar incorporada al Dockerfile ni a la imagen construida.

La misma imagen deberá poder utilizarse con diferentes valores de configuración sin necesidad de reconstruirla. Esto incluye, entre otros:

- host de la base de datos;
- nombre de la base de datos;
- usuario;
- contraseña;
- puerto publicado para acceder a la aplicación desde el host.

Los valores correspondientes al entorno local podrán suministrarse mediante Docker Compose y variables de entorno.

## 5.3 Credenciales

Ninguna contraseña utilizada para ejecutar los servicios deberá quedar:

- incorporada al `Dockerfile`;
- almacenada dentro de la imagen;
- versionada en el repositorio privado.

Puede utilizarse un archivo local `.env` para proporcionar valores durante la ejecución. Este archivo no constituye un sistema completo de gestión de secretos y no debe versionarse.

El repositorio deberá conservar únicamente un archivo `.env.example` con los nombres de las variables requeridas y valores de ejemplo no sensibles.

---

# 6. Ejecución con Docker Compose

La solución final deberá permitir reconstruir y ejecutar el sistema completo mediante Docker Compose.

Como mínimo, deberán existir los componentes necesarios para:

- ejecutar PostgreSQL;
- preparar la estructura y los datos iniciales de la base;
- ejecutar la API.

## 6.1 Comunicación entre servicios

La API deberá conectarse a PostgreSQL utilizando el mecanismo de resolución de nombres proporcionado por Docker Compose.

La solución no deberá depender de direcciones IP estáticas ni de configuraciones manuales posteriores al arranque.

## 6.2 Disponibilidad de PostgreSQL

El hecho de que el contenedor de PostgreSQL haya iniciado no implica que la base de datos ya pueda aceptar conexiones.

La solución deberá:

- definir un `healthcheck` apropiado para PostgreSQL;
- utilizar una condición de dependencia que espere a que PostgreSQL se encuentre saludable antes de ejecutar procesos que necesiten la base de datos.

## 6.3 Inicialización de la base de datos

La creación del esquema y la carga de datos deberán integrarse en el ciclo de Docker Compose.

La solución final no deberá requerir que una persona ejecute manualmente el script de inicialización después de levantar los servicios.

La tarea de inicialización deberá:

- esperar a que PostgreSQL esté disponible;
- ejecutar el script suministrado;
- finalizar correctamente;
- permitir que la API inicie únicamente si la inicialización terminó de forma satisfactoria.

## 6.4 Persistencia

Los datos de PostgreSQL deberán almacenarse en un volumen administrado por Docker.

La siguiente secuencia no deberá eliminar los datos de la aplicación:

```bash
docker compose down
docker compose up
```

La pareja deberá demostrarlo creando un producto, recreando los contenedores y comprobando posteriormente que el producto continúa almacenado.

---

# 7. Reproducibilidad de la solución

La solución deberá poder reconstruirse desde un clon nuevo del repositorio privado sin requerir Node.js ni PostgreSQL instalados localmente.

El procedimiento esperado deberá reducirse, en esencia, a:

1. clonar el repositorio;
2. crear el archivo local de variables de entorno a partir del ejemplo;
3. proporcionar los valores requeridos;
4. construir y levantar la solución con Docker Compose.

No deberán existir pasos manuales no documentados para:

- crear tablas;
- cargar datos;
- instalar dependencias en el host;
- modificar archivos internos del contenedor;
- conectarse manualmente a PostgreSQL para preparar el sistema.

---

# 8. Verificaciones obligatorias

La pareja deberá comprobar y documentar los siguientes puntos.

## 8.1 Imagen

Debe registrarse:

- tamaño de la imagen inicial;
- tamaño de la imagen final;
- usuario configurado en la imagen final;
- ausencia de las credenciales de la base de datos en la configuración persistente de la imagen;
- ausencia de dependencias de desarrollo innecesarias en la etapa de ejecución.

## 8.2 Caché

Debe mostrarse evidencia de que una modificación únicamente en código fuente permite reutilizar la capa de instalación de dependencias.

## 8.3 Servicios

Debe comprobarse que:

- PostgreSQL alcanza un estado saludable;
- la tarea de inicialización finaliza correctamente;
- la API queda en ejecución después de completarse la inicialización;
- `GET /health` responde satisfactoriamente;
- `GET /products` devuelve los datos iniciales.

## 8.4 Persistencia

Debe demostrarse que un producto creado mediante la API permanece disponible después de recrear los contenedores sin eliminar los volúmenes.

---

# 9. Evidencia técnica

La pareja deberá crear en la raíz del repositorio:

```text
EVIDENCIAS.md
```

El documento deberá contener únicamente la evidencia necesaria para demostrar las decisiones realizadas.

Como mínimo deberá incluir:

1. tabla de diagnóstico de la implementación inicial;
2. comparación entre imagen inicial y final;
3. evidencia del experimento de caché;
4. justificación de la imagen base seleccionada;
5. evidencia del usuario de ejecución;
6. evidencia de que las credenciales no permanecen en la configuración de la imagen;
7. evidencia del estado de los servicios de Compose;
8. evidencia de persistencia;
9. explicación breve de la secuencia de arranque del sistema final.

No se requiere un informe narrativo extenso.

Las capturas de pantalla deben utilizarse únicamente cuando aporten información que no pueda presentarse de manera más clara mediante texto u output de terminal.

---

# 10. Repositorio de trabajo

La pareja trabajará sobre el repositorio privado creado a partir del template suministrado.

El repositorio deberá conservar el historial de cambios realizados durante el desarrollo del laboratorio y la rama `main` deberá contener la versión definitiva de la solución al momento de la entrega.

No se exige una estrategia específica de ramas ni el uso obligatorio de pull requests para este laboratorio. Las decisiones de organización del trabajo dentro del repositorio quedan a criterio de la pareja, siempre que la versión entregada sea reproducible y técnicamente correcta.

---

# 11. Entrega

La entrega se realizará mediante Moodle.

Deberá enviarse:

1. enlace al repositorio privado de la pareja;
2. nombres y carnets de ambos integrantes.

Antes de enviar el enlace deberá comprobarse que:

- el docente tiene acceso al repositorio;
- la rama `main` contiene la solución final;
- existe `EVIDENCIAS.md`;
- existe `.env.example`;
- `.env` no está versionado;
- la solución puede construirse desde cero.

**Fecha límite: Lunes 21 de septiembre de 2026.**

---

# 12. Uso de herramientas de inteligencia artificial

Se permite utilizar documentación técnica, motores de búsqueda y herramientas de inteligencia artificial generativa durante el desarrollo del laboratorio.

La utilización de estas herramientas no transfiere la responsabilidad sobre el trabajo entregado.

Cada integrante deberá:

- comprender las modificaciones realizadas;
- verificar los comandos y configuraciones sugeridos por herramientas externas;
- poder explicar las decisiones técnicas incluidas en la solución;
- reconocer las limitaciones o riesgos de las alternativas utilizadas.

No se requiere entregar un registro de prompts.

---

# 13. Criterios de evaluación

El laboratorio se calificará sobre 100 puntos.

| Componente | Puntos |
|---|---:|
| Diagnóstico técnico de la implementación inicial | 10 |
| Contexto de construcción, `.dockerignore` y aprovechamiento de caché | 15 |
| Multi-stage build y separación entre construcción y ejecución | 20 |
| Imagen de ejecución, dependencias y optimización | 10 |
| Usuario no privilegiado y separación de configuración sensible | 15 |
| Docker Compose: comunicación, disponibilidad e inicialización | 15 |
| Persistencia y reproducibilidad del entorno | 10 |
| Evidencia técnica y entrega | 5 |
| **Total** | **100** |

## 13.1 Diagnóstico técnico — 10 puntos

Se evaluará:

- identificación de problemas reales;
- evidencia que sustenta cada observación;
- explicación de las consecuencias técnicas;
- correspondencia entre el diagnóstico y las correcciones realizadas.

## 13.2 Contexto y caché — 15 puntos

Se evaluará:

- `.dockerignore` apropiado;
- exclusión de contenido innecesario;
- instalación reproducible de dependencias;
- organización del `Dockerfile` para aprovechar la caché;
- evidencia del experimento solicitado.

## 13.3 Multi-stage build — 20 puntos

Se evaluará:

- separación correcta de etapas;
- compilación dentro de la etapa correspondiente;
- transferencia únicamente de artefactos necesarios;
- ausencia de herramientas de construcción innecesarias en la imagen final;
- funcionamiento correcto del script de inicialización en la imagen final.

## 13.4 Imagen de ejecución — 10 puntos

Se evaluará:

- selección apropiada de la imagen base;
- justificación técnica;
- dependencias necesarias en runtime;
- funcionamiento correcto de la aplicación.

## 13.5 Seguridad y configuración — 15 puntos

Se evaluará:

- ejecución con usuario no privilegiado;
- ausencia de credenciales dentro del Dockerfile y de la imagen;
- separación entre imagen y configuración del entorno;
- tratamiento apropiado de `.env` y `.env.example`.

## 13.6 Docker Compose — 15 puntos

Se evaluará:

- comunicación correcta entre servicios;
- `healthcheck` funcional para PostgreSQL;
- dependencia basada en estado saludable;
- inicialización automática de esquema y datos;
- dependencia de la API respecto del resultado satisfactorio de la inicialización.

## 13.7 Persistencia y reproducibilidad — 10 puntos

Se evaluará:

- volumen persistente para PostgreSQL;
- conservación de datos después de recrear contenedores;
- reconstrucción del sistema sin instalaciones locales de Node.js o PostgreSQL;
- ausencia de pasos manuales no documentados.

## 13.8 Evidencia técnica y entrega — 5 puntos

Se evaluará:

- claridad y suficiencia de `EVIDENCIAS.md`;
- presencia y corrección de `.env.example`;
- ausencia de `.env` en el historial versionado;
- repositorio accesible para el docente;
- rama `main` en estado ejecutable;
- información suficiente para reproducir la solución desde un clon nuevo.

---

# 14. Criterios de aceptación

La solución se considerará técnicamente completa cuando, a partir de un clon nuevo del repositorio:

1. pueda configurarse el entorno local sin modificar archivos versionados;
2. la imagen pueda construirse sin depender de `node_modules` ni `dist` generados en el host;
3. Docker Compose pueda levantar PostgreSQL, inicializar la base y ejecutar la API sin intervención manual intermedia;
4. PostgreSQL alcance un estado saludable antes de las operaciones que dependen de él;
5. la inicialización finalice satisfactoriamente antes de iniciar la API;
6. `GET /health` responda correctamente;
7. `GET /products` devuelva los datos iniciales;
8. los datos creados mediante la API sobrevivan a la recreación de los contenedores;
9. la aplicación se ejecute con un usuario no privilegiado;
10. las credenciales no estén incorporadas a la imagen ni versionadas en el repositorio.
