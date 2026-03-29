# Testing Guide

---

## 1. Test Suite Overview

The gateway has 58 unit tests across 6 test files. All tests are in `tests/unit/`.

| File | What it tests | Tests |
|---|---|---|
| `test_pii_detector.py` | Presidio PII detection: entity recognition, threshold filtering, entity redaction | ~14 |
| `test_policy_engine.py` | Policy rule evaluation: risk scoring, block/warn/allow decisions, rule matching | ~12 |
| `test_gateway_router.py` | FastAPI route handlers: authentication, request routing, response format, error codes | ~15 |
| `test_audit_publisher.py` | SQS audit event publishing: message format, async publish behavior, failure handling | ~8 |
| `test_config.py` | Pydantic settings validation: required fields, JSON parsing, default values | ~5 |
| `test_health.py` | Health check endpoint | ~4 |

> The exact count per file may vary as the test suite grows. Run `pytest tests/unit/ --collect-only | grep "test session"` to see the current total.

---

## 2. Running Unit Tests

### Full test suite

```bash
# Must be in the sentorix-gateway directory with .venv active
cd ~/workspace/sentorix/sentorix-gateway
source .venv/bin/activate

pytest tests/unit/ -v
```

`-v` (verbose) shows each test name and pass/fail status individually, rather than just dots.

### With short tracebacks on failure

```bash
pytest tests/unit/ -v --tb=short
```

`--tb=short` shows a condensed traceback when a test fails — enough to identify the cause without the full stack. Useful when you want to quickly scan multiple failures.

### With coverage report

```bash
pytest tests/unit/ -v --cov=app --cov-report=term-missing
```

`--cov=app` measures coverage for the `app/` module (the application code).
`--cov-report=term-missing` prints a summary table showing which lines in each file are not covered by any test. Lines marked `missing` are not tested.

Example output:
```
---------- coverage: platform darwin, python 3.12.x -----------
Name                          Stmts   Miss  Cover   Missing
-----------------------------------------------------------
app/config.py                    42      3    93%   87-89
app/pii_detector.py              58      4    93%   102, 115-117
app/policy_engine.py             71      8    89%   145-152
app/routers/gateway.py          112      5    96%   88, 201-204
app/audit_publisher.py           34      0   100%
app/main.py                      22      1    95%   18
-----------------------------------------------------------
TOTAL                           339     21    94%
```

### Run tests and stop on first failure

```bash
pytest tests/unit/ -v -x
```

`-x` stops after the first test failure. Useful when you are fixing a specific broken test and want fast feedback.

---

## 3. Running Specific Test Files

```bash
# PII detection tests only
pytest tests/unit/test_pii_detector.py -v

# Policy engine tests only
pytest tests/unit/test_policy_engine.py -v

# Gateway router tests only
pytest tests/unit/test_gateway_router.py -v

# Audit publisher tests only
pytest tests/unit/test_audit_publisher.py -v

# Config validation tests only
pytest tests/unit/test_config.py -v
```

### Running a single test by name

```bash
pytest tests/unit/test_pii_detector.py::test_ssn_detection_blocked -v
```

### Running tests matching a keyword

```bash
pytest tests/unit/ -v -k "ssn"
# Runs any test whose name contains "ssn"
```

---

## 4. What Is Mocked in Tests

### AWS services (moto)

