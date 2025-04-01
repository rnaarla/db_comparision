# OpenSearch Serverless Implementation Guide
## Overcoming Limitations for Product Catalog Applications

This guide provides practical implementation patterns to overcome the limitations of AWS OpenSearch Serverless when using it for product catalog applications. While OpenSearch Managed offers more features, Serverless can be a viable option if operational simplicity is prioritized.

## Table of Contents

- [Understanding Serverless Limitations](#understanding-serverless-limitations)
- [Implementation Patterns](#implementation-patterns)
  - [Cross-Collection Search Federation](#cross-collection-search-federation)
  - [Custom Document ID Workarounds](#custom-document-id-workarounds)
  - [Atomic Updates without Upsert](#atomic-updates-without-upsert)
  - [Collection Design Strategies](#collection-design-strategies)
  - [Client-Side Caching](#client-side-caching)
- [Setup and Configuration](#setup-and-configuration)
- [Usage Examples](#usage-examples)
- [Performance Considerations](#performance-considerations)

## Understanding Serverless Limitations

AWS OpenSearch Serverless has several limitations compared to OpenSearch Managed:

1. **Cross-Collection Operations**: Serverless doesn't support searching across multiple collections.
2. **Custom Document IDs**: Limited support for custom document IDs in time-series collections.
3. **Upsert Limitations**: Restricted upsert support for time-series collections.
4. **Plugin Restrictions**: No custom plugin uploads; limited to pre-installed plugins.

Despite these limitations, we can implement workarounds to still use Serverless effectively.

## Implementation Patterns

### Cross-Collection Search Federation

Since Serverless doesn't support searching across multiple collections, we can implement client-side collection federation:

```python
import asyncio
from opensearchpy import AsyncOpenSearch

async def search_across_collections(client, query_body, collection_names):
    """Search across multiple Serverless collections and merge results."""
    
    async def search_single_collection(collection_name):
        response = await client.search(
            index=collection_name,
            body=query_body
        )
        # Add collection source to each hit
        for hit in response['hits']['hits']:
            hit['_collection'] = collection_name
        return response
    
    # Run searches in parallel
    tasks = [search_single_collection(name) for name in collection_names]
    responses = await asyncio.gather(*tasks)
    
    # Merge results
    merged_results = {
        "took": sum(r["took"] for r in responses),
        "hits": {
            "total": {
                "value": sum(r["hits"]["total"]["value"] for r in responses),
                "relation": "eq"
            },
            "hits": []
        }
    }
    
    # Combine and sort by score
    all_hits = []
    for r in responses:
        all_hits.extend(r["hits"]["hits"])
    
    # Sort combined results by score
    merged_results["hits"]["hits"] = sorted(
        all_hits, 
        key=lambda x: x["_score"], 
        reverse=True
    )
    
    return merged_results
```

**Usage Example**:

```python
# Setup async client
from opensearchpy import AsyncOpenSearch, RequestsHttpConnection, AWSV4SignerAuth
import boto3
import asyncio

async def main():
    # Configure AWS credentials
    region = 'us-east-1'
    service = 'aoss'
    credentials = boto3.Session().get_credentials()
    auth = AWSV4SignerAuth(credentials, region, service)
    
    # Initialize the client
    client = AsyncOpenSearch(
        hosts=[{'host': 'your-endpoint.us-east-1.aoss.amazonaws.com', 'port': 443}],
        http_auth=auth,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection
    )
    
    # Define collections and query
    collections = ["satellite-basic", "satellite-premium", "satellite-addons"]
    query = {
        "query": {
            "multi_match": {
                "query": "sports entertainment",
                "fields": ["productname", "productdescription"]
            }
        }
    }
    
    # Search across collections
    results = await search_across_collections(client, query, collections)
    print(f"Found {results['hits']['total']['value']} results across all collections")
    
    # Close client
    await client.close()

# Run the async function
asyncio.run(main())
```

### Custom Document ID Workarounds

To work around limitations with custom document IDs in time-series collections, use search collections instead and implement time filtering via fields:

```python
from datetime import datetime

def index_product_with_pricing(client, collection_name, product_data):
    """Index product with embedded pricing history in a search collection."""
    
    # Use the product code as document ID (fully supported in search collections)
    document_id = product_data["productCode"]
    
    # Ensure each price point has a timestamp field for filtering
    for price in product_data.get("prices", []):
        # Convert dates to timestamps if needed
        if isinstance(price["startDate"], str):
            price["startTimestamp"] = int(datetime.fromisoformat(
                price["startDate"].replace("Z", "+00:00")
            ).timestamp() * 1000)
        
        if isinstance(price["endDate"], str):
            price["endTimestamp"] = int(datetime.fromisoformat(
                price["endDate"].replace("Z", "+00:00")
            ).timestamp() * 1000)
    
    # Index the full document
    response = client.index(
        index=collection_name,
        body=product_data,
        id=document_id,
        refresh=True
    )
    
    return response
```

**Query for Current Prices**:

```python
def get_current_prices(client, collection_name, product_code=None):
    """Find all current valid prices across products or for specific product."""
    
    now = int(datetime.now().timestamp() * 1000)
    
    query = {
        "query": {
            "bool": {
                "must": [
                    {
                        "nested": {
                            "path": "prices",
                            "query": {
                                "bool": {
                                    "must": [
                                        {"range": {"prices.startTimestamp": {"lte": now}}},
                                        {"range": {"prices.endTimestamp": {"gte": now}}}
                                    ]
                                }
                            },
                            "inner_hits": {}  # Return the matching price points
                        }
                    }
                ]
            }
        }
    }
    
    # Add product filter if specified
    if product_code:
        query["query"]["bool"]["must"].append({"term": {"productCode": product_code}})
    
    response = client.search(
        index=collection_name,
        body=query
    )
    
    return response
```

### Atomic Updates without Upsert

Implement atomic updates using optimistic concurrency control to work around upsert limitations:

```python
def update_product_pricing(client, collection_name, product_code, new_price_data):
    """Update product pricing with optimistic concurrency control."""
    
    # First, get the current document and its version
    try:
        current_doc = client.get(
            index=collection_name,
            id=product_code
        )
    except Exception as e:
        return {"error": f"Product not found: {str(e)}"}
    
    # Extract the current document and its prices
    source = current_doc["_source"]
    current_prices = source.get("prices", [])
    
    # Update the prices array
    updated_prices = current_prices.copy()
    
    # Logic to update, replace or append the new price
    price_updated = False
    for i, price in enumerate(updated_prices):
        # Find matching price by billingReferenceId
        if price["billingReferenceId"] == new_price_data["billingReferenceId"]:
            updated_prices[i] = new_price_data
            price_updated = True
            break
            
    # If not found, append the new price
    if not price_updated:
        updated_prices.append(new_price_data)
    
    # Create the updated document
    source["prices"] = updated_prices
    
    # Update with version control to ensure atomic update
    try:
        response = client.index(
            index=collection_name,
            body=source,
            id=product_code,
            if_seq_no=current_doc["_seq_no"],
            if_primary_term=current_doc["_primary_term"],
            refresh=True
        )
        return {"status": "success", "response": response}
    except Exception as e:
        # Handle version conflict
        return {"error": f"Update conflict, try again: {str(e)}"}
```

### Collection Design Strategies

Implement a structured collection naming and data partitioning strategy:

```python
def determine_collection_name(product_data):
    """Determine the appropriate collection name based on product attributes."""
    
    product_family = product_data.get("productFamily", "unknown")
    current_year = datetime.now().year
    
    # Create structured collection naming - better for management
    # Format: {product_family}-{current_year}
    collection_name = f"{product_family}-{current_year}"
    
    return collection_name

def initialize_collections(client, product_families):
    """Initialize collections with correct mappings for each product family."""
    
    for family in product_families:
        collection_name = f"{family}-{datetime.now().year}"
        
        # Check if collection exists
        exists = client.indices.exists(index=collection_name)
        if not exists:
            # Create with appropriate mappings for product data
            mapping = {
                "mappings": {
                    "properties": {
                        "productCode": {"type": "keyword"},
                        "productname": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                        "productdescription": {"type": "text"},
                        "prices": {
                            "type": "nested",
                            "properties": {
                                "startDate": {"type": "date"},
                                "endDate": {"type": "date"},
                                "startTimestamp": {"type": "long"},
                                "endTimestamp": {"type": "long"},
                                "billingReferenceId": {"type": "keyword"}
                            }
                        }
                    }
                }
            }
            
            client.indices.create(
                index=collection_name,
                body=mapping
            )
            print(f"Created collection: {collection_name}")
```

### Client-Side Caching

Reduce the impact of querying multiple collections with efficient caching:

```python
import functools
from datetime import datetime, timedelta

# Simple TTL cache implementation
def ttl_cache(ttl_seconds=300):
    """Time-based caching decorator."""
    def decorator(func):
        cache = {}
        
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # Create a cache key from function arguments
            key = str(args) + str(kwargs)
            now = datetime.now()
            
            # Check if result is in cache and not expired
            if key in cache:
                result, timestamp = cache[key]
                if now - timestamp < timedelta(seconds=ttl_seconds):
                    return result
            
            # Call the function and cache the result
            result = func(*args, **kwargs)
            cache[key] = (result, now)
            
            # Clean expired entries
            expired_keys = [k for k, (_, ts) in cache.items() 
                          if now - ts >= timedelta(seconds=ttl_seconds)]
            for k in expired_keys:
                del cache[k]
                
            return result
        return wrapper
    return decorator

# Apply to search functions
@ttl_cache(ttl_seconds=60)  # Cache results for 1 minute
def search_product_catalog(client, query_text, collections):
    # Implementation of the search
    # ...
    pass
```

## Setup and Configuration

### Installation

```bash
pip install opensearch-py boto3 requests-aws4auth
```

### Connection Setup

```python
import boto3
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth

def get_opensearch_client(region='us-east-1'):
    """Create an OpenSearch Serverless client with proper authentication."""
    
    # AWS credentials
    service = 'aoss'  # 'aoss' for OpenSearch Serverless
    credentials = boto3.Session().get_credentials()
    awsauth = AWS4Auth(
        credentials.access_key,
        credentials.secret_key,
        region,
        service,
        session_token=credentials.token
    )
    
    # Serverless endpoint - replace with your collection endpoint
    host = 'your-collection-endpoint.us-east-1.aoss.amazonaws.com'
    
    # Create the client
    client = OpenSearch(
        hosts=[{'host': host, 'port': 443}],
        http_auth=awsauth,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection,
        timeout=30
    )
    
    return client
```

## Usage Examples

### Complete Example: Product Catalog Management

```python
import boto3
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
from datetime import datetime
import json

# Setup client
def get_client():
    region = 'us-east-1'
    service = 'aoss'
    credentials = boto3.Session().get_credentials()
    awsauth = AWS4Auth(
        credentials.access_key,
        credentials.secret_key,
        region,
        service,
        session_token=credentials.token
    )
    
    host = 'your-collection-endpoint.us-east-1.aoss.amazonaws.com'
    
    client = OpenSearch(
        hosts=[{'host': host, 'port': 443}],
        http_auth=awsauth,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection,
        timeout=30
    )
    
    return client

# Example product data
sample_product = {
    "productFamily": "satellite",
    "billingCode": "P121232",
    "enablerPricePlanCode": "89077483",
    "productCode": "BOLTON-TRUTV-TCM_satellite",
    "productdescription": "TruTV and TCM",
    "displayName": "TruTV and TCM",
    "billingProductCode": "P121232",
    "productname": "TruTV and TCM",
    "prices": [
        {
            "startDate": "2024-02-07T00:01:00.000Z",
            "endDate": "2050-12-31T23:59:00.000Z",
            "billingReferenceId": "1"
        },
        {
            "startDate": "2024-04-02T07:00:00.000Z",
            "endDate": "2050-12-31T23:59:00.000Z",
            "billingReferenceId": "2"
        }
    ]
}

# Main workflow
def main():
    client = get_client()
    
    # 1. Determine collection name
    collection_name = f"{sample_product['productFamily']}-products"
    
    # 2. Initialize collection if needed
    if not client.indices.exists(index=collection_name):
        mapping = {
            "mappings": {
                "properties": {
                    "productCode": {"type": "keyword"},
                    "productname": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                    "productdescription": {"type": "text"},
                    "prices": {
                        "type": "nested",
                        "properties": {
                            "startDate": {"type": "date"},
                            "endDate": {"type": "date"},
                            "startTimestamp": {"type": "long"},
                            "endTimestamp": {"type": "long"},
                            "billingReferenceId": {"type": "keyword"}
                        }
                    }
                }
            }
        }
        client.indices.create(index=collection_name, body=mapping)
        print(f"Created collection: {collection_name}")
    
    # 3. Prepare and index the product
    product_with_timestamps = sample_product.copy()
    for price in product_with_timestamps.get("prices", []):
        # Convert dates to timestamps
        if isinstance(price["startDate"], str):
            price["startTimestamp"] = int(datetime.fromisoformat(
                price["startDate"].replace("Z", "+00:00")
            ).timestamp() * 1000)
        
        if isinstance(price["endDate"], str):
            price["endTimestamp"] = int(datetime.fromisoformat(
                price["endDate"].replace("Z", "+00:00")
            ).timestamp() * 1000)
    
    # 4. Index the product
    response = client.index(
        index=collection_name,
        body=product_with_timestamps,
        id=product_with_timestamps["productCode"],
        refresh=True
    )
    print(f"Indexed product: {json.dumps(response, indent=2)}")
    
    # 5. Search for current products
    now = int(datetime.now().timestamp() * 1000)
    query = {
        "query": {
            "bool": {
                "must": [
                    {
                        "nested": {
                            "path": "prices",
                            "query": {
                                "bool": {
                                    "must": [
                                        {"range": {"prices.startTimestamp": {"lte": now}}},
                                        {"range": {"prices.endTimestamp": {"gte": now}}}
                                    ]
                                }
                            },
                            "inner_hits": {}
                        }
                    }
                ]
            }
        }
    }
    
    search_response = client.search(
        index=collection_name,
        body=query
    )
    
    print(f"\nFound {search_response['hits']['total']['value']} products with current pricing")
    for hit in search_response['hits']['hits']:
        print(f"Product: {hit['_source']['productname']}")
        print(f"  Active Price Points: {len(hit['inner_hits']['prices']['hits']['hits'])}")
    
    # 6. Update a price
    new_price = {
        "startDate": "2024-05-01T00:00:00.000Z",
        "endDate": "2050-12-31T23:59:00.000Z",
        "billingReferenceId": "2",
        "amount": 9.99,
        "currency": "USD"
    }
    
    update_response = update_product_pricing(
        client, 
        collection_name, 
        product_with_timestamps["productCode"], 
        new_price
    )
    
    print(f"\nPrice update result: {json.dumps(update_response, indent=2)}")

def update_product_pricing(client, collection_name, product_code, new_price_data):
    """Update product pricing with optimistic concurrency control."""
    
    # First, get the current document and its version
    try:
        current_doc = client.get(
            index=collection_name,
            id=product_code
        )
    except Exception as e:
        return {"error": f"Product not found: {str(e)}"}
    
    # Process date fields
    if isinstance(new_price_data["startDate"], str):
        new_price_data["startTimestamp"] = int(datetime.fromisoformat(
            new_price_data["startDate"].replace("Z", "+00:00")
        ).timestamp() * 1000)
    
    if isinstance(new_price_data["endDate"], str):
        new_price_data["endTimestamp"] = int(datetime.fromisoformat(
            new_price_data["endDate"].replace("Z", "+00:00")
        ).timestamp() * 1000)
    
    # Extract the current document and its prices
    source = current_doc["_source"]
    current_prices = source.get("prices", [])
    
    # Update the prices array
    updated_prices = current_prices.copy()
    
    # Logic to update existing price or append new price
    price_updated = False
    for i, price in enumerate(updated_prices):
        if price["billingReferenceId"] == new_price_data["billingReferenceId"]:
            updated_prices[i] = new_price_data
            price_updated = True
            break
            
    if not price_updated:
        updated_prices.append(new_price_data)
    
    # Update the source document
    source["prices"] = updated_prices
    
    # Update with version control
    try:
        response = client.index(
            index=collection_name,
            body=source,
            id=product_code,
            if_seq_no=current_doc["_seq_no"],
            if_primary_term=current_doc["_primary_term"],
            refresh=True
        )
        return {"status": "success", "response": response}
    except Exception as e:
        return {"error": f"Update conflict, try again: {str(e)}"}

if __name__ == "__main__":
    main()
```

## Performance Considerations

When implementing these workarounds for OpenSearch Serverless, keep these performance considerations in mind:

1. **Client-Side Federation Overhead**: 
   - Cross-collection searches execute multiple network requests
   - Use parallel requests with async functions
   - Consider implementing result pagination
   - Cache results when appropriate

2. **Time-Based Filtering**:
   - Pre-calculate timestamp fields during indexing
   - Create appropriate mappings with field data types optimized for range queries
   - Consider time-bucketing for historical data

3. **Optimistic Concurrency Control**:
   - Implement retry logic for version conflicts during updates
   - Monitor update failure rates
   - Consider batching updates for efficiency

4. **Connection Pooling**:
   - Use a connection pool to efficiently manage connections
   - Set appropriate timeouts and retries

5. **Collection Organization**:
   - Group related data within single collections where possible
   - Balance between collection size and query complexity
   - Consider data lifecycle (archive older products to separate collections)

## Conclusion

While OpenSearch Serverless has limitations, using the patterns above allows you to leverage its operational simplicity while still building robust product catalog applications. These workarounds add some client-side complexity but eliminate the need for managing OpenSearch infrastructure.
