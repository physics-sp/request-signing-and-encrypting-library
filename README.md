
# request-signing-implementation

This project presents a very simple webpage that implements a simple signing protocol.  

It uses Elliptic-curve Diffie-Hellman (with curve25519) to generate a shared secret  
and then signs all the request with a modified version of the AWS v4 signature scheme.

The main idea is to make request signing as easy to implement as possible.  


## Key generation
Each time a user enters the page, the server and the client generate a ECDH key-pair and then exchange public keys.  
Then they calculate the same shared secret, pass it through SHA256 and then use the result for request signing.  
Every time a request is signed, the key passes through SHA256 again.


## Signature scheme: Differences with AWS v4
#### SignKey generation
The region, service, accessKeyId and secretAccessKey are not longer used to generate the signKey.  
This signKey is generated as stated above.  


#### Removed headers
The x-amz-date header is not longer used.

#### Added headers
The x-request-id header has been added.  
This header is nothing more than a UUID, it protects the server from replay attacks. 
The server provides a new UUID in every reply, and expects to receive it back in the next request.  
If the same request is received twice, the second one will be ignored because the UUIDs will not match.

#### Authorization header
The Authorization header no longer contains the "Credential" part as it is no longer needed.

The original Authorization header looked like:  
`Authorization: AWS4-HMAC-SHA256 Credential=ASIAIQTP2FX4MJ4J2DIA/20160520/eu-west-1/execute-api/aws4_request, SignedHeaders=accept;host;x-amz-date, Signature=cc870c6ea5174baad470e46a7f5642725ff9411e049cf24d730923fca7e5f2b4`

Now it looks like the following:  
`Authorization: AWS4-HMAC-SHA256 SignedHeaders=accept;content-type;header1;host;x-request-id, Signature=ad3f3bfb9ba8a5a8149554be61a1f8c1873c2bac5a57d3129ed58f56e5d18b1e`  


Sample of a signed request:

```
POST /some/path?a=val1&c=val2&b=val3 HTTP/1.1
Host: yourdomain.com
Connection: keep-alive
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Accept: application/json
User-Agent: python-requests/2.23.0
header1: headerValue
X-Request-Id: f914f338-576e-11ea-8e2d-0242ac130003
Authorization: AWS4-HMAC-SHA256 SignedHeaders=accept;content-type;header1;host;x-request-id, Signature=ad3f3bfb9ba8a5a8149554be61a1f8c1873c2bac5a57d3129ed58f56e5d18b1e

a=1&c=3&b=2
```

## Webpage example

The web page is written in HTML and JavaScript, the backend is written in Python3.

Feel free to clone the repository and play with the requests in Burp.

### Instalation

Simply clone the repository:
```
git clone https://github.com/physics-sp/request-signing-implementation.git
```
Run the web server:
```
cd request-signing-implementation
python3 backend.py
```
And visit the webpage: `http://127.0.0.1:5000/`


## What is it NOT for
This is **NOT** a replacement for HTTPS.
This type of cryptography can't and won't protect you from man in the middle attacks.  
This is because there is no authentication provided, meaning that you don't know for sure you are talking to the server and not an attacker. 


## What is it for
This implementation will help you prevent CSRF attacks (for the request that are signed).  

It will also make exploiting XSS vulnerabilities a little harder (but not impossible).  

The main benefit, in my opinion, is that it makes it harder to tamper with the requests and find vulnerabilities in the backend server.
Especially while using automated tools such as SQLMap.  

An experienced attacker will certainly be able to bypass this control.  

This is mostly a learning experience, but this type of protections *should* be easy to implement, but now a days there are no great libraries that implement these type of cryptography.


## Considerations
A real implementation of this library, should take into account that each session will have two values associated with the cookie(s), the shared key, and the next request id.  

The next request id should change in every request (obviously), and the shared key should ideally change over time.  
This can be done by additional Diffie Hellman key exchanges, or by a key derivation function.  

After a DH keypair is used to get a shared secret, it should be discarded and forgotten (to provide forward secrecy).

If you are planning to use a backend other than Python, take into account that there are some ECDH libraries that are not compatible with the JavaScript library, this is because some libraries represent the private/public keys in different ways.
The library you choose should encode their keys acoding to the [X25519 RFC](https://tools.ietf.org/html/rfc7748).

## Credit

Several projects where used to obtain some of the core logic (some have been modified):

### AWS v4
- https://github.com/danieljoos/aws-sign-web  (JavaScript)
- https://github.com/DavidMuller/aws-requests-auth (Python)

### ECDH with curve25519
- https://github.com/wavesplatform/curve25519-js (JavaScript)
- https://cryptography.io/en/latest/hazmat/primitives/asymmetric/x25519/ (Python)



## Ideas for the future
- Add backend implemented in PHP, Node and/or Java.
- Sign responses from the backend server.
