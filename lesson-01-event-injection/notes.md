# Lesson 1: Event Injection

## Part 1) Goal and Vulnerability Summary
This lesson demonstrates an event injection vulnerability in the DVSA order processing workflow. The affected component is the DVSA-ORDER-MANAGER Lambda function, which accepts user-controlled input and passes it through an unsafe deserialization function. The security impact is that an attacker can inject malicious JavaScript functions directly into the API request body, which are then executed on the backend Lambda environment. The root weakness is the use of the node-serialize library which evaluates serialized functions from attacker-controlled input.

## Part 2) Why This Works / Root Cause
The vulnerable behavior is caused by the use of `serialize.unserialize()` on attacker-controlled input. Unlike `JSON.parse()`, the node-serialize library accepts serialized JavaScript functions. When the payload contains the special marker `$$ND_FUNC$$` followed by a function definition ending with parentheses, the library evaluates and executes the function body during deserialization. This means an attacker can achieve code execution on the backend as soon as the payload is parsed, before any other application logic runs.

## Part 3) Environment and Setup
- DVSA API URL: https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/Stage/order
- AWS Region: us-east-1
- Lambda function involved: DVSA-ORDER-MANAGER
- CloudWatch log group: /aws/lambda/DVSA-ORDER-MANAGER
- Tools used: curl, AWS CloudShell, AWS CloudWatch

## Part 4) Reproduction Steps

**Step 1** — Ensure the vulnerable code is deployed. In DVSA-ORDER-MANAGER, lines 89-90 should read:

```javascript
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

**Step 2** — Run the injection payload from CloudShell:

```bash
curl -X POST "https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/Stage/order" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer dummy" \
-d '{"action": "_$$ND_FUNC$$_function(){ console.error(\"LAMEES_LESSON_1_SUCCESS\"); }()", "cart-id": ""}'
```

**Step 3** — The terminal returns `{"message": "Internal server error"}` which is expected. The important evidence is in the backend logs.

**Step 4** — Open CloudWatch → /aws/lambda/DVSA-ORDER-MANAGER → most recent log stream and look for the injected message.

## Part 5) Evidence and Proof

The CloudWatch log stream shows the following entries after the injection request:
START RequestId: eb0f2555-5620-48ee-b949-33dab392d0e4
ERROR   LAMEES_LESSON_1_SUCCESS
ERROR   Invoke Error: TypeError...
END RequestId: eb0f2555-5620-48ee-b949-33dab392d0e4

The `LAMEES_LESSON_1_SUCCESS` message confirms that the injected JavaScript function executed successfully on the backend. The fact that the terminal returned a generic error is not a failure — the malicious code ran before the application crashed. This proves that attacker-controlled input was treated as executable code rather than plain data.

*(See screenshot: CloudWatch_Success.png)*

## Part 6) Fix Strategy / Probable Mitigation
The fix belongs in the DVSA-ORDER-MANAGER Lambda function. The `serialize.unserialize()` calls must be replaced with safe JSON parsing that does not evaluate functions. Input should be strictly validated against an expected schema before processing. The node-serialize library should be removed from the dependency chain entirely as it poses an inherent risk when used with untrusted input.

## Part 7) Code / Config Changes

**File changed:** DVSA-ORDER-MANAGER/order-manager.js

**Before the fix**, the handler used unsafe deserialization on both the request body and headers:

```javascript
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

*(See screenshot: Vulnerable Code.png)*

**After the fix**, safe JSON parsing is used instead:

```javascript
var req = typeof event.body === "string" ? JSON.parse(event.body) : event.body;
var headers = typeof event.headers === "string" ? JSON.parse(event.headers) : event.headers;
```

*(See screenshot: Fixed Code.png)*

The main change is replacing `serialize.unserialize()` with `JSON.parse()`, which only parses data and never evaluates functions, completely eliminating the injection vector.

## Part 8) Verification After Fix

After deploying the fix, the same injection curl command was run again. Checking CloudWatch logs for the most recent log stream showed only START, END, and REPORT entries with no `LAMEES_LESSON_1_SUCCESS` message appearing. The injected function was no longer executed, confirming the fix successfully prevents event injection.

*(See screenshot: Verification After Fix.png)*

## Part 9) Structured Operation and Security Analysis

**Part A — Intended Logic and Behavior Comparison:**

The vulnerability is Event Injection. The intended rule is that user-supplied request data must be treated strictly as plain data and must never be evaluated or executed by the backend. The artifacts used to infer this rule include the DVSA order API request, the order-manager.js source code, the node-serialize library behavior, CloudShell curl output, and CloudWatch log entries.

Under normal intended behavior, a JSON request body sent to the order API should be parsed as plain data and routed to the appropriate Lambda function based on the action field. No part of the request body should ever be executed as code. Under the exploit, the `$$ND_FUNC$$` marker in the request body caused node-serialize to evaluate the injected function during deserialization, executing attacker-controlled JavaScript on the Lambda backend before any other logic ran.

**Part B — Deviation Analysis and Fix:**

The exploit represents a security deviation because the backend treated attacker-controlled input as executable code, violating the fundamental principle that data and code must remain separate. This is classified as intentional misuse / security-relevant abuse.

The fix was applied in DVSA-ORDER-MANAGER/order-manager.js by replacing `serialize.unserialize()` with `JSON.parse()` for both the request body and headers. After the fix, the same injection payload was sent and CloudWatch logs showed no execution of the injected function, confirming the issue is resolved.

## Part 10) Takeaway / Lessons Learned
This lesson demonstrates one of the most dangerous classes of vulnerability: treating user input as executable code. In serverless environments this is especially severe because Lambda functions run with IAM execution roles and have access to environment variables, cloud services, and sensitive credentials. A single unsafe deserialization call can give an attacker full code execution inside a privileged cloud environment. The general principle that prevents this is strict input validation — user-supplied data must always be parsed safely and never evaluated.
