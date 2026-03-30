---
id: dns
topic: DNS and Host in Dev
category: Design & Architecture
---

Below is a ready-to-study version with:

* **interview answer**
* **simple example**
* **extra note** for real-world understanding

---

# 1. What is DNS?

 

**DNS** stands for **Domain Name System**.
It is the system that translates a human-readable name such as `example.com` into an IP address such as `93.184.216.34`.

Humans prefer names, but computers communicate using IP addresses. DNS acts like the internet’s phonebook and helps clients find the correct server.

## Simple example

You type:

```text
https://api.myapp.com
```

Your browser does not automatically know where `api.myapp.com` is.
It asks DNS for the IP address of that host.
DNS returns the IP, then the browser connects to that server.

## Extra note

Without DNS, users would need to remember raw IP addresses instead of domain names.

---

# 2. How DNS resolution works

 

DNS resolution is the process of converting a domain or hostname into an IP address.

A simplified flow is:

1. The browser checks its own cache
2. The OS may check its cache
3. The system may check the local hosts file
4. The request goes to a DNS resolver, usually provided by the ISP or network
5. The resolver finds the answer, possibly by asking other DNS servers
6. The IP address is returned to the client
7. The browser connects to that IP and sends the HTTP request

## Simple example

A user enters:

```text
https://shop.example.com/products
```

Typical sequence:

* Browser sees host = `shop.example.com`
* DNS lookup happens
* DNS returns something like `203.0.113.10`
* Browser opens a connection to `203.0.113.10`
* Browser sends HTTP request for `/products`


---

# 3. Difference between domain / hostname / host / IP

 

These terms are related but different:

* **Domain**: the main naming space, such as `example.com`
* **Hostname**: a specific name assigned to a machine or service, such as `api.example.com`
* **Host**: depending on context, it can mean either:

  * the hostname in a URL, or
  * the server/machine running the application
* **IP address**: the numeric network address used by computers, such as `192.168.1.10`

## Simple example

For this URL:

```text
https://api.example.com/orders
```

* Domain: `example.com`
* Hostname: `api.example.com`
* Host in URL: `api.example.com`
* IP: something like `203.0.113.10`


---

# 4. A record vs CNAME

 

An **A record** maps a domain or hostname directly to an IPv4 address.

A **CNAME record** maps one domain name to another domain name.

So:

* **A record**: name -> IP
* **CNAME**: name -> another name

## Simple example

### A record

```text
api.example.com -> 203.0.113.10
```

### CNAME

```text
www.example.com -> app-host.azurewebsites.net
```

Then DNS will resolve `app-host.azurewebsites.net` further to an IP.

## Extra note

CNAME is common when pointing a custom domain to a cloud-hosted service, because the cloud provider’s target hostname may change its IP behind the scenes.

---

# 5. What localhost means

 

`localhost` is a special hostname that refers to the current machine.

It usually resolves to:

```text
127.0.0.1
```

for IPv4, or:

```text
::1
```

for IPv6.

When developers run an app on `localhost`, they are accessing a service running on their own computer.

## Simple example

```text
https://localhost:5001
```

This usually means your local ASP.NET Core app is running on your machine on port `5001`.


---

# 6. What the hosts file does

 

The **hosts file** is a local operating system file that manually maps hostnames to IP addresses.

It can override DNS for that machine.

This is useful for local testing, development, or temporarily redirecting a domain to a different IP.

## Simple example

A hosts file entry like:

```text
127.0.0.1 myapp.local
```

means when your machine tries to access:

```text
http://myapp.local
```

it will go to your own computer instead of performing a public DNS lookup.



---

# 7. What the HTTP Host header is

 

The **Host header** is an HTTP request header that tells the server which host the client is trying to access.

This is important because one server or IP address can host multiple websites. The server uses the Host header to determine which site or application should handle the request.

## Simple example

Request:

```http
GET /products HTTP/1.1
Host: api.example.com
```

The server sees the `Host` header and knows the request is intended for `api.example.com`.


---

# 8. How multiple domains can share one IP

 

Multiple domains can share one IP through **virtual hosting**.

This works because:

* DNS can point multiple hostnames to the same IP
* the client includes the **Host header**
* the web server or reverse proxy uses that Host header to route the request to the correct site

## Simple example

All these hostnames may resolve to the same IP:

```text
api.example.com
admin.example.com
shop.example.com
```

But Nginx or Apache can route them differently:

* `api.example.com` -> API app
* `admin.example.com` -> admin site
* `shop.example.com` -> storefront


---

# 9. What TTL and DNS caching are

 

**TTL** stands for **Time To Live**.

It tells DNS resolvers and clients how long they are allowed to cache a DNS record before checking again.

**DNS caching** means storing DNS lookup results temporarily so future requests are faster and do not need a full DNS lookup every time.

