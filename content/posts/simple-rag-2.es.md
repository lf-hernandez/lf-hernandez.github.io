---
title: "RAG Sencillo: Parte 2"
date: 2026-05-08T16:07:05-04:00
draft: false
summary: ""
description: ""
tags: []
categories: []
ShowToc: true
TocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowCodeCopyButtons: true
ShowWordCount: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
# canonicalURL: ""
# cover:
#   image: ""
#   alt: ""
#   caption: ""
#   relative: false
#   hidden: false
---

> **En esta serie**
> - [Parte 1: ¿Qué es RAG y cómo funciona?](/posts/simple-rag/)
> - **Parte 2**: Implementación práctica con CVEs *(este artículo)*

## La meta

En la primera parte hablamos del qué y el por qué de RAG. Ahora vamos a ver como es la implementación. Como fuente de datos vamos a usar CVEs (Common Vulnerabilities and Exposures), un estándar internacional que cataloga vulnerabilidades de ciberseguridad de software y hardware. ¿Por qué CVEs? Se publican unos 130 por día en 2026, así que es casi garantizado que cualquier modelo no tenga el catálogo al día. Un caso ideal donde aumentar el contexto con RAG vale la pena.

En esta parte nos enfocamos en la fase de indexación. Como mencioné antes, la indexación pasa offline; no toca correrla en cada consulta.

