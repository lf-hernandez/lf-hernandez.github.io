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
> - [Parte 1 — ¿Qué es RAG y cómo funciona?](/posts/simple-rag/)
> - **Parte 2** — Implementación práctica con CVEs *(este artículo)*

# Meta

La meta de este artículo es demostrar un sistema RAG funcional utilizando CVEs como fuente de datos. ¿Por qué? CVE (Common Vulnerabilities and Exposures) es un estándar de industria internacional, un diccionario que cataloga vulnerabilidades de ciberseguridad de software y hardware y nos permite explorar un caso de uso donde la precisión importa. CVEs se descubren y se publican en un promedio de 130 por día en 2026. Así que es casi garantizado que cualquier modelo no tendrá el catálogo al día, así que podemos aumentar el contexto con nuestro sistema RAG.

La meta en esta parte es crear la fase de indexación. Como elaboré en la primera parte, la fase de indexación pasa offline en el sentido de que no requiere que indexemos la fuente de datos en cada solicitud/consulta.

La fuente de datos la vamos a obtener de Kaggle. Una plataforma para todo lo relacionado con la ciencia de datos y ML (Machine Learning). La vamos a utilizar principalmente para obtener la fuente de datos.

Vamos a comenzar con una sencilla y popular, [CVE (Common Vulnerabilities and Exposures)](https://www.kaggle.com/datasets/andrewkronser/cve-common-vulnerabilities-and-exposures), que está un poco desactualizada (solo tiene datos hasta el 2019) pero nos va a servir para establecer una base en el concepto. Pueden obtener la fuente manualmente via la web y descargando el archivo zip o creando una cuenta de Kaggle, descargando el cliente Kaggle CLI y hacerlo programáticamente. De igual manera, solo vamos a trabajar con el archivo `cve.csv`. Haremos una modificación pequeña, la primera columna de la primera hilera está en blanco. Pero observando el contenido sabemos que es el identificador del CVE, entonces sencillamente le daremos el nombre `id` para poder consumir y analizar el documento programáticamente. El resto de las columnas que vamos a utilizar (`cwe_name`, `summary`, `cvss`) ya vienen nombradas en el archivo original.

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
## Almacenamiento Vectorial

Vamos a utilizar ChromaDB que es una base de datos de código abierto (Open Source) y liviana. Nos permitirá crear un índice de búsqueda semántica para texto, imágenes o audio para aplicaciones RAG.

Conceptualmente nos enfocaremos en un solo primitivo, Collections, que establece una colección de embeddings.

Iniciaremos el cliente persistente primero:

```python
# indexacion.py
import chromadb
import pandas as pd
from sentence_transformers import SentenceTransformer

client = chromadb.PersistentClient(path="./chroma")

```

ChromaDB ofrece 2 otros tipos de clientes, cloud y in-memory. Cloud requiere una cuenta Chroma Cloud y para nuestro uso de aprendizaje e experimentación es un poco excesivo. In-memory crea un servidor efímero, es útil para experimentación o con un Jupyter Notebook pero para nuestro uso queremos un cliente persistente que almacena la base de datos localmente (chroma/, en nuestro caso).

## Creando una Colección

Ya teniendo nuestro cliente en su lugar, procedemos a crear nuestra colección que contendrá nuestros embeddings de CVEs.

```python
collection = client.get_or_create_collection(name="CVEs")
```

## Procesando y Creando los Embeddings

Podríamos utilizar la API de OpenAI para crear nuestros embeddings, pero en este caso vamos a utilizar algo un poco más sencillo, Sentence Transformers. Es un framework que se especializa en embeddings de oración, texto e imágenes. Normalmente se recomienda utilizar OpenAI Embeddings API, pero para salvarnos el costo, me parece que sentence-transformers cumple su función en nuestro caso, y nos da un poco más de flexibilidad.

```python
model = SentenceTransformer("all-MiniLM-L6-v2")
```

Vamos a utilizar el modelo pre-entrenado, all-MiniLM-L6-v2, ya que nos ofrece un buen balance entre calidad y velocidad.

## Funciones Auxiliares

Antes de seguir a la ingestión de la fuente de datos vamos a crear unas funciones auxiliares para poder procesar nuestros registros de CVEs. Algo que se puede observar muy claramente es que no tenemos un nivel de severidad. Si, existe un CVSS que nos da el valor numeral pero si queremos hacer una consulta en lenguaje natural sería un poco más rápido tener la asignación a mano:

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

Y también queremos normalizar los fragmentos para poder crear el embedding con más precisión relativa a la información que queremos.

```python
def normalize(cve: pd.Series) -> str:
    return f"CVE ID: {cve['id']}\nCWE Name: {cve['cwe_name']}\nSummary: {cve['summary']}\nCVSS: {cve['cvss']}"
```

Como pueden ver, creamos cada fragmento a partir de 4 atributos: El identificador, nombre canónico de la vulnerabilidad, un resumen de qué trata y el puntaje de CVSS.

## Ingeriendo la fuente de datos

Bueno ya tenemos lo básico para poder ingerir los CVEs, vamos a utilizar Pandas (una librería para poder manejar y analizar datos, nos ayudará específicamente para manejar el archivo CSV).

```python
def ingest(path: str):
    df = pd.read_csv(path)

    documents = []
    ids = []
    metadatas = []

    for _, row in df.iterrows():
        documents.append(normalize(row))
        ids.append(str(row["id"]))
        metadatas.append({"severity": get_severity(row["cvss"])})

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
1. leemos la fuente de datos, cargándola en un dataframe en memoria para extraer y manipular la data más fácilmente
2. Creamos listas para almacenar nuestros documentos, identificadores de cada CVE y la metadata.
3. Iteramos por cada hilera del data frame, ignorando el índice y manejando la hilera en sí
4. En cada iteración vamos a normalizar la data y añadir el contenido normalizado a nuestra lista de documentos, añadir el identificador a su lista y añadir la severidad resuelta a la lista de metadata
5. Ya teniendo nuestras listas en orden vamos a crear los embeddings de todos nuestros documentos y resolverlos en una lista de embeddings
6. ChromaDB recomienda insertar en batches en lugar de un solo `add` con todos los registros. El tercer argumento de `range` es el paso, así que `range(0, len(ids), 5000)` produce los índices `0, 5000, 10000, ...`. En cada iteración, los slices `ids[i:i+batch_size]` toman los próximos 5.000 elementos. Con más de 93.000 registros, eso resulta en unas 19 llamadas a `collection.add`, donde la última toma los registros restantes (menos de 5.000)

### Corriendo nuestro módulo de indexación

Ya tenemos la primera iteración de nuestro módulo de indexación listo, así que solo falta correrlo para probar que el resultado esté válido ante nuestra expectativa:

```python
# main.py
from indexacion import ingest

ingest("./data/cve.csv")
print("Proceso completado")
```
