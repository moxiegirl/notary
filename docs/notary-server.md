<!--[metadata]>
+++
title = "Notary Server"
description = "Description of the Notary Server"
keywords = ["docker, notary, notary-server"]
[menu.main]
parent="mn_notary"
+++
<![end-metadata]-->

# Notary server architecture

The Notary Server secures the interaction between a client and a  Docker registry. This page describes the components in the Notary server that secure these communications.

## Overview of Notary interactions

The Notary Server stores and updates a repository's signed TUF (The Update Framework) metadata files. TUF support Notary's system for managing image security. Metadata files are generated and signed by the client at upload time. These files include root, snapshot, and targets files.  

**REVIEWER**: Are the clients uploading images as well with the metadata --- I think so -- any problem with mention that here?  What kind of conflicts can happen? Are conflicts possible the first time an image is uploaded? Is the very first image upload different than the subsequent uploads?  Is the Notary Signer pluggable? The original language implied it.

When clients upload images they also upload these metadata files. The Notary
server checks the metadata files for conflicts and then verifies both the
signatures and keys. If the checks pass, the server generates and signs a
timestamp metadata file for the repository. The timestamp metadata file
certifies that the clients' uploaded files are the most recent for that
repository.

A remote key storage/signing service makes the timestamp creation and signing
possible. You'll learn more about the key storage/signing service later in this
document.


### Authentication

Notary Server supports authentication from clients using JSON Web Tokens (JWT).
JWT is an open, industry standard method for representing claims securely
between two parties.  JWT requires an authorization server to manage access
controls. A certificate bundle from this authorization server contains the
public key the server uses to sign tokens. A Notary Server is configured to trust signatures from this authorization server.

**REVIEWER**: Under what circumstances would Notary Server have token authentication *disabled*?  If a client makes a request and is redirected to JWT, do they have reissues the first request?

If token authentication is enabled on Notary Server, then any client that does
not have a token is redirected to the JWT authorization server. The client must
log in to the authorization server, obtain a token, and present that token to
Notary Server on future requests.