La fuente la sacamos de Kaggle, una plataforma de datasets y ML (Machine Learning). Vamos a empezar con un dataset sencillo y popular, [CVE (Common Vulnerabilities and Exposures)](https://www.kaggle.com/datasets/andrewkronser/cve-common-vulnerabilities-and-exposures), que está un poco desactualizado (solo llega al 2019) pero sirve para establecer la base. Se puede bajar de dos formas: manualmente desde la web (descargando el zip) o creando una cuenta de Kaggle y usando el cliente Kaggle CLI para hacerlo programáticamente.

Solo vamos a trabajar con el archivo `cve.csv`. Una nota: la primera columna de la primera fila viene en blanco. Observando el contenido sabemos que es el identificador del CVE, así que más adelante le ponemos el nombre `id` programáticamente al cargar el CSV. El resto de columnas que usamos (`cwe_name`, `summary`, `cvss`) ya vienen nombradas.

## Configuración

Iniciamos un proyecto nuevo y preparamos el entorno:

```bash
mkdir rag-sencillo
cd rag-sencillo
python3 -m venv .venv
source .venv/bin/activate
pip install chromadb pandas sentence-transformers
pip freeze > requirements.txt
touch indexacion.py main.py
mkdir data
mv Downloads/cve.csv data/
```

## Almacenamiento vectorial

Vamos a usar ChromaDB, una base de datos de código abierto (Open Source) y liviana, que nos permite crear un índice de búsqueda semántica sobre texto, imágenes o audio para aplicaciones RAG.

Solo nos interesa una primitiva: `Collection`, que básicamente es una agrupación de embeddings.

Arrancamos con el cliente persistente:

```python
# indexacion.py
import chromadb
import pandas as pd
from sentence_transformers import SentenceTransformer

client = chromadb.PersistentClient(path="./chroma")

```

ChromaDB ofrece dos tipos más de clientes: cloud y in-memory. Cloud requiere una cuenta de Chroma Cloud, lo cual es un poco excesivo para experimentar. In-memory crea un servidor efímero, es útil para pruebas rápidas o con un Jupyter Notebook, pero acá queremos persistencia local (en `./chroma/`, en nuestro caso).

## Creando una colección

Ya con el cliente listo, creamos la colección que va a contener los embeddings de los CVEs.

```python
collection = client.get_or_create_collection(name="CVEs")
```

## Procesando y creando los embeddings

Podríamos usar la API de OpenAI para los embeddings, pero acá vamos por algo un poco más sencillo: Sentence Transformers. Es un framework que se especializa en embeddings de oración, texto e imágenes. Normalmente la recomendación es la OpenAI Embeddings API, pero para ahorrarnos el costo me parece que sentence-transformers cumple la función y nos da un poco más de flexibilidad.

```python
model = SentenceTransformer("all-MiniLM-L6-v2")
```

Usamos el modelo pre-entrenado `all-MiniLM-L6-v2`, que ofrece un buen balance entre calidad y velocidad.

## Funciones auxiliares

Antes de seguir con la ingestión, creamos unas funciones auxiliares para procesar los registros de CVEs. Algo que llama la attencion casi de inmediato es que el dataset no incluye nivel de severidad. Sí, existe el CVSS que da el valor numérico, pero si queremos hacer una consulta en lenguaje natural conviene tener la categoría a mano:

```python
def get_severity(cvss: str) -> str:
    try:
        score = float(cvss)
    except (ValueError, TypeError):
        return "Unknown"
    match score:
        case n if n >= 9.0 and n <= 10.0:
            return "Critical"
        case n if n >= 7.0 and n <= 8.9:
            return "High"
        case n if n >= 4.0 and n <= 6.9:
            return "Medium"
        case n if n >= 0.1 and n <= 3.9:
            return "Low"
        case _:
            return "Unknown"
```

Antes de seguir vale la pena hablar de chunking. En este caso no nos toca: cada CVE es un registro corto que cabe cómodamente en un solo embedding. El chunking entra cuando la fuente son documentos largos (regulaciones, manuales, libros), no es nuestro caso.

Lo que sí queremos hacer es construir el documento que va a ir al embedding. Darle al modelo un formato consistente ayuda a que el embedding capture lo importante en vez de ruido. Y un detalle más: solo queremos meter contenido semántico en el embedding. El identificador del CVE es una cadena única que el modelo nunca ha visto, y el CVSS es solo un número... ninguno aporta significado al vector. Esos los dejamos como metadata; al embedding solo le pasamos el nombre canónico de la vulnerabilidad y el resumen.

```python
def build_document(cve: pd.Series) -> str:
    cwe = cve["cwe_name"] if pd.notna(cve["cwe_name"]) else ""
    summary = cve["summary"] if pd.notna(cve["summary"]) else ""
    return f"{cwe}. {summary}".strip(". ")
```

El `pd.notna(...)` es por las filas donde alguno de los dos campos viene vacío. Sin ese chequeo, pandas mete un `NaN` que la f-string convierte en el literal `"nan"`, y terminaríamos embebiendo basura. El identificador, el puntaje de CVSS y la severidad ya resuelta los guardamos como metadata. ChromaDB las usa como filtros en la consulta. Por ejemplo, "tráeme CVEs de severidad `Critical` relacionados con SQL injection".

## Ingiriendo la fuente de datos

Bueno, ya tenemos lo básico para ingerir los CVEs. Vamos a usar Pandas, una librería para analizar datos tabulares, perfecta para el CSV.

```python
def ingest(path: str):
    df = pd.read_csv(path)
    df.rename(columns={df.columns[0]: "id"}, inplace=True)

    documents = []
    ids = []
    metadatas = []

    for _, row in df.iterrows():
        documents.append(build_document(row))
        ids.append(str(row["id"]))
        metadatas.append({
            "cve_id": str(row["id"]),
            "cvss": float(row["cvss"]) if pd.notna(row["cvss"]) else 0.0,
            "severity": get_severity(row["cvss"]),
            "cwe_name": str(row["cwe_name"]) if pd.notna(row["cwe_name"]) else "",
        })

    embeddings = model.encode(documents, show_progress_bar=True).tolist()

    batch_size = 5000
    for i in range(0, len(ids), batch_size):
        collection.add(
            ids=ids[i:i + batch_size],
            documents=documents[i:i + batch_size],
            embeddings=embeddings[i:i + batch_size],
            metadatas=metadatas[i:i + batch_size],
        )

    print("Indexing phase complete")
```

La función es sencilla:

1. Leemos la fuente de datos, cargándola en un dataframe en memoria para extraer y manipular la data más fácilmente.
2. Renombramos la primera columna a `id` (esa que viene en blanco en el CSV original), así no toca tocar el archivo a mano.
3. Creamos listas para almacenar los documentos, los identificadores de cada CVE y la metadata.
4. Iteramos por cada fila del dataframe, ignorando el índice y trabajando con la fila en sí.
5. En cada iteración construimos el documento para el embedding con `build_document`, añadimos el identificador a su lista y armamos el dict de metadata con el ID, el CVSS, la severidad y el nombre canónico (`cwe_name`).
6. Ya con las listas en orden, generamos los embeddings de todos los documentos de una sola pasada.
7. ChromaDB recomienda insertar en batches en vez de un solo `add` con todos los registros. El tercer argumento de `range` es el paso, así que `range(0, len(ids), 5000)` produce los índices `0, 5000, 10000, ...`. En cada iteración, los slices `ids[i:i+batch_size]` toman los próximos 5.000 elementos. Con más de 93.000 registros, eso son unas 19 llamadas a `collection.add`, donde la última se lleva los restantes (menos de 5.000).

## Corriendo el módulo de indexación

Ya tenemos la primera iteración del módulo lista, así que solo falta correrlo para verificar que el resultado salga como esperamos:

```python
# main.py
from indexacion import ingest

ingest("./data/cve.csv")
print("Proceso completado")
```

## Código completo

Para referencia, acá quedan los dos archivos completos.

`indexacion.py`:

```python
import chromadb
import pandas as pd
from sentence_transformers import SentenceTransformer

client = chromadb.PersistentClient(path="./chroma")

collection = client.get_or_create_collection(name="CVEs")

model = SentenceTransformer("all-MiniLM-L6-v2")


def get_severity(cvss: str) -> str:
    try:
        score = float(cvss)
    except (ValueError, TypeError):
        return "Unknown"
    match score:
        case n if n >= 9.0 and n <= 10.0:
            return "Critical"
        case n if n >= 7.0 and n <= 8.9:
            return "High"
        case n if n >= 4.0 and n <= 6.9:
            return "Medium"
        case n if n >= 0.1 and n <= 3.9:
            return "Low"
        case _:
            return "Unknown"


def build_document(cve: pd.Series) -> str:
    cwe = cve["cwe_name"] if pd.notna(cve["cwe_name"]) else ""
    summary = cve["summary"] if pd.notna(cve["summary"]) else ""
    return f"{cwe}. {summary}".strip(". ")


def ingest(path: str):
    df = pd.read_csv(path)
    df.rename(columns={df.columns[0]: "id"}, inplace=True)

    documents = []
    ids = []
    metadatas = []

    for _, row in df.iterrows():
        documents.append(build_document(row))
        ids.append(str(row["id"]))
        metadatas.append({
            "cve_id": str(row["id"]),
            "cvss": float(row["cvss"]) if pd.notna(row["cvss"]) else 0.0,
            "severity": get_severity(row["cvss"]),
            "cwe_name": str(row["cwe_name"]) if pd.notna(row["cwe_name"]) else "",
        })

    embeddings = model.encode(documents, show_progress_bar=True).tolist()

    batch_size = 5000
    for i in range(0, len(ids), batch_size):
        collection.add(
            ids=ids[i:i + batch_size],
            documents=documents[i:i + batch_size],
            embeddings=embeddings[i:i + batch_size],
            metadatas=metadatas[i:i + batch_size],
        )

    print("Indexing phase complete")
```

`main.py`:

```python
from indexacion import ingest

ingest("./data/cve.csv")
print("Proceso completado")
```