## Simple example

Suppose a DNS record has:

```text
TTL = 3600
```

That means the result may be cached for **3600 seconds**, or **1 hour**.

If the IP changes during that hour, some users may still go to the old IP until the cached entry expires.


---

# 10. How to troubleshoot DNS issues in real projects

 

When troubleshooting DNS issues check:

1. Whether the DNS record exists
2. Whether it points to the correct target
3. Whether DNS propagation or caching is delaying the change
4. Whether the app/server is actually listening for that domain
5. Whether HTTPS/certificate configuration is also correct
6. Whether the issue is DNS or something else such as firewall, reverse proxy, or application config

Useful tools include:

* `nslookup`
* `dig`
* `ping` for basic connectivity name resolution
* browser developer tools
* cloud portal DNS settings
* server/reverse proxy logs

## Simple example

Scenario:

* `api.example.com` should point to your Azure App Service
* users say the domain does not work

Possible troubleshooting steps:

### Step 1: Check DNS

Run:

```bash
nslookup api.example.com
```

See if it resolves to the expected target or IP.

### Step 2: Check record type

Confirm whether you configured:

* A record
* CNAME
* TXT verification record if required by provider

### Step 3: Check caching / TTL

If you changed DNS recently, some users may still see old values.

### Step 4: Check hosting config

Make sure the app service or reverse proxy is configured to accept that host name.

### Step 5: Check HTTPS

If DNS is correct but HTTPS fails, the certificate may be missing or not bound correctly.

## Extra note

A very important real-world rule:

> A domain resolving correctly does not guarantee the app itself is configured correctly.

Sometimes DNS is fine, but the issue is:

* wrong SSL certificate
* missing host binding
* reverse proxy misconfiguration
* app not running
* firewall/network block

---

# Multiple domains share one IP

On Azure, the usual way to handle sharing the **same public IP** is to use **host-based routing**. The DNS for all three names can point to the same Azure entry point, and Azure then looks at the **HTTP Host header** to decide which backend app should receive the request. Azure Application Gateway explicitly supports multiple-site hosting on the same public IP and port by relying on the host header, and Azure Front Door also supports multiple custom domains with routing rules.

## The basic idea

All three DNS records can point to the same Azure-facing endpoint:

```text
api.example.com   -> same Azure public IP or same Azure frontend host
admin.example.com -> same Azure public IP or same Azure frontend host
shop.example.com  -> same Azure public IP or same Azure frontend host
```

Then Azure routes like this:

* requests with host `api.example.com` -> API app
* requests with host `admin.example.com` -> admin app
* requests with host `shop.example.com` -> shop app

That works because the browser sends a Host header such as:

```http
GET / HTTP/1.1
Host: shop.example.com
```

and Azure’s edge or gateway can match on that host name. Application Gateway’s multi-site listeners are built for exactly this scenario.


## Two common Azure patterns

### Option 1: Azure Application Gateway

This is common when you want one public IP inside Azure and route to multiple backends.

You configure:

* one **public IP** on the Application Gateway
* one or more **multi-site listeners**
* rules that match each host name
* separate backend pools for each app if needed

Example mapping:

* `api.example.com` -> backend pool for your API App Service or VM
* `admin.example.com` -> backend pool for your admin site
* `shop.example.com` -> backend pool for your shop site

Microsoft’s docs say Application Gateway can host more than one website on the same public IP address and port by using HTTP 1.1 host headers, and each website can be directed to its own backend pool. 

A simple mental model:

```text
Internet
   |
Same public IP
   |
Azure Application Gateway
   |-- Host = api.example.com   -> API backend
   |-- Host = admin.example.com -> Admin backend
   |-- Host = shop.example.com  -> Shop backend
```

### Option 2: Azure Front Door

This is common when you want a global edge entry point, custom domains, TLS at the edge, and routing to origins.

You configure:

* custom domains in Front Door
* routes based on domain/host name
* one or more origin groups behind it

Front Door supports multiple custom domains and routing rules, and Microsoft’s guidance for multitenant solutions highlights its support for custom domains, TLS certificates, and routing capabilities.

A simple mental model:

```text
Internet
   |
Azure Front Door
   |-- api.example.com   -> API origin
   |-- admin.example.com -> Admin origin
   |-- shop.example.com  -> Shop origin
```



## Common real-world setup

A typical setup  would be:

### DNS

```text
api.example.com
admin.example.com
shop.example.com
```

All point to the same Azure frontend.

### Azure layer

Use either:

* **Application Gateway** with multi-site listeners, or
* **Front Door** with multiple custom domains and routes

### Backends

* API App Service
* Admin App Service
* Shop App Service

This gives you one public-facing entry point while still routing by hostname.

