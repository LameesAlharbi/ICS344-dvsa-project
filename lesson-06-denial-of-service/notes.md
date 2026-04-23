
# Lesson 6: Denial of Service (DoS)

## What is this?
The billing endpoint has no rate limiting. Sending 50 requests at the same time overwhelms the service and blocks real users.

## How to Reproduce
we set the API URL and token:
```bash
export API="https://nuxbyqip03.execute-api.us-east-1.amazonaws.com/dvsa/order"
export TOKEN= #we add the token 
```

we create an order and add shipping then run the flood

## What we see
Mass 500 errors the service is overwhelmed and crashes.
![Diagram](assets/BeforeFix.png)

## Fix applied
Added throttling in API Gateway:
- Go to API Gateway → DVSA-APIS → Stages → dvsa
- Set Rate: 5 requests/second, Burst: 10

![Diagram](assets/Fix1.png)
![Diagram](assets/Fix2.png)

No code changes were needed, this is a configuration fix.

## After Fix:
![Diagram](assets/AfterFix.png)


