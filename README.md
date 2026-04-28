# ProyectoFinalNoSQL

[Documentación](https://github.com/bluesky-social/jetstream/blob/main/README.md)

## Origen y propósito del stream de datos

### Resumen

Jetstream es un servicio de streaming de datos y convierte datos de un servicio ATProto en un JSON amigable y ligero. Hay 4 instancias públicas, dos para US-EAST y dos para US-WEST.

Hay tres eventos (hasta el momento) que se pueden utilizar con Jetstream:

1. commit: Para observar creaciones, actualizaciones y eliminaciones de publicaciones.
2. identity: Actualizar el DID de un usuario.
3. account: Indica un cambio en el estado de una cuenta

Ejemplo de commit de me gusta en el repo:

```json
{
  "did": "did:plc:eygmaihciaxprqvxpfvl6flk",
  "time_us": 1725911162329308,
  "kind": "commit",
  "commit": {
    "rev": "3l3qo2vutsw2b",
    "operation": "create",
    "collection": "app.bsky.feed.like",
    "rkey": "3l3qo2vuowo2b",
    "record": {
      "$type": "app.bsky.feed.like",
      "createdAt": "2024-09-09T19:46:02.102Z",
      "subject": {
        "cid": "bafyreidc6sydkkbchcyg62v77wbhzvb2mvytlmsychqgwf2xojjtirmzj4",
        "uri": "at://did:plc:wa7b35aakoll7hugkrjtf3xf/app.bsky.feed.post/3l3pte3p2e325"
      }
    },
    "cid": "bafyreidwaivazkwu67xztlmuobx35hs2lnfh3kolmgfmucldvhd3sgzcqi"
  }
}
```

### Origen y Auditoría

El servicio es operado y mantenido por Bluesky Social, PBC. Para más información, visitar su README oficial: [Bluesky Social](https://github.com/bluesky-social/jetstream/blob/main/README.md)

### Diccionario de datos

Atributo - Definición Técnica - Tipo de dato
did | Identificador descentralizado del repositorio del usuario. | Cualitativo (ID)
time_us | Marca de tiempo en microsegundos UNIX del evento. | Serie temporal
kind | Tipo de evento (commit, identity, account). | Cualitativo
commit | Objeto que contiene los detalles del evento. | Objeto
commit.operation | Acción realizada (create, update, delete). | Cualitativo
commit.collection | NSID de la colección (ej. app.bsky.feed.post). | Cualitativo
commit.rkey | Clave del registro dentro de la colección. | Cualitativo
commit.record | Registro del evento. | Objeto
commit.record.text | Contenido de texto del post (si aplica). | Texto estructurado
commit.record.createdAt | Fecha de creación del registro en formato ISO 8601. | Serie temporal
commit.record.subject.uri | Referencia al post original (en caso de likes/reposts). | Cualitativo (URI)
seq | Número de secuencia para ordenamiento de eventos (en identity/account) | Cuantitativo

### Variables cuantitativas

time_us (como valor numérico de microsegundos), seq (número de secuencia)

### Variables cualitativas

kind, operation, collection, y el estado active en eventos de cuenta

### Texto no estructurado

El campo record.text que contiene el microblogueo puro de los usuarios.

### Series temporales

time_us y record.createdAt

### Consideraciones éticas

El procesamiento de datos provenientes de redes sociales en tiempo real a través de Jetstream implica diversos retos éticos y técnicos que deben ser gestionados para garantizar la integridad y el respeto a la privacidad de los usuarios.

#### Riesgos de sesgo (Bias)

- Sesgo demográfico: Los datos de Bluesky no representan a la población global, sino a un subconjunto de usuarios que suelen ser "early adopters" de tecnología o perfiles con inclinaciones técnicas, lo que puede sesgar las conclusiones analíticas.

- Sesgo de actividad: El flujo de datos está influenciado por la presencia de cuentas automatizadas (bots) y usuarios altamente activos, lo que puede opacar las tendencias de los usuarios promedio en la capa de procesamiento analítico.

#### Manejo de datos sensibles

- Identificación de usuarios: Aunque los DIDs (Identificadores Descentralizados) son públicos, su almacenamiento masivo permite realizar perfiles de usuario detallados, lo cual es sensible bajo normativas de privacidad.

- Metadatos de comportamiento: El uso del atributo time_us permite deducir patrones de actividad diaria, zonas horarias y hábitos de publicación que, al ser analizados en conjunto, revelan información privada sobre el comportamiento humano.

#### Dilemas éticos y manejo de la información

- Derecho al olvido: Jetstream emite eventos de tipo delete cuando un usuario elimina un registro o su cuenta. Un dilema crítico es si nuestra arquitectura de datos debe conservar estos registros en la capa analítica o reflejar la eliminación de inmediato para respetar la voluntad del usuario.

- Uso de la información: Dado que el proyecto busca transformar "eventos crudos" en "información estratégica", existe la responsabilidad ética de no utilizar estos datos para fines de vigilancia o manipulación de opinión pública, limitándose estrictamente al análisis académico y de negocio propuesto.

## Implementación de la capa de ingesta

Para la captura y almacenamiento de datos del stream, se desarrolló un consumidor asíncrono en Python que actúa como puente entre el WebSocket público y nuestro clúster distribuido.

### Estado de Verdad Operativa

Se seleccionó **Apache Cassandra** como el stage de verdad operativa debido a su arquitectura orientada a escrituras últra rápidas. Se diseñó una tabla optimizada para esto:

```sql
CREATE TABLE IF NOT EXISTS jetstream_data.events (
    collection text,
    time_us bigint,
    did text,
    kind text,
    operation text,
    rkey text,
    record text,
    PRIMARY KEY ((collection), time_us, did)
) WITH CLUSTERING ORDER BY (time_us DESC);
```

La llave de partición por `collection` garantiza una distribución equitativa de la carga de escritura entre los nodos del clúster, ya que los eventos de cada tipo de colección se distribuyen aleatoriamente entre los nodos. Mientras que el ordenamiento por `time_us DESC` permite consultas puntuales sobre el estado actual de los eventos (ej. "los últimos 10 follows") con latencia mínima. Para preservar el polimorfismo de los eventos de Bluesky, el payload completo se almacena como un texto crudo JSON en la columna `record`, delegando su transformación a la capa analítica.

### Garantía de Caudal y Pruebas de carga

Para soportar el volumen de entrada sin pérdida de mensajes ni saturación de red, se implementó una estrategia de **Micro-Batching** y **Concurrencia**:

1. Control de Flujo: Para mantener una carga estable y realista en un entorno local distribuido, el consumidor se suscribió específicamente a los eventos `app.bsky.graph.follow` y `app.bsky.graph.block`, garantizando un caudal promedio de ~25 eventos por segundo (cumpliendo y superando el requerimiento mínimo del proyecto).
2. Inserción Concurrente: El script no inserta registro por registro, lo cual sería un cuello de botella. Acumula los eventos en memoria y utiliza el método `execute_concurrent` del driver de DataStax para inyectar bloques (batches) directamente a los nodos de Cassandra de manera no bloqueante.

### Resiliencia y Alta Disponibilidad

El sistema está diseñado para tolerar fallos en dos frentes principales:

- **Resiliencia en la Red (Consumidor):** El script asíncrono incluye un mecanismo de Exponential Backoff (`asyncio.sleep`). Si se pierde la conexión con el WebSocket de Jetstream o hay un micro-corte en la VPN, el sistema intercepta la excepción, espera un intervalo creciente de segundos y se reconecta automáticamente sin detener la ejecución ni perder el hilo del event loop.

- **Resiliencia en el Almacenamiento (Base de Datos):** El clúster de Cassandra consta de 3 nodos desplegados con un Replication Factor de 3 (`NetworkTopologyStrategy`). Basado en el Teorema CAP (priorizando AP), si uno de los nodos de la VPN cae, el clúster sigue aceptando todas las escrituras sin pérdida de información (Eventual Consistency).

### Ejecución y Seguridad

Cumpliendo con las directrices de control de accesos, las credenciales del clúster y los endpoints de la VPN no se exponen en el código fuente. Se inyectan en tiempo de ejecución mediante la librería `python-dotenv`.

**Para ejecutar el consumidor:**

1. Crear un archivo `.env` en la raíz basado en el `.env.example`
2. Activar el entorno virtual e instalar dependencias (`pip install -r requirements.txt`)
3. Ejecutar el orquestador `python proyecto_ingesta.ipynb` (o desde el notebook proporcionado).
