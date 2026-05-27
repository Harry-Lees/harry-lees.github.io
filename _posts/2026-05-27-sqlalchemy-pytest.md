## Identify SQLAlchemy N+1 Queries in Pytest

While working on several FastAPI projects I have found it can be challenging to identify N+1 Queries in endpoints. There are things that can be done to mitigate against the issue such as using `lazy="raiseload"` on relationships to catch accidental attribute access but I still end up receiving many Sentry alerts with N+1 queries over time.

[This StackOverflow](https://stackoverflow.com/questions/19073099/how-to-count-sqlalchemy-queries-in-unit-tests) post provides
several options for counting queries using SQLAlchemy's built-in [Event API](https://docs.sqlalchemy.org/en/20/core/events.html#sqlalchemy.events.ConnectionEvents.after_execute). One of these options can be seen in the snippet below:

```python
class QueryCounter:
    def __init__(self, connection) -> None:
        self.connection = connection
        self.count = 0

    def callback(
        self,
        connection: Session,
        clause_element: str | Compiled,
        multiparams: list[dict[str, Any]],
        params: dict[str, Any],
        execution_options: dict[str, Any],
        result: CursorResult,
    ) -> None:
        self.count += 1

    def __enter__(self) -> Self:
        event.listen(self.connection, "after_execute", self.callback)
        return self

    def __exit__(self, *_) -> None:
        event.remove(self.connection, "after_execute", self.callback)

    def __repr__(self) -> str:
        return f"QueryCounter(count={self.count})"

    __str__ = __repr__
```

This could be used directly in a test snippet like the following:

```python
@pytest.fixture(name="engine")
def sqlalchemy_engine_fixture():
    return create_engine("...")

@pytest.fixture(name="query_counter")
def query_counter_fixture(engine: Engine):
    return QueryCounter(engine)

@pytest.fixture(name="test_client")
def test_client_fixture(engine: Engine):
    app.dependency_overrides[engine] = lambda : engine

    return TestClient(app)

def test_get_events(
    client: TestClient,
    query_counter: QueryCounter,
):
    with query_counter as counter:
        response = client.get("/events/1")
        assert response.status_code == status.HTTP_200_OK, response.text
        assert counter.count < 10
```

However it ends up with quite a bit of boilerplate in the test function. Through some trial-and-error I have ended up with a solution that looks more like the following

```python
@pytest.fixture(autouse=True)
def autolimit_queries(
    request: FixtureRequest,
    engine: Engine,
):
    count = 0

    def callback(
        self,
        connection: Session,
        clause_element: str | Compiled,
        multiparams: list[dict[str, Any]],
        params: dict[str, Any],
        execution_options: dict[str, Any],
        result: CursorResult,
    ) -> None:
        nonlocal count
        count += 1

    if (mark := request.node.get_closest_marker("limit_queries")) is None:
        return

    limit = mark.args[0]
    event.listen(engine, "after_execute", callback)
    yield
    event.remove(engine, "after_execute", callback)

    assert count < limit

@pytest.mark.limit_queries(10)
def test_something(client: TestClient) -> None:
    response = client.get("/events/1")
    assert response.status_code == status.HTTP_200_OK, response.text
```
