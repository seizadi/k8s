
I used those charts as is with no customization.  I had earlier brought up the selfservice-ui-node chart as well.
With port-forward enabled for 'kratos' and the 'kratos-selfservice-ui-node', the UI comes up onmy localhost.

```bash
❯ k get pods
NAME                           READY   STATUS    RESTARTS   AGE
kratos-5dcb74448f-8jh97        1/1     Running   0          21h
kratos-ui-7958955697-kpbzp     0/1     Running   0          7s
mailslurper-796bbd89bf-88s49   1/1     Running   0          46h
```

```bash
kubectl port-forward kratos-5dcb74448f-8jh97 4433:4433

kubectl port-forward kratos-ui-7958955697-kpbzp 4455:3000
```
and then on a local browser window:

[http://127.0.0.1:4433](http://127.0.0.1:4433)

I got 404 Error, instead I tried to login to ui-node

[http://127.0.0.1:4455](http://127.0.0.1:4455)
It does redirect and then ends up with DNS address instead of localhost:4455

```bash
GET /self-service/login/browser 302 46 - 1.039 ms
GET / 302 66 - 5.154 ms
No flow ID found in URL, initializing login flow.
```

The DNS address in browser is:
```bash
Request URL: http://127.0.0.1:4455/ 302 Redirect to: /auth/login

Request URL: http://127.0.0.1:4455/auth/login 302 Redirect to: http://kratos-browserui/self-service/login/browser
```
Looks like we need to fix configuration on ui-node :(
Here are the values that need to be set from:

```bash
# Set this to ORY Kratos's Admin URL
kratosAdminUrl: "http://kratos-admin"

# Set this to ORY Kratos's public URL
kratosPublicUrl: "http://kratos-public"

# Set this to ORY Kratos's public URL accessible from the outside world.
kratosBrowserUrl: "http://kratos-browserui"
```

to:

```bash
# Set this to ORY Kratos's Admin URL
kratosAdminUrl: "http://127.0.0.1:4434"

# Set this to ORY Kratos's public URL
kratosPublicUrl: "http://127.0.0.1:4433"

# Set this to ORY Kratos's public URL accessible from the outside world.
kratosBrowserUrl: "http://127.0.0.1:4455"
```

This fixed the redirect to kratos-browserui, but now it redirects in a loop and stops.
The loop is:

```bash
http://127.0.0.1:4455 -> http://127.0.0.1:4455/auth/login -> http://127.0.0.1:4455/self-service/login/browser +
        ^                                                                                                     |
        |                                                                                                     |
        +-----------------------------------------------------------------------------------------------------+

```

I changed the values to the following to fix the issue:

```bash
# Set this to ORY Kratos's Admin URL
kratosAdminUrl: "http://kratos:4434"

# Set this to ORY Kratos's public URL
kratosPublicUrl: "http://kratos:4433"

# Set this to ORY Kratos's public URL accessible from the outside world.
kratosBrowserUrl: "http://127.0.0.1:4433"
```

Now I get SQL server error:

```bash
{"error":{"code":500,"status":"Internal Server Error","message":"sqlite create: no such table: selfservice_errors"}}
```

Look like it expects a sqlite server to be configured!

The problem was that the database was not migrated, set the configuration to auto-migrate.

```bash
  autoMigrate: true
```

Now I get error with ui-node:

```bash

  "message": "getaddrinfo ENOTFOUND kratos",
  "name": "Error",
  "stack": "Error: getaddrinfo ENOTFOUND kratos\n    at GetAddrInfoReqWrap.onlookup [as oncomplete] (dns.js:64:26)",
  "config": {
    "url": "http://kratos:4433/self-service/login/flows?id=79dc365e-890c-4229-9845-95767b93dae3",
    "method": "get",
    "headers": {
      "Accept": "application/json, text/plain, */*",
      "User-Agent": "axios/0.19.2"
    },
    "transformRequest": [
      null
    ],
    "transformResponse": [
      null
    ],
    "timeout": 0,
    "xsrfCookieName": "XSRF-TOKEN",
    "xsrfHeaderName": "X-XSRF-TOKEN",
    "maxContentLength": -1
  },
  "code": "ENOTFOUND"
}
```

The problem is that the DNS name for service is not set properly for kubernetes, so change them to proper values:

For ui-node:

```bash
# Set this to ORY Kratos's Admin URL
kratosAdminUrl: "http://kratos-admin:4434"

# Set this to ORY Kratos's public URL
kratosPublicUrl: "http://kratos-public:4433"
```

For Kratos config:
```bash
      admin:
        port: 4434
        base_url: http://kratos-admin:4434/
```
