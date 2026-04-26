# Lesson 9: Vulnerable Dependencies

## Part 1) Goal and Vulnerability Summary
This lesson demonstrates how a vulnerable third-party dependency can introduce a critical security weakness into a serverless application. The affected component is the DVSA-ORDER-MANAGER Lambda function, which uses the node-serialize package to deserialize request input. This library is known to allow unsafe deserialization of JavaScript functions, making it directly responsible for enabling the remote code execution demonstrated in Lesson 1. The security impact is that any attacker who sends a crafted payload to the API can execute arbitrary code on the backend simply because an unsafe library is included in the dependency chain.

## Part 2) Why This Works / Root Cause
The node-serialize library was designed to serialize and deserialize JavaScript objects including functions. When it encounters the `$$ND_FUNC$$` marker in input, it reconstructs and evaluates the function. This behavior is intentional in the library but becomes a critical vulnerability when the library is used with attacker-controlled input. The root cause is therefore a poor dependency choice — using a library that supports function deserialization in a context where the input comes from untrusted external sources.

## Part 3) Environment and Setup
- DVSA API URL: https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/Stage/order
- AWS Region: us-east-1
- Lambda function involved: DVSA-ORDER-MANAGER
- Vulnerable package: node-serialize
- CloudWatch log group: /aws/lambda/DVSA-ORDER-MANAGER
- Tools used: curl, AWS CloudShell, AWS CloudWatch, AWS Lambda console

## Part 4) Reproduction Steps

**Step 1** — Identify the vulnerable dependency. In Lambda → DVSA-ORDER-MANAGER → Code, open package.json and confirm node-serialize is listed as a dependency.

**Step 2** — Confirm the library is used on untrusted input. In order-manager.js, lines 89-90 show:

```javascript
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

**Step 3** — Send a crafted payload that exploits the library:

```bash
curl -X POST "https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/Stage/order" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer dummy" \
-d '{"action": "_$$ND_FUNC$$_function(){ console.error(\"LAMEES_LESSON_1_SUCCESS\"); }()", "cart-id": ""}'
```

**Step 4** — Check CloudWatch → /aws/lambda/DVSA-ORDER-MANAGER → most recent log stream for the injected message as proof the library executed attacker-controlled code.

## Part 5) Evidence and Proof

The CloudWatch log stream confirms that the node-serialize library executed the injected function:
START RequestId: eb0f2555-5620-48ee-b949-33dab392d0e4
ERROR   LAMEES_LESSON_1_SUCCESS
ERROR   Invoke Error: TypeError...
END RequestId: eb0f2555-5620-48ee-b949-33dab392d0e4

The LAMEES_LESSON_1_SUCCESS message proves that the vulnerable node-serialize package evaluated and executed attacker-controlled JavaScript. The entry point was the library itself — without node-serialize, this payload would have been treated as plain text with no execution.

## Part 6) Fix Strategy / Probable Mitigation
The fix is to remove the node-serialize library entirely and replace its usage with safe JSON parsing using the built-in JSON.parse() function. JSON.parse() only handles plain data and never evaluates functions, completely eliminating the attack vector introduced by node-serialize. Additionally, all project dependencies should be regularly scanned for known vulnerabilities using tools such as npm audit to catch unsafe packages before deployment.

## Part 7) Code / Config Changes

**File changed:** DVSA-ORDER-MANAGER/order-manager.js

Before the fix:

```javascript
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

After the fix:

```javascript
var req = typeof event.body === "string" ? JSON.parse(event.body) : event.body;
var headers = typeof event.headers === "string" ? JSON.parse(event.headers) : event.headers;
```

This eliminates the use of the node-serialize library on attacker-controlled input entirely.

## Part 8) Verification After Fix
After deploying the fix, the same injection payload was sent to the API. CloudWatch logs for the most recent log stream showed only START, END, and REPORT entries with no LAMEES_LESSON_1_SUCCESS message. The node-serialize library no longer processes the request body, so the injected function is never evaluated.

## Part 9) Structured Operation and Security Analysis

The vulnerability is Vulnerable Dependencies. The intended rule is that all third-party libraries used in production must be safe for use with untrusted input and must not introduce code execution risks. The artifacts used to infer this rule include the order-manager.js source code, the node-serialize library documentation and known CVEs, the package.json dependency list, CloudShell curl output, and CloudWatch log entries.

Under normal intended behavior, the order API should safely parse incoming JSON request bodies and route them to the appropriate handler without ever evaluating any part of the input as code. Under the exploit, the node-serialize library evaluated a function embedded in the request body, executing attacker-controlled JavaScript on the Lambda backend.

The exploit represents a security deviation because a third-party library introduced a code execution path that was never part of the intended application logic. The application trusted the library to safely handle input when in fact the library was designed to evaluate functions. This is classified as accidental misconfiguration — the developer likely used node-serialize without understanding its function evaluation behavior.

The fix was applied in DVSA-ORDER-MANAGER/order-manager.js by replacing node-serialize with JSON.parse for input handling. After the fix, the same injection payload produced no code execution on the backend, confirming the vulnerable dependency no longer poses a risk.

## Part 10) Takeaway / Lessons Learned
This lesson highlights that security is only as strong as the weakest dependency. A single unsafe third-party library can introduce a critical remote code execution vulnerability regardless of how well the rest of the application is written. In serverless environments this is especially dangerous because Lambda functions run with cloud permissions and have access to sensitive resources. The secure design principles that prevent this class of vulnerability are careful dependency selection, regular vulnerability scanning with tools like npm audit, and always preferring built-in safe alternatives over third-party libraries when they exist.
