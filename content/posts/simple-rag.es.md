---
title: "RAG Sencillo, Parte 1"
date: 2026-05-06T00:00:00-07:00
summary: "Una mirada de alto nivel a Retrieval Augmented Generation... por qué lo necesitamos, qué es, y cómo se ve la arquitectura. Primera parte de una serie."
tags: ["rag", "llm", "ia"]
categories: ["ia"]
series: ["Construyendo RAG"]
---

## ¿Qué pasa cuando un LLM no sabe cómo responder tu consulta?

Supongamos que trabajas en una industria regulada, como Medicina. Partiendo de esa premisa, es muy importante que el resultado de una consulta a un sistema utilizando un LLM (Modelo de Lenguaje Grande) tenga los detalles específicos del dominio en su contexto. En otras palabras, queremos aumentar el prompt en sí. ¿Y por qué? Pues los LLMs son entrenados con un corpus finito y con una fecha de corte. Si los requisitos del sistema dependen de información privada (récords médicos, documentos internos, historiales de chat internos, documentos legales, etc.), el modelo simplemente nunca la vio y la calidad de la respuesta comienza a degradar. Preguntarle al sistema cuál es el protocolo de tratamiento interno de tu hospital para cierta condición, o qué dice el historial clínico de un paciente en particular, va a resultar en una respuesta incompleta o incorrecta.

Por eso es que preguntar cuál es la capital de Colombia nos da una respuesta correcta.

![Pregunta a un LLM, cuál es la capital de Colombia, respuesta correcta... Bogotá](/posts/simple-rag/consulta_capital_colombia.png)

Pero preguntar cómo se llamaba mi primera mascota no tanto.

![Pregunta a un LLM, cómo se llamaba mi primera mascota, el modelo no sabe](/posts/simple-rag/consulta_primera_mascota.png)

Pero no estamos limitados a los datos de entrenamiento, hay varias técnicas para mejorar la calidad de respuesta. Una de ellas, y el tema principal de esta serie, es RAG (Retrieval Augmented Generation o Generación Aumentada por Recuperación). Elaboraré específicamente sobre naive RAG (RAG ingenuo). Es la versión más básica, sin re-ranking, sin reescritura de consulta, sin filtros adicionales. Buena base para entender la mecánica antes de meterle complejidad.

## ¿Qué es RAG (Generación Aumentada por Recuperación)?

Es una técnica donde consultamos una fuente externa (típicamente una base de datos vectorial, aunque también se ven implementaciones híbridas con búsqueda léxica) para inyectar fragmentos relevantes al contexto del prompt. Es una operación opaca para el LLM. Es decir, no sabe de los detalles de implementación del RAG, sencillamente recibe la consulta original del usuario más la información obtenida de la búsqueda. 

¿Y por qué una fuente de almacenamiento vectorial y no texto plano o de pronto una base de datos relacional? La razón es que queremos buscar por significado, no por coincidencia de palabras. Sí, uno podría hacer una búsqueda de palabra clave (keyword search), donde se buscan los términos exactos de la consulta en el corpus y se rankean los resultados... pero esa técnica no toma en cuenta intención, contexto ni significado. Sencillamente dice cuáles registros contienen los términos. La búsqueda semántica funciona distinto. Primero convertimos los fragmentos en vectores numéricos (embeddings) y después buscamos los vectores más cercanos a la consulta en ese espacio. La magia es que dos textos pueden quedar cerca aunque no compartan ni una palabra (por ejemplo *anular el contrato* y *cancelar el acuerdo*).

Es importante resaltar que tenemos que preparar la fuente de datos antes de pasar por el proceso de vector embedding. Hay dos cosas que hacer aquí. Por un lado limpieza. Por otro, cuando los datos vienen estructurados, construir documentos en lenguaje natural a partir de ellos. Los embeddings rinden mucho mejor con texto coherente que con registros crudos de una base de datos o filas de un CSV. Así que uno coge cada registro y lo acomoda de tal manera que terminemos con un bloque de texto estándar y ahí se crea el documento en sí.

Bueno, suficientemente sencillo con algo como un catálogo de películas en el cual cada registro solo maneja un título, director, género, año, rating, etc... terminaríamos con un documento relativamente pequeño. Pero, ¿qué pasaría si estamos hablando de algo más grande, como regulaciones o constitución de ley? Aquí el problema principal no es el tamaño en sí. Es que cuando metemos un documento gigante en un solo vector, ese vector termina mezclando muchos temas a la vez y la búsqueda pierde precisión. No podemos decir "tráeme la parte que habla de X" porque el documento entero es un solo punto en el espacio vectorial. Aparte de eso hay límites prácticos. El modelo de embedding tiene un máximo de tokens que acepta de entrada (típicamente algunos miles), y cuando inyectamos los chunks recuperados al prompt también gastamos del contexto del LLM.

