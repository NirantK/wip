# Pinecone to Qdrant Migration

## Introduction

Why migrate from Pinecone to Qdrant?

### Terminology

#### Pinecone Terminology

- Collection: In Pinecone, a collection refers to a static copy or snapshot of the index. It represents a distinct set of vectors with associated metadata. In Qdrant, this concept is analogous to a collection.
- Index: An index in Pinecone serves as the highest-level organizational unit for vector data. It accepts and stores vectors, provides query services, and performs various vector operations. Each index in Pinecone is deployed on one or more pods. In Qdrant, the concept of an index is similar, representing a logical container for vectors.
- Pods: Pods in Pinecone are pre-configured units of hardware that host and execute Pinecone services. An index in Pinecone runs on one or more pods, where more pods generally lead to increased storage capacity, reduced latency, and improved throughput. In Qdrant, the equivalent of pods is TK (To Kome Later).

#### Qdrant Terminology

- Collection: In Qdrant, a collection is a named set of points, which are vectors with associated payloads. Similar to Pinecone, vectors within the same collection in Qdrant must have the same dimensionality and are compared using a single metric. This aligns with the concept of a collection in Pinecone.
- Payload: One of the notable features of Qdrant is the ability to store additional information along with vectors. This extra information is referred to as the payload in Qdrant terminology. This aligns with the concept of associated metadata in Pinecone.
- Points: In Qdrant, the central entity that the system operates on is a point. A point is a record consisting of a vector and an optional payload. This closely corresponds to the concept of vectors in Pinecone, where each vector is associated with relevant metadata.

| Vector Database  | Pinecone                   | Qdrant                                                                                       |
| ---------------- | -------------------------- | -------------------------------------------------------------------------------------------- |
| DB Capacity/Perf | Pod                        | Cluster                                                                                      |
| Collection       | Is a snapshot of the Index | Not Separate                                                                                 |
| Index            | Index                      | Collection                                                                                   |
| Vector           | Vector                     | Point                                                                                        |
| Metadata         | Metadata                   | Payload                                                                                      |
| Namespace        | Namespace(1 NS per vector) | None. But can do indexing                                                                    |
| Size             | 40kb metadata limit        | No default. Can be set up in collection creation/updation for vectors. Nothing about payload |

## Planning your migration

### Assess your current Pinecone environment

### Evaluating Current Pinecone Setup

### Mapping Data Structures Between Pinecone and Qdrant

## Initial Configuration

Firstly, you will need to install some clients for Pinecone and Qdrant. In this guide, I assume that to be Python:

```bash
pip install pinecone-io
pip install qdrant-client
```

## Moving both Data & Embedding

