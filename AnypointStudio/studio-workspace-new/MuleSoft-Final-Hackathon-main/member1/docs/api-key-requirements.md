# API Key Authentication Requirements

## 1. Overview

The IoT Sensor Data Ingestion API authenticates devices using API keys. Each device has a unique API key. The key is never stored in plain text — only its SHA-256 hash is stored in the database. On every request, the API hashes the incoming key and compares it against the stored hash.

---

## 2. API Key Format

API keys follow this pattern:

    iot_<segment>_<random>

Example:

    iot_ajdnnd_A1x7Kp4mQ9vR

Rules:
- Keys must start with the prefix `iot_`
- Keys are case-sensitive
- Keys are assigned per device and must not be shared between devices

---

## 3. How to Send the API Key

Include the API key in the HTTP request header on every POST request.

    Header : x-api-key
    Value  : iot_ajdnnd_A1x7Kp4mQ9vR

The header is required. Requests without it will be rejected with 400 Bad Request.

---

## 4. Authentication Flow

Step 1 — Receive the POST request with x-api-key header and JSON body.

Step 2 — Validate that x-api-key header is present and non-empty.
         If missing → return 400 Bad Request.

Step 3 — Validate that deviceId is present in the request body.
         If missing → return 400 Bad Request.

Step 4 — Query the database:

    SELECT device_id, site_id, device_status
    FROM device_registry
    WHERE device_id = :deviceId
      AND api_key_hash = SHA2(:apiKey, 256)

Step 5 — Check the query result:
         No row returned → return 401 Unauthorized (wrong key or unknown device)
         Row returned    → check device_status:
                           INACTIVE → return 403 Forbidden
                           ACTIVE   → return 202 Accepted

---

## 5. Database Storage

API keys are stored as lowercase hexadecimal SHA-256 hashes. The plain-text key is never written to the database.

    Table   : device_registry
    Column  : api_key_hash
    Type    : VARCHAR(128)
    Format  : lowercase hex SHA-256

To insert a new device with a hashed key:

    INSERT INTO device_registry (device_id, site_id, device_status, api_key_hash)
    VALUES ('DEV011', 'SITE-D-01', 'ACTIVE', SHA2('iot_ajdnnd_yourkey', 256));

To verify a hash manually:

    SHA-256("iot_ajdnnd_A1x7Kp4mQ9vR")
    = 522e96ad36e2443318abad5634613bb1b89ccb104e4b466f53ef4219f6060658

---

## 6. Registered Test Devices

These devices are pre-loaded in the database for local testing only.

    Device ID   API Key                         Status
    ---------   --------------------------------  --------
    DEV001      iot_ajdnnd_A1x7Kp4mQ9vR          ACTIVE
    DEV002      iot_ajdnnd_B2y8Lm5nW0sT          ACTIVE
    DEV003      iot_ajdnnd_C3z9Np6qX1uV          INACTIVE
    DEV004      iot_ajdnnd_D4a0Qr7rY2wX          INACTIVE
    DEV005      iot_ajdnnd_E5b1St8sZ3xY          INACTIVE
    DEV006      iot_ajdnnd_F6c2Uv9tA4zB          INACTIVE
    DEV007      iot_ajdnnd_G7d3Wx0uB5aC          INACTIVE
    DEV008      iot_ajdnnd_H8e4Yz1vC6bD          INACTIVE
    DEV009      iot_ajdnnd_J9f5Za2wD7cE          INACTIVE
    DEV010      iot_ajdnnd_K0g6Ab3xE8dF          INACTIVE

Do not use these credentials in production.

---

## 7. HTTP Response Reference

    Scenario                              Status   Description
    ------------------------------------  -------  ----------------------------------------
    Missing x-api-key header             400      Missing required header: x-api-key
    Missing or empty deviceId            400      Missing or empty required field: deviceId
    Empty readings array                 400      Readings array must contain at least one reading
    Missing reading field                400      Reading is missing required fields
    Unknown device or incorrect key      401      Invalid device ID or API key
    Device is INACTIVE                   403      Device is not active
    Valid ACTIVE device                  202      Telemetry accepted for processing
    Database unavailable                 500      The service is temporarily unavailable

---

## 8. Security Rules

1. Never log the plain-text API key. Log only the device ID and the authentication result.
2. Never include the API key or its hash in any HTTP response.
3. Return the same 401 message for both an unknown device and a wrong key. Do not reveal which condition occurred.
4. Always use parameterized SQL queries. Never concatenate the API key into a query string.
5. Database errors must return 500 Internal Server Error, not 401 Unauthorized.
6. Use HTTPS in production to protect the key during transmission.

---

## 9. Postman Test Setup

Method  : POST
URL     : http://localhost:8081/api/telemetry

Headers:
    Content-Type : application/json
    x-api-key    : iot_ajdnnd_A1x7Kp4mQ9vR

Body (raw JSON):

    {
      "deviceId": "DEV001",
      "readings": [
        {
          "metricName": "TEMPERATURE",
          "value": 30.5,
          "unit": "CELSIUS",
          "timestamp": "2026-10-09T09:00:00Z"
        }
      ]
    }

Expected response: 202 Accepted

---

## 10. Test Cases

    Test    Description                          Expected Status
    ------  -----------------------------------  ---------------
    A       DEV001 + correct key                 202 Accepted
    B       DEV002 + correct key                 202 Accepted
    C       DEV001 + wrong key                   401 Unauthorized
    D       Missing x-api-key header             400 Bad Request
    E       Unknown device + any key             401 Unauthorized
    F       DEV003 + correct key (inactive)      403 Forbidden
    G       DEV010 + correct key (inactive)      403 Forbidden
    H       Missing deviceId in body             400 Bad Request
    I       Empty readings array                 400 Bad Request
    J       Reading missing unit field           400 Bad Request
