# Laboratorio 2. Pipeline de integración continua

| Información | Detalle |
|---|---|
| Modalidad | Parejas |
| Ponderación | 5% |
| Fecha límite | Sábado 26 de septiembre de 2026 |
| Repositorio base | [devops-cicd-starter](https://github.com/guillermocalderon2021/devops-cicd-starter) |
| Entrega | Enlace al repositorio privado  |

## 1. Propósito

En este laboratorio se construirá un pipeline de integración continua para una API desarrollada con FastAPI.

La aplicación, sus pruebas automatizadas y su imagen de contenedor se proporcionan en estado funcional. El objeto del laboratorio no es desarrollar la aplicación ni corregir su Dockerfile, sino diseñar un proceso de integración continua que valide automáticamente los cambios propuestos antes de incorporarlos a la rama `main`.

El repositorio utilizado en este laboratorio continuará utilizándose en el Laboratorio 3, donde el pipeline se extenderá hacia entrega continua y despliegue sobre servicios de Google Cloud. Por esta razón, el resultado de este laboratorio deberá conservarse en un estado reproducible y funcional.

## 2. Resultados esperados

Al finalizar el laboratorio, la pareja deberá ser capaz de:

1. configurar un workflow de GitHub Actions que responda a cambios propuestos y cambios integrados;
2. separar validaciones independientes en jobs que puedan ejecutarse en paralelo;
3. utilizar dependencias entre jobs para implementar gates dentro del pipeline;
4. incorporar análisis estático, pruebas automatizadas y cobertura dentro de CI;
5. reutilizar caché de dependencias para evitar trabajo innecesario;
6. conservar resultados producidos por una ejecución mediante workflow artifacts;
7. verificar que un cambio puede producir correctamente la imagen de contenedor de la aplicación;
8. identificar la imagen construida a partir del commit que originó la ejecución;
9. limitar los permisos concedidos al workflow;
10. configurar un ruleset que impida integrar cambios mientras las validaciones requeridas no hayan finalizado satisfactoriamente;
11. utilizar evidencia de GitHub Actions para diagnosticar fallas del pipeline.

## 3. Requisito de GitHub Education

El repositorio de la pareja será privado y pertenecerá a la cuenta personal de uno de sus integrantes.

El propietario del repositorio deberá disponer de **GitHub Pro** mediante GitHub Education. Este requisito permite utilizar branch rulesets sobre un repositorio privado.

La verificación estudiantil puede solicitarse en:

<https://education.github.com/pack/join>

Antes de iniciar el laboratorio deberá comprobarse que el propietario del repositorio tiene activo el beneficio correspondiente.

Si la aprobación de GitHub Education continúa pendiente al iniciar el laboratorio, deberá informarse al docente antes de continuar con la configuración del ruleset.

## 4. Preparación del repositorio

Uno de los integrantes deberá crear el repositorio de trabajo a partir del template suministrado.

No deberá utilizarse `Fork`.

La secuencia será:

1. abrir el repositorio `devops-cicd-starter`;
2. seleccionar **Use this template**;
3. crear un repositorio **privado** en la cuenta personal de uno de los integrantes;
4. agregar como colaboradores al otro integrante de la pareja y al docente, utilizando para este último la cuenta de GitHub **`guillermocalderon2021`**;
5. clonar el nuevo repositorio;
6. comprobar que la rama `main` contiene el estado inicial suministrado.

El repositorio privado creado por la pareja será el repositorio oficial de trabajo para los laboratorios 2 y 3.

El acceso del docente al repositorio es obligatorio. Deberá comprobarse que la cuenta de GitHub **`guillermocalderon2021`** tenga acceso al repositorio privado antes de realizar la entrega.

## 5. Sistema suministrado

El proyecto contiene una API para administrar incidencias.

Entre sus operaciones se encuentran:

```text
GET    /health
GET    /incidents
GET    /incidents/{id}
POST   /incidents
PATCH  /incidents/{id}/status
```

La aplicación utiliza FastAPI y contiene:

- lógica de negocio ya implementada;
- almacenamiento en memoria para ejecución local y pruebas;
- una implementación preparada para Firestore que se utilizará posteriormente;
- pruebas automatizadas;
- configuración de Ruff;
- un Dockerfile funcional;
- un workflow inicial de GitHub Actions.

Durante este laboratorio **no se requiere una cuenta de Google Cloud ni una instancia de Firestore**.

Antes de modificar el workflow deberá comprobarse localmente que el proyecto suministrado se encuentra en estado funcional:

```bash
python -m ruff check .
python -m pytest
docker build -t incident-api:local .
```

Estos comandos deben ejecutarse dentro del entorno virtual donde instal'o las dependencias (revise el readme.md del repositorio).

## 6. Workflow inicial

El repositorio incluye un workflow inicial de GitHub Actions.

Este workflow es funcional, pero representa únicamente un punto de partida. No satisface los requisitos finales del laboratorio.

Antes de modificarlo, deberá analizarse:

- qué evento provoca actualmente su ejecución;
- cuántos jobs contiene;
- qué validación realiza;
- qué validaciones relevantes todavía no forman parte del pipeline;
- qué trabajo se realiza de forma secuencial;
- qué resultados producidos por la ejecución se conservan;
- qué condición determina actualmente si un cambio es integrable.

Las conclusiones de este análisis no requieren un informe separado, pero deberán comprenderse antes de modificar el workflow.

## 7. Flujo de trabajo con Git

Para este laboratorio se utilizará una rama de trabajo distinta de `main`.

Puede utilizarse, por ejemplo:

```text
lab02-ci
```

Los cambios finales deberán integrarse mediante un pull request cuyo destino sea `main`.

El pull request no deberá fusionarse hasta haber:

1. construido el pipeline solicitado;
2. ejecutado los experimentos de fallo indicados en esta guía;
3. configurado el ruleset de `main`;
4. comprobado que el ruleset bloquea la integración cuando corresponde;
5. restaurado el proyecto a un estado satisfactorio.

No se exige una cantidad específica de commits ni una distribución artificial de commits entre los integrantes.

## 8. Eventos del workflow

El workflow final deberá ejecutarse como mínimo ante:

- un `push` a `main`;
- un `pull_request` cuyo destino sea `main`.

La configuración deberá evitar ejecuciones que no correspondan a estos casos.

## 9. Job de análisis estático

El workflow deberá contener un job independiente dedicado al análisis estático.

Este job deberá:

- utilizar un runner de GitHub;
- obtener el contenido del repositorio;
- configurar la versión de Python utilizada por el proyecto;
- instalar las dependencias necesarias;
- ejecutar Ruff mediante:

```bash
python -m ruff check .
```

El job deberá aparecer en GitHub Actions con el nombre:

```text
Lint
```

## 10. Job de pruebas

El workflow deberá contener un segundo job independiente dedicado a pruebas automatizadas.

Este job deberá:

- utilizar un runner de GitHub;
- obtener el contenido del repositorio;
- configurar Python;
- instalar las dependencias necesarias;
- ejecutar la suite de pruebas;
- medir cobertura sobre el paquete `app`;
- generar un reporte de cobertura en formato XML.

El job deberá aparecer en GitHub Actions con el nombre:

```text
Test
```

No se establece un porcentaje mínimo de cobertura. Las pruebas suministradas forman parte del proyecto base y no deberán modificarse para aumentar artificialmente el resultado.

### 10.1 Workflow artifact

El reporte XML de cobertura deberá conservarse como workflow artifact.

El artifact deberá permitir comprobar, desde la ejecución correspondiente en GitHub Actions, que el archivo producido por las pruebas quedó disponible después de finalizar el job.

No se requiere almacenar el artifact dentro del repositorio Git.

## 11. Paralelización

Los jobs `Lint` y `Test` deberán ser independientes.

Ninguno deberá declarar una dependencia respecto del otro.

Por tanto, GitHub Actions deberá poder ejecutarlos de forma concurrente:

```text
                 ┌───────┐
cambio ─────────►│ Lint  │
        │        └───────┘
        │
        │        ┌───────┐
        └───────►│ Test  │
                 └───────┘
```

No deberá unificarse lint y pruebas dentro de un único job.

## 12. Caché de dependencias

Los jobs que instalan las dependencias Python deberán aprovechar el mecanismo de caché disponible para `pip`.

La caché deberá estar asociada a los archivos que determinan las dependencias del proyecto, de forma que:

- una ejecución posterior pueda reutilizar dependencias cuando estas no hayan cambiado;
- un cambio en las dependencias provoque la actualización correspondiente de la caché.

La evidencia deberá permitir distinguir entre una ejecución que prepara la caché y una ejecución posterior que puede reutilizarla.

## 13. Job de construcción del contenedor

El workflow deberá contener un tercer job encargado de comprobar que el commit puede producir correctamente la imagen de la aplicación.

El job deberá aparecer en GitHub Actions con el nombre:

```text
Container
```

Este job deberá depender de:

```text
Lint
Test
```

y no deberá ejecutarse normalmente si alguna de esas validaciones falla.

La estructura conceptual esperada es:

```text
             ┌────────┐
             │  Lint  │
             └───┬────┘
                 │
                 │
                 ▼
             ┌───────────┐
             │ Container │
                 ▲
                 │
             ┌───┴────┐
             │  Test  │
             └────────┘
```

El job deberá construir el Dockerfile suministrado.

La imagen construida deberá utilizar como parte de su identificación el SHA del commit que produjo la ejecución.

Conceptualmente:

```text
incident-api:<commit-sha>
```

En este laboratorio **no deberá publicarse la imagen en ningún registry**.

La publicación del artefacto de contenedor y su despliegue corresponden al Laboratorio 3.

## 14. Permisos del workflow

El workflow deberá declarar explícitamente sus permisos.

Solo deberán concederse los permisos requeridos para las operaciones realizadas durante este laboratorio.

La solución no necesita permisos de escritura sobre el repositorio.

No deberán incorporarse credenciales, tokens personales ni secretos para satisfacer los requisitos de este laboratorio.

## 15. Ruleset de la rama `main`

Una vez que los tres checks hayan aparecido al menos una vez en el repositorio, deberá configurarse un **branch ruleset** para proteger `main`.

En:

```text
Settings
→ Rules
→ Rulesets
→ New branch ruleset
```

el ruleset deberá:

1. estar activo;
2. aplicarse a la rama `main`;
3. exigir que los cambios se integren mediante pull request;
4. exigir que los status checks requeridos finalicen satisfactoriamente antes del merge.

Los tres checks obligatorios serán:

```text
Lint
Test
Container
```

No se exige una cantidad mínima de aprobaciones de revisores.

No deberán configurarse excepciones o bypasses para evitar las validaciones solicitadas.

Si los nombres `Lint`, `Test` y `Container` todavía no aparecen al configurar los status checks requeridos, deberá ejecutarse primero el workflow con esos jobs y volver posteriormente a la configuración del ruleset.

## 16. Experimento de fallo de lint

El pipeline no deberá evaluarse únicamente mediante una ejecución satisfactoria.

En la rama del laboratorio deberá provocarse temporalmente una violación detectable por Ruff.

Puede utilizarse, por ejemplo, un archivo temporal que contenga un import no utilizado.

Después del `push`, deberá comprobarse un comportamiento equivalente a:

```text
Lint       ❌
Test       ✅
Container  no ejecutado
```

Además, el pull request deberá quedar **bloqueado para merge** por el ruleset.

Deberá conservarse el enlace a esta ejecución como evidencia.

Después del experimento, la violación deberá eliminarse y el cambio correctivo deberá enviarse al mismo pull request.

## 17. Experimento de fallo de pruebas

Con el lint nuevamente correcto, deberá provocarse temporalmente una falla de pruebas.

Puede crearse un archivo de prueba temporal con una aserción deliberadamente falsa.

Después del `push`, deberá comprobarse un comportamiento equivalente a:

```text
Lint       ✅
Test       ❌
Container  no ejecutado
```

El ruleset deberá mantener bloqueado el merge.

Deberá conservarse el enlace a esta ejecución.

Posteriormente, la prueba deliberadamente incorrecta deberá eliminarse.

Las pruebas suministradas con el proyecto no deberán alterarse para conseguir una ejecución satisfactoria.

## 18. Ejecución final

Una vez eliminados los cambios temporales utilizados para los experimentos, deberá obtenerse:

```text
Lint       ✅
Test       ✅
Container  ✅
```

El pull request deberá indicar que se satisfacen los requisitos establecidos por el ruleset y permitir el merge.

Solo entonces deberá integrarse el pull request a `main`.

Después del merge deberá comprobarse que el workflow asociado al `push` sobre `main` también finaliza satisfactoriamente.

## 19. Evidencia técnica

El repositorio deberá contener:

```text
EVIDENCIAS.md
```

Este archivo continuará utilizándose en el Laboratorio 3.

Para este laboratorio deberá contener una sección:

```markdown
# Evidencias

## Laboratorio 2. Integración continua
```

y registrar, como mínimo:

### 19.1 Pipeline final

Descripción breve de:

- los tres jobs;
- qué responsabilidad tiene cada uno;
- cuáles pueden ejecutarse en paralelo;
- de qué jobs depende `Container`.

### 19.2 Fallo de lint

- enlace a la ejecución;
- cambio que provocó el fallo;
- evidencia de que `Container` no se ejecutó;
- evidencia de que el merge quedó bloqueado.

### 19.3 Fallo de pruebas

- enlace a la ejecución;
- cambio que provocó el fallo;
- evidencia de que `Container` no se ejecutó;
- evidencia de que el merge quedó bloqueado.

### 19.4 Ejecución satisfactoria

- enlace a una ejecución con los tres jobs satisfactorios;
- enlace al pull request utilizado para integrar el laboratorio;
- SHA del commit final evaluado.

### 19.5 Artifact

- identificación del workflow artifact generado por el job `Test`.

### 19.6 Caché

- evidencia breve de una ejecución en la que se observe reutilización de la caché de dependencias;
- explicación de qué cambio provocaría que dicha caché tuviera que actualizarse.

No se requiere un informe extenso ni capturas para información que pueda verificarse mediante enlaces directos a GitHub.

## 20. Restricciones

No forma parte de este laboratorio:

- agregar nuevas funcionalidades a la API;
- modificar la lógica de negocio;
- rediseñar el Dockerfile;
- modificar permanentemente las pruebas suministradas;
- activar Firestore;
- configurar Google Cloud;
- publicar imágenes en un registry;
- desplegar la aplicación;
- implementar entrega continua.

Las modificaciones temporales realizadas para los experimentos de fallo deberán eliminarse antes de la entrega.

## 21. Entrega

En Moodle deberá entregarse:

1. enlace al repositorio privado de la pareja;
2. nombres y carnets de ambos integrantes, cuando corresponda al mecanismo de entrega utilizado.

Antes de enviar deberá comprobarse que:

- la cuenta de GitHub **`guillermocalderon2021`** tiene acceso al repositorio privado;
- el repositorio continúa siendo privado;
- `main` contiene la solución final;
- el workflow se ejecuta ante `push` y `pull_request` según lo solicitado;
- existen los jobs `Lint`, `Test` y `Container`;
- el ruleset de `main` está activo;
- los tres checks son obligatorios para merge;
- existe `EVIDENCIAS.md`;
- no quedaron archivos creados únicamente para provocar fallos;
- la última ejecución sobre `main` finaliza satisfactoriamente.

No se requiere crear un tag Git para esta entrega.

## 22. Uso de herramientas de inteligencia artificial

El uso de herramientas de inteligencia artificial está permitido durante el laboratorio.

Estas herramientas pueden utilizarse para consultar sintaxis, interpretar mensajes de error, explorar alternativas o apoyar procesos de diagnóstico.

La pareja es responsable de:

- verificar cualquier configuración incorporada;
- comprender el workflow entregado;
- comprender el efecto de cada dependencia entre jobs;
- poder explicar las evidencias obtenidas;
- distinguir entre una ejecución satisfactoria y una configuración técnicamente correcta.

No se requiere entregar el historial de prompts.

## 23. Rúbrica

| Criterio | Puntos |
|---|---:|
| Eventos `push` y `pull_request` | 10 |
| Job `Lint` | 10 |
| Job `Test`, cobertura y ejecución de pruebas | 15 |
| Paralelización y dependencias entre jobs | 15 |
| Job `Container` e identificación de la imagen por commit | 15 |
| Caché de dependencias | 10 |
| Workflow artifact de cobertura | 5 |
| Permisos del workflow | 5 |
| Ruleset y bloqueo efectivo del merge | 10 |
| Evidencia técnica y entrega | 5 |
| **Total** | **100** |

### 23.1 Eventos `push` y `pull_request` — 10 puntos

Se evaluará que el workflow responda correctamente a los dos tipos de evento solicitados y que los filtros de ramas correspondan al flujo definido para el laboratorio.

### 23.2 Job `Lint` — 10 puntos

Se evaluará que el análisis estático:

- se encuentre en un job independiente;
- utilice correctamente la configuración suministrada;
- provoque una ejecución fallida cuando Ruff detecte una violación.

### 23.3 Job `Test`, cobertura y ejecución de pruebas — 15 puntos

Se evaluará:

- ejecución de la suite suministrada;
- generación de cobertura sobre la aplicación;
- generación del reporte solicitado;
- comportamiento correcto ante una prueba fallida.

### 23.4 Paralelización y dependencias — 15 puntos

Se evaluará que:

- `Lint` y `Test` sean independientes;
- ninguno dependa innecesariamente del otro;
- `Container` dependa de ambos;
- una falla en cualquiera de las validaciones impida continuar hacia la construcción del contenedor.

### 23.5 Job `Container` — 15 puntos

Se evaluará:

- existencia de un job independiente;
- construcción correcta de la imagen;
- identificación de la imagen mediante el commit correspondiente;
- ejecución únicamente cuando las validaciones requeridas hayan finalizado satisfactoriamente.

### 23.6 Caché — 10 puntos

Se evaluará:

- configuración de caché para dependencias Python;
- relación apropiada entre la caché y los archivos de dependencias;
- evidencia de reutilización entre ejecuciones.

### 23.7 Workflow artifact — 5 puntos

Se evaluará que el reporte de cobertura generado por `Test` quede disponible como artifact de la ejecución.

### 23.8 Permisos — 5 puntos

Se evaluará que los permisos concedidos al workflow sean explícitos y se ajusten al principio de mínimo privilegio.

### 23.9 Ruleset y bloqueo del merge — 10 puntos

Se evaluará:

- ruleset activo sobre `main`;
- pull request obligatorio;
- `Lint`, `Test` y `Container` configurados como checks requeridos;
- evidencia de que un PR con validaciones fallidas no puede integrarse;
- evidencia de que el merge queda habilitado cuando las validaciones requeridas son satisfactorias.

### 23.10 Evidencia técnica y entrega — 5 puntos

Se evaluará:

- claridad y suficiencia de `EVIDENCIAS.md`;
- enlaces verificables a ejecuciones y pull request;
- correspondencia entre el SHA entregado y el estado final evaluado;
- acceso del docente al repositorio;
- estado reproducible de `main`.


## 24. Referencias técnicas

- GitHub Actions: <https://docs.github.com/actions>
- Sintaxis de workflows: <https://docs.github.com/actions/writing-workflows/workflow-syntax-for-github-actions>
- Caché con `setup-python`: <https://github.com/actions/setup-python>
- Workflow artifacts: <https://docs.github.com/actions/using-workflows/storing-workflow-data-as-artifacts>
- Rulesets: <https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets>
- Reglas disponibles para rulesets: <https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets>
- GitHub Student Developer Pack: <https://education.github.com/pack/>