Most AWS services in tests use [moto](https://docs.getmoto.org/en/latest/) — a library that intercepts boto3 calls and provides in-memory AWS service implementations without hitting the real AWS API.

Example usage in a test:
```python
import boto3
from moto import mock_aws

@mock_aws
def test_something_that_reads_dynamodb():
    # boto3 calls inside this function hit the moto in-memory DynamoDB
    dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
    table = dynamodb.create_table(...)
    # ... test logic
```

### aioboto3 / SQS audit publishing (AsyncMock)

The audit publisher uses `aioboto3` for async SQS publishing. **Moto does not work with `aiobotocore`** (the underlying async boto3 client). When moto tries to intercept an async boto3 session, it fails because moto patches the synchronous `botocore` layer, not the async version.

For this reason, `test_audit_publisher.py` uses `unittest.mock.AsyncMock` to mock the SQS client directly:

```python
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_audit_event_published_to_sqs():
    mock_sqs_client = AsyncMock()
    mock_sqs_client.send_message.return_value = {"MessageId": "test-id"}

    with patch("app.audit_publisher.get_sqs_client", return_value=mock_sqs_client):
        publisher = AuditPublisher(queue_url="https://sqs.us-east-1.amazonaws.com/123/test-queue")
        await publisher.publish(audit_event)

    mock_sqs_client.send_message.assert_called_once()
    call_kwargs = mock_sqs_client.send_message.call_args.kwargs
    assert call_kwargs["QueueUrl"] == "https://sqs.us-east-1.amazonaws.com/123/test-queue"
    assert "MessageBody" in call_kwargs
```

This approach tests the audit publisher's behavior (correct arguments, correct method called) without requiring a real or mocked AWS network call.

### Presidio

Presidio is patched in `conftest.py` to avoid loading the spaCy model during tests (the model takes several seconds to load and is not needed for unit tests). Tests for the PII detector use a patched version that returns configurable mock detections.

```python
# In conftest.py
@pytest.fixture(autouse=True)
def mock_presidio_analyzer(mocker):
    mock = mocker.patch("app.pii_detector.AnalyzerEngine")
    mock.return_value.analyze.return_value = []  # Default: no PII detected
    return mock
```

Individual tests that need PII to be detected override this fixture to return specific detections.

### LLM providers

LLM provider calls (OpenAI, Anthropic, Bedrock) are patched in gateway router tests using `httpx.MockTransport` or `unittest.mock.patch` to return predetermined responses without making real API calls. This keeps tests fast and free (no API charges during testing).

---

## 5. Adding New Tests

### Where to put test files

Unit tests go in `tests/unit/`. Each file should test one module from `app/`.

| Application module | Test file |
|---|---|
| `app/pii_detector.py` | `tests/unit/test_pii_detector.py` |
| `app/policy_engine.py` | `tests/unit/test_policy_engine.py` |
| `app/routers/gateway.py` | `tests/unit/test_gateway_router.py` |
| `app/audit_publisher.py` | `tests/unit/test_audit_publisher.py` |
| A new module `app/foo.py` | `tests/unit/test_foo.py` |

### Using fixtures from conftest.py

The `tests/unit/conftest.py` file defines shared fixtures. Import them by parameter name in your test function — pytest injects them automatically:

```python
# conftest.py provides: mock_presidio_analyzer, test_settings, sample_request

def test_something(test_settings, sample_request):
    # test_settings is a Settings object with test values
    # sample_request is a sample ChatRequest object
    ...
```

### Example: writing an async test

Many gateway functions are async. Tests for async code must use the `@pytest.mark.asyncio` decorator:

```python
import pytest

@pytest.mark.asyncio
async def test_pii_detector_detects_email(mock_presidio_analyzer):
    # Override the default mock to return an email detection
    mock_presidio_analyzer.return_value.analyze.return_value = [
        RecognizerResult(entity_type="EMAIL_ADDRESS", start=0, end=20, score=0.85)
    ]

    detector = PIIDetector()
    result = await detector.analyze("contact me at test@example.com")

    assert len(result.entities) == 1
    assert result.entities[0].entity_type == "EMAIL_ADDRESS"
```

The `@pytest.mark.asyncio` decorator is required for any test that uses `await`. Without it, the async function returns a coroutine object instead of executing, and the test will silently pass without running any assertions.

---

## 6. Why Certain Tests Are Structured the Way They Are

### asyncio STRICT mode

The project configures pytest-asyncio in STRICT mode in `pytest.ini` or `pyproject.toml`:

```ini
[pytest]
asyncio_mode = strict
```

In STRICT mode, async test functions must have the `@pytest.mark.asyncio` decorator explicitly — they are not auto-detected. This is intentional. Auto-detection (`asyncio_mode = auto`) silently converts all async functions to tests, which can cause async utility functions or fixtures to be collected as tests accidentally.

If you write an async test and forget the decorator in STRICT mode, pytest will raise an error rather than silently skipping it:
```
PytestUnraisableExceptionWarning: Exception ignored in: <coroutine object ...>
RuntimeWarning: coroutine 'test_your_test' was never awaited
```

The fix is always: add `@pytest.mark.asyncio`.

### Why `test_audit_publisher.py` uses `AsyncMock` instead of moto

As described in Section 4, this is a known incompatibility between moto and aiobotocore. The moto library patches `botocore` (the synchronous AWS SDK core), but `aiobotocore` has its own implementation of the low-level HTTP layer that moto does not intercept.

Attempts to use moto with aioboto3 sessions result in:
```
botocore.exceptions.NoRegionError: You must specify a region
```
or the mock simply not activating, causing real AWS API calls during tests.

The `AsyncMock` approach is more reliable and arguably more readable: it tests that the publisher calls `send_message` with the correct arguments, which is exactly what we care about. The behavior of SQS itself is AWS's responsibility to test.
