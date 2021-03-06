# Configuring Service Discovery via .well-known

Service discovery is a way for the Matrix network to discover where a Matrix server is.

There are 2 types of well-known service discovery that Matrix makes use of:

- (important) **Federation Server discovery** (`/.well-known/matrix/server`) -- assists other servers in the Matrix network with finding your server. Without a proper configuration, your server will effectively not be part of the Matrix network. Learn more in [Introduction to Federation Server Discovery](#introduction-to-federation-server-discovery)

- (not that important) **Client Server discovery** (`/.well-known/matrix/client`) -- assists programs that you use to connect to your server (e.g. Riot), so that they can make it more convenient for you by automatically configuring the "Homeserver URL" and "Identity Server URL" addresses. Learn more in [Introduction to Client Server Discovery](#introduction-to-client-server-discovery)


## Introduction to Federation Server Discovery

All services created by this playbook are meant to be installed on their own server (such as `matrix.<your-domain>`).

As [per the Server-Server specification](https://matrix.org/docs/spec/server_server/r0.1.0.html#server-discovery), to use a Matrix user identifier like `@<username>:<your-domain>` while hosting services on a subdomain like `matrix.<your-domain>`, we need to instruct the Matrix network of such a delegation/redirection by means of setting up a `/.well-known/matrix/server` file on the base domain (`<your-domain.com>).

We have discussed this same thing already in the "`_matrix._tcp` SRV record setup (temporary requirement)" section of [Configuring DNS](configuring-dns.md).

In short, you are required to set up both a `_matrix._tcp` DNS SRV record and the `/.well-known/matrix/server` file at the moment.

As the Synapse server progresses towards v1.0, only the `/.well-known/matrix/server` file will be used. At that future moment, you would need to remove the `_matrix._tcp` SRV record because Synapse v1.0+ will do the wrong thing if a SRV record exists. During the transitional phase (before Synapse 1.0), we do need to have both a SRV record and a `/.well-known/matrix/server` file, in order to federate correctly with v0.99 and older Synapse versions.

To learn how to set it up, read the Installing section below.


## Introduction to Client Server Discovery

Client Server Service discovery lets various client programs which support it, to receive a full user id (e.g. `@username:example.com`) and determine where the Matrix server is automatically (e.g. `https://matrix.example.com`).

This lets you (and your users) easily connect to your Matrix server without having to customize connection URLs. When using client programs that support it, you won't need to point them to `https://matrix.example.com` in Custom Server options manually anymore. The connection URL would be discovered automatically from your full username.

As [per the Client-Server specification](https://matrix.org/docs/spec/client_server/r0.4.0.html#server-discovery) Matrix does Client Server service discovery using a `/.well-known/matrix/client` file hosted on the base domain (e.g. `example.com`).

However, this playbook installs your Matrix server on another domain (e.g. `matrix.example.com`) and not on the base domain (e.g. `example.com`), so it takes a little extra manual effort to set up the file.

To learn how to set it up, read the Installing section below.


## Installing well-known files on the base domain's server

To implement the two service discovery mechanisms, your base domain's server (e.g. `example.com`) needs to support HTTPS.

To make things easy for you to set up, this playbook generates and hosts 2 well-known files on the Matrix domain's server (e.g. `https://matrix.example.com/.well-known/matrix/server` and `https://matrix.example.com/.well-known/matrix/client`), even though this is the wrong place to host them.

You have 2 options when it comes to installing the files on the base domain's server:


### (Option 1): **Copying the files manually** to your base domain's server

**Hint**: Option 2 (below) is generally a better way to do this. Make sure to go with that one, if possible.

All you need to do is:

- copy `/.well-known/matrix/server` and `/.well-known/matrix/client` from the Matrix server (e.g. `matrix.example.com`) to your base domain's server (`example.com`).

- set up the server at your base domain (e.g. `example.com`) so that it adds an extra HTTP header when serving the `/.well-known/matrix/client` file. [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), the `Access-Control-Allow-Origin` header should be set with a value of `*`. If you don't do this step, web-based Matrix clients (like Riot) may fail to work. Setting up headers for the `/.well-known/matrix/server` file is not necessary, as this file is only consumed by non-browsers, which don't care about CORS.

This is relatively easy to do and possibly your only choice if you can only host static files from the base domain's server.
It is, however, **a little fragile**, as future updates performed by this playbook may regenerate the well-known files and you may need to notice that and copy them over again.


### (Option 2): **Setting up reverse-proxying** of the well-known files from the base domain's server to the Matrix server

This option is less fragile and generally better.

On the base domain's server (e.g. `example.com`), you can set up reverse-proxying, so that any access for the `/.well-known/matrix` location prefix is forwarded to the Matrix domain's server (e.g. `matrix.example.com`).

With this method, you **don't need** to add special HTTP headers for [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) reasons (like `Access-Control-Allow-Origin`), because your Matrix server (where requests ultimately go) will be configured by this playbook correctly.

**For nginx**, it would be something like this:

```nginx
# This is your HTTPS-enabled server for DOMAIN.
server {
	server_name DOMAIN;

	location /.well-known/matrix {
		proxy_pass https://matrix.DOMAIN/.well-known/matrix;
		proxy_set_header X-Forwarded-For $remote_addr;
	}

	# other configuration
}
```

**For Apache**, it would be something like this:

```apache
<VirtualHost *:443>
	ServerName DOMAIN

	SSLProxyEngine on
	<Location /.well-known/matrix>
		ProxyPass "https://matrix.DOMAIN/.well-known/matrix"
	</Location>

	# other configuration
</VirtualHost>
```

**For Caddy**, it would be something like this:

```caddy
proxy /.well-known/matrix https://matrix.DOMAIN
```

Make sure to:

- **replace `DOMAIN`** in the server configuration with your actual domain name
- and: to **do this for the HTTPS-enabled server block**, as that's where Matrix expects the file to be


## Confirming it works

No matter which method you've used to set up the well-known files, if you've done it correctly you should be able to see a JSON file at both of these URLs:

- `https://<domain>/.well-known/matrix/server`
- `https://<domain>/.well-known/matrix/client`

You can also check if everything is configured correctly, by [checking if services work](maintenance-checking-services.md).
