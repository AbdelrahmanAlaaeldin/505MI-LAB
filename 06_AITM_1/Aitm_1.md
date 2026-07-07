## Tools

- Burp Suite Community Edition v2025.12.5

### Proxy Setup:

- Instead of standard end-to-end HTTPS, an SSL strip forces the victim to use plain HTTP while keeping only the backend connection to the server on TLS.
- To enable SSL stripping in Burp, apply the following settings: - Settings → Tools → Proxy → Proxy listeners → Edit → Request handling - **Redirect to port:** `443` - **Force use of TLS:** Yes - In **Response Modification Rules**, check the boxes: - Convert HTTPS links to HTTP - Remove secure flag from cookies
  ![[AITM_1_1.png]]
  ![[AITM_1_2.png]]

## Case A:

- As a site that does not use **Strict Transport Security**, we can analyze [example.com](example.com)
- We can show that by executing `curl` command.
  ![[AITM_1_3.png]]

### The attack goes as follows:

- Go to `http://example.com`
- Burp intercepts the traffic, pulling the data from the server over TLS and handing it over plain HTTP
- Before forwarding, modify the response body by changing the header and modify for example the heading to prove the tampering worked:

- ![[AITM_1_4.jpeg]]

- ![[AITM_1_5.jpeg]]

- ![[AITM_1_6.jpeg]]

## Case B:

- As a site that does use **Strict Transport Security**, we can analyze [secure.com](secure.com)
- We can show that by executing `curl` command.
  ![[AITM_1_7.png]]

- Confirm the browser is holding an HSTS policy for the host, at `chrome://net-internals/#hsts`:
  ![[AITM_1_8.png]]

- Now go to `http://www.secure.com` with the same proxy setup. The browser **upgrades the request to HTTPS itself, before anything leaves it** and the real site loads:
  ![[AITM_1_9.png]]

### The First Visit Window:

- Since HSTS needs a prior visit to activate, a first-time user defaults to HTTP, leaving them open to an SSLStrip attack. We can recreate this scenario by wiping the domain security policy under `net-internals` to give the browser a fresh slate.
- Also checking the DNS records we can see that this site does not have an HTTPS record which will make the browser upgrade the connection even if it is not included in the HSTS list.
  ![[AITM_1_10.png]]
- Now http://www.secure.com loads over plain HTTP and is not secure:
  ![[AITM_1_11.png]]
