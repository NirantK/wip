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

Here are some tips for getting the best performance out of [Pinecone](https://www.pinecone.io/docs/).

## Security Considerations

### Qdrant

Qdrant takes a unique approach to security that is minimalist yet flexible, allowing for robust security configurations tailored to each unique deployment environment:

1. **Environment-level Security**: Qdrant is designed to be deployed in secure environments, such as Kubernetes with Network Policies or a Virtual Private Cloud (VPC). This approach puts the control of security aspects in the hands of the deployer, allowing for custom security measures tailored to the specific needs of the deployment environment. If you're using Qdrant Cloud, it also uses API keys for authentication. Ensure these keys are securely managed.

2. **Data Encryption**: While Qdrant does not support built-in data encryption, it leaves the choice of encryption strategy to the underlying storage service. This allows for a wide variety of encryption methods to be employed depending on the specific storage solution used, providing greater flexibility.

3. **Authentication**: Qdrant's can be easily integrated with any existing authentication system at the service level, such as a reverse proxy. This allows for seamless integration with existing infrastructure without the need for modifications to accommodate a built-in authentication system.

4. **Network Security**: Qdrant leaves network security to be handled at the environment level. This approach allows for a wide range of network security configurations to be employed depending on the specific needs of the deployment environment.

5. **Reporting Security Issues**: Qdrant encourages the reporting of any security-related issues to their dedicated security email, demonstrating their commitment to ongoing security improvements.

### Pinecone

1. **Authentication**: Pinecone requires a valid API key and environment pair for accessing its services, which can be a limiting factor if integration with an existing authentication system is desired.

2. **Data and Network Safeguards**: Pinecone runs on a fully managed and secure AWS infrastructure, which may not provide the flexibility required for some deployment scenarios.

3. **Compliance and Certifications**: While Pinecone's SOC2 Type II certification and GDPR-readiness are reassuring, they may not be sufficient for some organizations who want to work strictly within their VPC. This means deploying an on-premise of Pinecone under enterprise offering, which can be prohibitively expensive for some organizations.ß

4. **Security Policies and Practices**: Pinecone's rigorous security policies may not align with the security practices of all organizations. This also moves the burden of finding the difference between the security policies and ironing them out to the end user.

5. **Incident Management and Monitoring**: Pinecone's incident management and monitoring practices are not integrated with the organisation's existing incident management and monitoring systems, potentially complicating incident response.

In conclusion, Qdrant's minimalist approach to security allows for greater flexibility and customization according to the specific needs of the deployment environment. It puts the control of security measures in the hands of the deployer, allowing for robust, tailored security configurations. On the other hand, Pinecone's built-in security measures and compliance certifications provide a comprehensive, ready-to-use security solution that may not provide the same level of customization as Qdrant. The choice between the two depends largely on the specific security needs of your application and the flexibility of your deployment environment.

### Phased Rollout Plan

Here is a sample migration plan:

1. **Understanding Qdrant** (2 weeks): Spend a few hours getting familiar with Qdrant, its documentation, and its APIs. This includes understanding how to create collections, add points, and query collections

2. **Planning the migration** (1 week): Create a detailed plan for the migration, including a data migration plan (how to move your vectors and metadata from Pinecone to Qdrant), a feature migration plan (how to ensure all features you're currently using in Pinecone are available and set up in Qdrant), and a rollback plan (in case there are unforeseen issues during the migration). The information in guide should help you do this a lot better.

3. **Setting up a parallel Qdrant system** (1 week): Set up a Qdrant system running in parallel with your current Pinecone system. This would allow you to start testing Qdrant without affecting your existing Pinecone system.

4. **Migrating data** (2-3 weeks): This involves transferring your vectors and metadata from Pinecone to Qdrant. The exact duration will depend on the amount of data to be transferred and the rate limitations of Pinecone APIs.

5. **Testing and Switching Over** (2 weeks): Once the data has been migrated, you'll need to thoroughly test the Qdrant system to ensure it's working as expected. Once testing is complete and you're confident in the Qdrant system, you can switch over from Pinecone to Qdrant.

6. **Monitoring and optimizing** (ongoing): After the switch, you'll want to closely monitor the Qdrant system to ensure it's performing well and optimize as needed.

Please note that these are rough estimates and the actual timeline may vary based on the specific details of your setup and the complexity of the migration.