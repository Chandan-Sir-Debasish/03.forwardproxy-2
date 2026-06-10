# Test Cases and Scenarios: Nginx Forward Proxy Verification

This document specifies the test cases and scenarios designed to validate the access control list (ACL) rules, DNS hostname resolution, and domain-based content filtering configuration for the Nginx Forward Proxy.

---

## Scenario 1: IP-Based Access Control List (Port 8888)

This scenario verifies that Nginx correctly permits whitelisted client containers while blocking any other unauthorized client IPs.

### Test Case 1.1: Request from Authorized Client (client1)
*   **Description:** Validate that client1 (IP 172.20.0.10) can successfully make proxy requests.
*   **Command:**
    ```bash
    docker compose exec client1 curl -I -x http://172.20.0.2:8888 http://example.com
    ```
*   **Expected Behavior:** The request is allowed. Nginx forwards the connection and returns HTTP 200 OK.

### Test Case 1.2: Request from Authorized Client (client2)
*   **Description:** Validate that client2 (IP 172.20.0.11) can successfully make proxy requests.
*   **Command:**
    ```bash
    docker compose exec client2 curl -I -x http://172.20.0.2:8888 http://example.com
    ```
*   **Expected Behavior:** The request is allowed. Nginx forwards the connection and returns HTTP 200 OK.

### Test Case 1.3: Request from Unauthorized Client (Host Machine)
*   **Description:** Validate that requests originating from the host machine (IP 172.20.0.1) are rejected.
*   **Command:**
    ```bash
    curl.exe -I -x http://localhost:8888 http://example.com
    ```
*   **Expected Behavior:** The request is rejected by Nginx before forwarding. The proxy returns HTTP 403 Forbidden.

---

## Scenario 2: Internal DNS Name Resolution (Port 8888)

This scenario verifies that Nginx resolves internal container hostnames within the Docker bridge network.

### Test Case 2.1: Resolving and Contacting Sibling Container
*   **Description:** Validate that Nginx can resolve "client2" to its container IP (172.20.0.11).
*   **Command:**
    ```bash
    docker compose exec client1 curl -I -x http://172.20.0.2:8888 http://client2
    ```
*   **Expected Behavior:** Nginx resolves "client2" to 172.20.0.11 and attempts connection on port 80. Since client2 is not hosting a web server, Nginx gets a Connection Refused and returns HTTP 502 Bad Gateway. The error log must show: `connect() failed (111: Connection refused) while connecting to upstream ... upstream: "http://172.20.0.11:80/"`.

---

## Scenario 3: Domain-Based Content Filtering Basic Checks (Port 8889)

This scenario verifies that Nginx intercepts and blocks specific domain categories while permitting non-blacklisted traffic.

### Test Case 3.1: Accessing an Allowed Web Domain
*   **Description:** Validate that example.com is allowed.
*   **Command:**
    ```bash
    docker compose exec client1 curl -I -x http://172.20.0.2:8889 http://example.com
    ```
*   **Expected Behavior:** Request is proxied. Target server returns HTTP 200 OK.

### Test Case 3.2: Accessing a Blocked Social Media Domain (facebook.com)
*   **Description:** Validate that facebook.com is blocked by company policy.
*   **Command:**
    ```bash
    docker compose exec client1 curl -i -x http://172.20.0.2:8889 http://facebook.com
    ```
*   **Expected Behavior:** Request is intercepted. Nginx returns HTTP 403 Forbidden with plain-text body: `"Access to social media is blocked by company policy"`.

### Test Case 3.3: Accessing a Blocked Subdomain (sub.facebook.com)
*   **Description:** Validate that subdomains of blacklisted domains are also blocked.
*   **Command:**
    ```bash
    docker compose exec client1 curl -i -x http://172.20.0.2:8889 http://sub.facebook.com
    ```
*   **Expected Behavior:** Request is intercepted. Nginx returns HTTP 403 Forbidden with policy block body.

---

## Scenario 4: Boundary and Case Sensitivity Verification (Port 8889)

This scenario validates the boundaries of regex patterns to ensure case-insensitivity works and false positive domain blockings are prevented.

### Test Case 4.1: Case Insensitivity Verification (FACEBOOK.COM)
*   **Description:** Validate that blocking is case-insensitive.
*   **Command:**
    ```bash
    docker compose exec client1 curl -i -x http://172.20.0.2:8889 http://FACEBOOK.COM
    ```
*   **Expected Behavior:** The request is blocked and returns HTTP 403 Forbidden because of the case-insensitive regex operator (`~*`).

### Test Case 4.2: Domain Boundary Protection (myfacebook.com)
*   **Description:** Validate that allowed domains containing keyword substrings are not blocked.
*   **Command:**
    ```bash
    docker compose exec client1 curl -i -x http://172.20.0.2:8889 http://myfacebook.com
    ```
*   **Expected Behavior:** Nginx does not match `myfacebook.com` against `facebook.com` because it lacks a dot boundary (`.facebook.com`) or start of line boundary (`facebook.com`). The request is allowed, forwarding to the destination.

---

## Scenario 5: Transparent Protocol Forwarding Verification (Port 8889)

This scenario validates that Nginx preserves and forwards query parameters, request headers, and POST payloads intact to the destination.

### Test Case 5.1: Query Parameter Preservation
*   **Description:** Validate that query parameters are forwarded unmodified.
*   **Command:**
    ```bash
    docker compose exec client1 curl -s -x http://172.20.0.2:8889 "http://httpbin.org/get?testParam=active&user=teacher"
    ```
*   **Expected Behavior:** Nginx proxies the request to httpbin.org/get preserving the parameters, which are echoed back in the JSON body under `"args"`.

### Test Case 5.2: Request Header Forwarding
*   **Description:** Validate that custom HTTP headers are forwarded unmodified.
*   **Command:**
    ```bash
    docker compose exec client1 curl -s -x http://172.20.0.2:8889 -H "X-Academic-Institution: DevOps-Lab-University" http://httpbin.org/headers
    ```
*   **Expected Behavior:** Nginx proxies the headers, and the response from httpbin.org/headers echoes the custom header in the JSON body under `"headers"`.

### Test Case 5.3: HTTP POST Body Payload Forwarding
*   **Description:** Validate that HTTP POST bodies are preserved.
*   **Command:**
    ```bash
    docker compose exec client1 curl -s -x http://172.20.0.2:8889 -X POST -H "Content-Type: application/json" -d '{"testKey":"testValue"}' http://httpbin.org/post
    ```
*   **Expected Behavior:** Nginx forwards the payload, and the response from httpbin.org/post echoes the JSON payload in the response body.