For more information about **XXXXX**, please see the docs for [Docker Registry
v2 authentication](
https://github.com/docker/distribution/blob/master/docs/spec/auth/token.md)

### Server storage

Notary Server uses MySQL as a backend for storing the timestamp public keys and
the TUF metadata for each repository.  It relies on a signing service to store
the private keys.

**REVIEWER**: Is the Server storage backend configurable?

### Signing service

**REVIEWER**: Which private keys are

You can store the private keys on the XXXX server or you can configure the
Notary Server to use a separate, remote signing service. Using a remote signing
service is preferred over storing the keys on the server itself. By default,
Docker users the Notary Signer signing service.

This signing service generates and stores the timestamp private keys and
manages signing for the Notary server. Notary Signer supports mutual authentication. This mean you need to generate generate client
certificates for your deployment of Notary Server.  These client certificates **are not CAs**. Were they CAs, a compromised Notary server could sign any number of other client certs.

To see how to generate client SSL certs with basic constraints using OpenSSL, see [this OpenSSL certificate generation script](opensslCertGen.sh) .

## How to configure and run Notary Server

You use a JSON configuration file to configure Notary Server. In this file you configure:

* the server's address and signatures
* the signing service (optional)
* storage backend
* authentication service
* logging and reporting

A full configuration file looks like the following:


```json
{
	"server": {
		"http_addr": ":4443",
		"tls_key_file": "./fixtures/notary-server.key",
		"tls_cert_file": "./fixtures/notary-server.crt",
	},
	"trust_service": {
		"type": "remote",
		"hostname": "notarysigner",
		"port": "7899",
		"key_algorithm": "ecdsa",
		"tls_ca_file": "./fixtures/root-ca.crt",
		"tls_client_cert": "./fixtures/notary-server.crt",
		"tls_client_key": "./fixtures/notary-server.key"
	},
	"storage": {
		"backend": "mysql",
		"db_url": "user:pass@tcp(notarymysql:3306)/databasename?parseTime=true"
	},
	"auth": {
		"type": "token",
		"options": {
			"realm": "https://auth.docker.io/token",
			"service": "notary-server",
			"issuer": "auth.docker.io",
			"rootcertbundle": "/path/to/auth.docker.io/cert"
		}
	},
	"logging": {
		"level": "debug"
	},
	"reporting": {
		"bugsnag": {
			"api_key": "c9d60ae4c7e70c4b6c4ebd3e8056d2b8",
			"release_stage": "production"
		}
	}
}
```
**REVIEWER**: Where do I do this override on the client end? Why would I override the configuration?

 Once the Notary server is configured, you can override the configuration
parameters by setting environment variables in the form `NOTARY_SERVER_var`.
Where `var` is the ALL-CAPS, `"_"`(underscore) delimited full path of the configuration's JSON keys.  For instance, consider the storage URL of
the Notary Server configuration:

```json
"storage": {
	"backend": "mysql",
	"db_url": "dockercondemo:dockercondemo@tcp(notary-mysql)/dockercondemo"
}
```

The full JSON path of the keys is `storage -> db_url`. So the environment
variable you'd need to set would be `NOTARY_SERVER_STORAGE_DB_URL`.  For example, if running the binary:

```
$ export NOTARY_SERVER_STORAGE_DB_URL=myuser:mypass@tcp(my-db)/dbname?parseTime=true
$ NOTARY_SERVER_LOGGING_LEVEL=info notary-server -config /path/to/config.json
```

You can only override keys whose values are strings or numbers. You cannot
override a key whose value is another map. For instance, setting
`NOTARY_SERVER_STORAGE='{"storage": {"backend": "memory"}}'` does not set
in-memory storage.  It fails to parse.  

## Running Notary Server

**Reviewer**: There is something missing here. Who runs Notary Server? What kind of server does it need if any. Does Docker need to be running (looks like yes) Does it need to be replicated -- what happens if it goes down?

You run Notary Server from ####. The Notary Server is a Docker image you run on a server in your network.  

- `-config=<config file>` - The JSON configuration file.

- `-debug` - Passing this flag enables the debugging server on `localhost:8080`.
	The debugging server provides [pprof](https://golang.org/pkg/net/http/pprof/)
	and [expvar](https://golang.org/pkg/expvar/) endpoints.


Get the official Docker image, which comes with [some sane defaults](
https://github.com/docker/notary/blob/master/fixtures/server-config-local.json),
which include a remote trust service but not (?) local in-memory backing store.

You can override the default configuration with environment variables.
For example, if you wanted to run it with just a local signing service instead
(not recommended for production):

```
$ docker pull docker.io/docker/notary-server
$ docker run -p "4443:4443" \
	-e NOTARY_SERVER_TRUST_SERVICE_TYPE=local
	notary-server
```

Alternately, you can run the image with your own configuration file entirely.
You just need to mount your configuration directory, and then pass the path to
that configuration file as an argument to the `docker run` command:

```
$ docker run -p "4443:4443" \
	-v /path/to/config/dir/on/host:/etc/docker/notary-server/ \
	notary-server -config=/etc/docker/notary-server/config.json
```

You can also pass the `-debug` flag to the container in addition to the
configuration file, but the debug server port is not exposed by the container.
In order to view the debug endpoints, you will have to `docker exec` into
your container.

### What happens if the server is compromised?

The server does not hold any keys for repositories, except the for timestamp
keys if you are using a local signing service, so the attacker cannot modify
the root, targets, or snapshots metadata.

If you are using a signer service, an attacker cannot get access to the
timestamp key either. They can use the server's credentials to get the signer
service to sign arbitrary data, such as an empty timestamp,
an invalid timestamp, or an old timestamp.

However, TOFU (trust on first use) would prevent the attacker from tricking
existing clients for existing repositories to download arbitrary data.
They would need the original root/target/snapshots keys to do that. The
attacker could only, by signing bad timestamps, prevent the such a user from
seeing any updated metadata.

The attacker can also make all new keys, and simply replace the repository
metadata with metadata signed with these new keys.  New clients who have not
seen the repository before will trust this bad data, but older clients will
know that something is wrong.

### Ops features

Notary Server provides the following features for operational friendliness:

1. A health endpoint at `/_notary_server/health` which returns 200 and a
	body of `{}` if the server is healthy, and a 500 with a map of
	failed services if the server cannot access its storage backend.

	If it cannot contact the signing service, an error will be logged but the
	service will still be considered healthy, because it can still serve
	existing metadata.  It cannot accept updates, so the service is degraded.

1. A [Bugsnag](https://bugsnag.com) hook for error logs, if a Bugsnag
	configuration is provided.

1. A [prometheus](http://prometheus.io/) endpoint at `/_notary_server/metrics`
	which provides HTTP stats.


## Related information

* TUF metadata files](
https://github.com/theupdateframework/tuf/blob/develop/docs/tuf-spec.txt#L348)
* [Notary Signer](notary-signer.md)
* [JWT](http://jwt.io/)
[Docker Registry v2 authentication](
https://github.com/docker/distribution/blob/master/docs/spec/auth/token.md)
As an example, please see [this script](opensslCertGen.sh) to see how to
generate client SSL certs with basic constraints using OpenSSL.
Please see the
[Notary Server configuration document](notary-server-config.md)
for more details about the format of the configuration file.
