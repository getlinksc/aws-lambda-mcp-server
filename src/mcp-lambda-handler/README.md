# MCP Lambda Handler

A Python library for building serverless [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) HTTP servers on AWS Lambda. Register tools and resources using decorators, plug in session storage, and drop the handler into any Lambda function.

## Installation

```bash
pip install aws-lambda-mcp-server
```

## Quick Start

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server", version="1.0.0")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

def lambda_handler(event, context):
    return mcp.handle_request(event, context)
```

## Tools

### Type hints become the input schema automatically

```python
from typing import Optional
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server")

@mcp.tool()
def search_products(
    query: str,
    max_results: int,
    category: Optional[str],
) -> str:
    """Search the product catalogue.

    Args:
        query: Search keywords
        max_results: Maximum number of results to return
        category: Optional category filter
    """
    # ... your logic here
    return f"Found results for: {query}"
```

Supported types: `str`, `int`, `float`, `bool`, `List[T]`, `Dict[str, T]`, `Optional[T]`, and `Enum`.

### Enum parameters

```python
from enum import Enum
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server")

class Environment(Enum):
    DEV = "dev"
    STAGING = "staging"
    PROD = "prod"

@mcp.tool()
def deploy(service: str, env: Environment) -> str:
    """Deploy a service to an environment.

    Args:
        service: Name of the service to deploy
        env: Target environment
    """
    return f"Deploying {service} to {env.value}"
```

### Returning images

Return `bytes` from a tool and the handler automatically base64-encodes them as an `image` content block. JPEG, PNG, GIF, and WebP are auto-detected.

```python
import boto3
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server")

@mcp.tool()
def get_chart(metric: str) -> bytes:
    """Generate a chart for a metric and return it as an image.

    Args:
        metric: The metric name to chart
    """
    s3 = boto3.client("s3")
    obj = s3.get_object(Bucket="my-charts", Key=f"{metric}.png")
    return obj["Body"].read()
```

## Resources

### Static resource

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler
from awslabs.mcp_lambda_handler.types import StaticResource

mcp = MCPLambdaHandler(name="my-mcp-server")

mcp.add_resource(StaticResource(
    uri="config://app/settings",
    name="App Settings",
    content='{"timeout": 30, "retries": 3}',
    description="Application configuration",
    mime_type="application/json",
))
```

### File resource

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler
from awslabs.mcp_lambda_handler.types import FileResource

mcp = MCPLambdaHandler(name="my-mcp-server")

mcp.add_resource(FileResource(
    uri="docs://openapi",
    path="/var/task/openapi.yaml",
    name="OpenAPI Spec",
    description="API specification",
))
```

### Resource decorator (dynamic content)

```python
import json
import boto3
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server")

@mcp.resource(
    uri="data://live-config",
    name="Live Config",
    description="Config fetched from Parameter Store at read time",
    mime_type="application/json",
)
def live_config():
    ssm = boto3.client("ssm")
    value = ssm.get_parameter(Name="/myapp/config")["Parameter"]["Value"]
    return value
```

## Session Management

### Stateless (default)

No session store is configured — each request is independent. Session IDs are still issued but nothing is persisted.

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server")
```

### DynamoDB sessions

Pass a DynamoDB table name to enable persistent sessions with a 24-hour TTL.

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(
    name="my-mcp-server",
    session_store="mcp-sessions",  # DynamoDB table name
)
```

Or use `DynamoDBSessionStore` directly:

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler
from awslabs.mcp_lambda_handler.session import DynamoDBSessionStore

mcp = MCPLambdaHandler(
    name="my-mcp-server",
    session_store=DynamoDBSessionStore(table_name="mcp-sessions"),
)
```

### Reading and writing session data from a tool

```python
from awslabs.mcp_lambda_handler import MCPLambdaHandler

mcp = MCPLambdaHandler(name="my-mcp-server", session_store="mcp-sessions")

@mcp.tool()
def increment_counter() -> str:
    """Increment the per-session counter."""
    session = mcp.get_session()
    count = session.get("count", 0) + 1 if session else 1
    mcp.update_session(lambda s: s.set("count", count))
    return f"Count is now {count}"

@mcp.tool()
def get_counter() -> str:
    """Get the current per-session counter value."""
    session = mcp.get_session()
    count = session.get("count", 0) if session else 0
    return f"Count: {count}"
```

### Custom session backend

Subclass `SessionStore` to use any storage backend:

```python
from typing import Any, Dict, Optional
from awslabs.mcp_lambda_handler import MCPLambdaHandler
from awslabs.mcp_lambda_handler.session import SessionStore
import uuid

class RedisSessionStore(SessionStore):
    def __init__(self, redis_client):
        self.redis = redis_client

    def create_session(self, session_data: Optional[Dict[str, Any]] = None) -> str:
        session_id = str(uuid.uuid4())
        self.redis.setex(session_id, 86400, json.dumps(session_data or {}))
        return session_id

    def get_session(self, session_id: str) -> Optional[Dict[str, Any]]:
        data = self.redis.get(session_id)
        return json.loads(data) if data else None

    def update_session(self, session_id: str, session_data: Dict[str, Any]) -> bool:
        self.redis.setex(session_id, 86400, json.dumps(session_data))
        return True

    def delete_session(self, session_id: str) -> bool:
        self.redis.delete(session_id)
        return True

mcp = MCPLambdaHandler(
    name="my-mcp-server",
    session_store=RedisSessionStore(redis_client),
)
```

## Example Architecture

A typical deployment looks like:

```
Client → API Gateway (/mcp)
              │
              ├── Lambda Authorizer  (validates bearer token)
              │
              └── MCP Lambda         (this library)
                        │
                        └── DynamoDB  (optional session store)
```

The Lambda function receives API Gateway proxy events and returns proxy responses — no extra configuration needed.

## Development

```bash
git clone https://github.com/awslabs/mcp.git
cd mcp/src/mcp-lambda-handler
pip install -e ".[dev]"
pytest
```

## License

Apache-2.0 — see [LICENSE](LICENSE).