You can use the fetch operation in Pinecone to retrieve vectors and metadata from your existing Pinecone index. This operation looks up and returns vectors, by id, from an index. The returned vectors include the vector data and/or metadata. The typical fetch latency is under 5ms ([source](https://docs.pinecone.io/docs/manage-data)).

In Qdrant, a collection is a named set of points (vectors with a payload) among which you can search. Vectors within the same collection must have the same dimensionality and be compared by a single metric ([source](https://qdrant.tech/documentation/concepts/collections/)).

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

client = QdrantClient(host="localhost", port=6333)
client.recreate_collection(
    collection_name="my_collection",
    vectors_config=VectorParams(size=100, distance=Distance.COSINE),
)
```

When migrating from Pinecone to Qdrant, the data transfer process will involve exporting data from Pinecone and importing that data into Qdrant.

### Exporting Data from Pinecone

Pinecone provides a `Fetch` operation to look up and return vectors, by id, from an index. This operation returns the vector data and/or metadata. Here is an example which creates random vectors, upserts them into Pinecone and fetches them again.

```python
import os
import pinecone
import numpy as np

# Instantiate the Pinecone client
pinecone.init(api_key=os.environ["PINECONE_API_KEY"])

# Create a new index
index_name = "example-index"
pinecone.deindex(index_name)  # Ensure the index is not already present
pinecone.create_index(index_name)

# Instantiate an Index object for interacting with the specific index
index = pinecone.Index(index_name=index_name)

# Generate some example vectors
num_vectors = 1000  # This should not exceed 1000 due to Pinecone's limit
vector_dimension = 300
ids = [str(i) for i in range(num_vectors)]
vectors = np.random.rand(num_vectors, vector_dimension)

# Upsert vectors
index.upsert(ids=ids, vectors=vectors)

# Fetch vectors
fetched_vectors = index.fetch(ids=ids)

# You can print out the fetched vectors for confirmation
for id, vector in zip(ids, fetched_vectors.values):
    print(f"ID: {id}, Vector: {vector}")

# Remember to delete the index once you're done with it
pinecone.deindex(index_name)

# Deinitialize the Pinecone client
pinecone.deinit()
```

Please replace "PINECONE_API_KEY" with your actual Pinecone API key. This script creates an index, upserts 1000 vectors (the maximum allowed by Pinecone in a single request), then fetches those vectors.

### Extracting Data from Pinecone – Pinecone Export Limitations

When extracting data from Pinecone, it is important to be aware of certain limitations:

1. **Queries**: The maximum value for `top_k`, the number of results to return, is 10,000. However, if the `include_metadata=True` or `include_data=True` is used, the maximum value for `top_k` is 1,000

2. **Fetch and Delete**: The maximum number of vectors per fetch or delete request is 1,000.

3. **Metadata**: The maximum metadata size per vector is 40 KB. Null metadata values are not supported, and metadata with high cardinality can cause the pods to become full

### Importing Data into Qdrant

Qdrant requires the data in float format. The data can be imported into Qdrant by creating a collection and then indexing the data. This process involves the following steps:

1. **Creating a Collection**: A collection in Qdrant is a named set of points (vectors with a payload) among which you can search. Vectors within the same collection must have the same dimensionality and be compared by a single metric.

> You can create a collection by sending a PUT request to `/collections/{collection_name}` with the necessary parameters including the name of the collection and the vectors' size and distance.

2. **Indexing Data**: Data can be indexed into the created collection. An example approach is to implement a class, such as `SearchClient`, with methods for data conversion to necessary formats, indexing, and searching. The `index` method in this class would prepare the data in the required format and then use the `upsert` function of the Qdrant client to index the data .
   Test your applications with the migrated data
   For both Pinecone and Qdrant, you can test your applications with the migrated data by interacting with their respective Python client libraries.

## Testing your applications with the migrated data

For Qdrant, you can test your applications by:

1. **Creating a Collection**: This is done by the recreate_collection method, which creates a collection with specified parameters like vector size and distance metric​3​.

2. **Inserting Vectors**: Use the upsert method to insert vectors into your collection. The vectors can be inserted with associated metadata, known as payload in Qdrant terminology​​.

3. **Querying Vectors**: Use the search method to find similar vectors within the collection. Qdrant also allows for more complex queries with filters based on vector payloads​​.

Remember, these are ideas that you can start with. Your actual testing scope may be larger based on the specifics of your application.

## Performance Considerations

### Cost and Network Latency

One of the primary considerations when choosing a vector database system is cost and network latency.

Pinecone recommends operating within known limits and deploying your application in the same region as your Pinecone service for optimal performance. For users on the Free plan, Pinecone runs in GCP US-West (Oregon).

On the other hand, Qdrant is an open-source vector database that can be self-hosted, providing more flexibility. It can be deployed in your own environment, which can help optimize network latency and potentially lower costs.

### Throughput and Speed

Pinecone suggests increasing the number of replicas for your index to increase throughput (QPS). It also provides a gRPC client that can offer higher upsert speeds for multi-pod indexes. However, be aware that reads and writes cannot be performed in parallel in Pinecone, which means that writing in large batches might affect query latency

Qdrant, on the other hand, offers different performance profiles [2](https://qdrant.tech/documentation/tutorials/how-to/) based on your specific use case:

1. **Low memory footprint with high speed search**: Qdrant achieves this by keeping vectors on disk and minimizing the number of disk reads. You can make this even better by configuring in-memory quantization, with on-disk original vectors.

Vector quantization can be used to achieve this by converting vectors into a more compact representation, which can be stored in memory and used for search.

Below is an example of how to configure this in Python:

```python
from qdrant_client import QdrantClient
from qdrant_client.http import models

client = QdrantClient("localhost", port=6333)

client.recreate_collection(
    collection_name="{collection_name}",
    vectors_config=models.VectorParams(size=768, distance=models.Distance.COSINE),
    optimizers_config=models.OptimizersConfigDiff(memmap_threshold=20000),
    quantization_config=models.ScalarQuantization(
        scalar=models.ScalarQuantizationConfig(
            type=models.ScalarType.INT8,
            always_ram=True,
        ),
    ),
)
```

2. **High precision with low memory footprint**: For scenarios where high precision is required but RAM is limited, you can enable on-disk vectors and HNSW index. Here is an example of how to configure this in Python:

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("localhost", port=6333)

client.recreate_collection(
    collection_name="{collection_name}",
    vectors_config=models.VectorParams(size=768, distance=models.Distance.COSINE),
    optimizers_config=models.OptimizersConfigDiff(memmap_threshold=20000),
    hnsw_config=models.HnswConfigDiff(on_disk=True)
)
```

3. **High precision with high speed search**: If you want high speed and high precision search, Qdrant can achieve this by keeping as much data in RAM as possible and applying quantization with re-scoring. Here is an example of how to configure this in Python:

```python
from qdrant_client import QdrantClient
from qdrant_client.http import models

client = QdrantClient("localhost", port=6333)

client.recreate_collection(
    collection_name="{collection_name}",
    vectors_config=models.VectorParams(size=768, distance=models.Distance.COSINE),
    optimizers_config=models.OptimizersConfigDiff(memmap_threshold=20000),
    quantization_config=models.ScalarQuantization(
        scalar=models.ScalarQuantizationConfig(
            type=models.ScalarType.INT8,
            always_ram=True,
        ),
    ),
)
```

There are also search/query time changes you can make to influence Qdrant's performance characteristics:

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("localhost", port=6333)

client.search(
    collection_name="{collection_name}",
    search_params=models.SearchParams(
        hnsw_ef=128,
        exact=False
    ),
    query_vector=[0.2, 0.1, 0.9, 0.7],
    limit=3,
)
```

Overall, Qdrant's flexibility allows for a wide range of performance tuning options to suit various use cases, which could make it a better option for those looking for a customizable vector database.

Please note that when tuning for performance, you must consider your specific use case, infrastructure, and workload characteristics. The suggested configurations are starting points and may need to be adjusted based on actual performance observations and requirements.

## Pinecone

Here are some tips for getting the best performance out of Pinecone[28](https://www.pinecone.io/docs/).

### Security Considerations

**API Key Management**: If you're using Qdrant Cloud, it also uses API keys for authentication. Ensure these keys are securely managed.

**Data Security**: Test for data security in Qdrant by verifying that the data is not accessible without proper authentication.

**Data Isolation**: In multi-tenant environments, ensure that data from one tenant is not accessible to another.

## Troubleshooting common migration issues

### How to troubleshoot migration issues

## [Optional] Rollout and Backup Strategy

### Phased Rollout Plan
