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

## Scenario 3: Domain-Based Content Filtering (Port 8889)

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

### Test Case 3.4: Accessing a Domain with Keyword Substring (myfacebook.com)
*   **Description:** Validate that domains containing "facebook.com" as a substring but not representing the exact target domain are permitted.
*   **Command:**
    ```bash
    docker compose exec client1 curl -i -x http://172.20.0.2:8889 http://myfacebook.com
    ```
*   **Expected Behavior:** Request is permitted by Nginx (not blocked with 403). Nginx attempts to contact the destination server.