Así que chunking o fragmentación alivia este problema. El documento se divide en piezas más manejables. Hay varias estrategias. La más sencilla parte el texto en un tamaño fijo. Otras toman en cuenta límites naturales como oración, párrafo, semántica o estructura. Y casi siempre se deja que los fragmentos se monten un poquito entre sí (un 10 o 20%) para que no te quede ninguna idea partida a la mitad.

Ya teniendo los fragmentos el sistema puede llamar un API de embedding que básicamente transforma el fragmento y genera un vector (una lista de números e.g. [0.23, -0.7743,... ]), típicamente dimensiones de 384 a 3,072. La cercanía entre vectores se mide con similitud coseno (u otra métrica equivalente), y así el sistema encuentra contenido relacionado por significado, no por coincidencia textual.

Ya teniendo los embeddings vectoriales, necesitamos almacenarlos en un lugar donde podamos hacer una consulta vectorial y recuperar los top-k vectores que semánticamente tienen relación con la consulta. Por lo general se almacena el vector, el fragmento original (representado como texto) y metadata. Ya teniendo todo indexado, uno puede procesar una consulta de parte del usuario en lenguaje natural, convertir esa consulta en un vector utilizando el mismo modelo que utilizamos en la fase de indexación e inyectar los resultados como contexto en el prompt que pasamos al LLM. 

Desglosemos el acrónimo.

- **Recuperación** -> traer información relacionada a la consulta
- **Aumento** -> agregar ese contexto recuperado al prompt
- **Generación** -> el LLM produciendo un resultado con el prompt ya aumentado

## La arquitectura del sistema

```mermaid
flowchart TB
    subgraph Indexacion["Fase de Indexación"]
        A[Fuente de Datos<br/>API, archivos] --> B[Limpiar]
        B --> C[Chunking<br/>fragmentar]
        C --> D[Embedding<br/>vectorizar]
        D --> E[(Vector Store)]
    end

    subgraph Consulta["Fase de Consulta"]
        U[Usuario<br/>pregunta] --> S[Orquestador]
        S --> Q[Vectorizar consulta]
        Q --> R[Buscar similitud]
        E -.recupera.-> R
        R --> P[Construir Prompt<br/>contexto + query]
        P --> L[Llamar LLM<br/>API]
        L --> Resp[Respuesta al Usuario]
    end
```

Hay dos fases prácticas. En la fase de **indexación** preparamos los datos. Tomamos la fuente, la limpiamos, la fragmentamos en chunks, generamos embeddings (vectores) y los guardamos en una base de datos vectorial. La fase de **consulta** ocurre cuando un usuario hace una pregunta. La convertimos en embedding, buscamos los chunks más similares en el almacenamiento, los inyectamos al prompt junto con la consulta original, y se lo pasamos al LLM.

La literatura y el material educativo suelen descomponer esto un poco más, en tres etapas canónicas... indexación, recuperación y generación. Es la misma idea, solo que la fase de consulta se parte en dos.

```mermaid
flowchart TB
    subgraph Indexing["1. Indexación"]
        direction LR
        Docs[Documentos] --> Load[Cargar]
        Load --> Split[Fragmentar<br/>en chunks]
        Split --> Embed1[Generar<br/>embeddings]
        Embed1 --> VS[(Vector Store)]
    end

    subgraph Retrieval["2. Recuperación"]
        direction LR
        Q[Consulta<br/>del usuario] --> Embed2[Generar<br/>embedding]
        Embed2 --> Search[Búsqueda<br/>por similitud]
        Search --> Topk[Top-k chunks<br/>relevantes]
    end

    subgraph Generation["3. Generación"]
        direction LR
        Prompt[Prompt aumentado<br/>consulta + contexto] --> LLM[LLM]
        LLM --> Answer[Respuesta]
    end

    VS -.-> Search
    Q --> Prompt
    Topk --> Prompt
```

- **Indexación** es lo mismo que arriba. Cargamos documentos, los fragmentamos, los vectorizamos y los guardamos. Se hace una sola vez (o periódicamente cuando los datos cambian).
- **Recuperación** es la búsqueda en sí. Tomamos la consulta del usuario, la convertimos en embedding y traemos los chunks más relevantes del almacenamiento vectorial.
- **Generación** es donde el LLM entra en escena. Aquí ocurre el *aumento* que mencionamos antes. Construimos el prompt aumentado (consulta original + contexto recuperado) y el modelo produce la respuesta.

## Qué sigue

En la siguiente parte voy a armar un RAG sencillo usando hallazgos de CVE como fuente de datos.
