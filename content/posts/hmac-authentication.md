---
title: HMAC Authentication
date: 2014-11-15 20:47:21
---

Recently I was working on yet another startup project (mobile app with backend). The goal was to make the backend stateless and highly scalable. I did not want to maintain sessions. So I was looking for authentication and authorization mechanism for RESTful APIs which is scalable, secure and appropriate for mobile. The solution was HMAC. This post describes how to handle it using [spray.io](http://spray.io).

<!--more-->

## Theory

The main constraint of REST web APIs is stateless client-server communication. The client is required to provide all the information necessary to handle the request. The most simple way to handle the authentication and authorization in RESTful APIs is HTTP basic authentication. In this approach, username and password are provided as a header. Thus, this method is insecure. Hash-based message authentication code (HMAC) provides more secure mechanism by usage of a hash-based code instead of password.

In HMAC, authentication based on user identifier and hash code which is calculated and attached to each request. The HTTP method, URL and abridged timestamp are combined and hashed using user's secret. The user's secret is stored on client and server side. The server builds own hash using the same variables and compare it with given one. See [RFC 2104](https://tools.ietf.org/html/rfc2104) for more details.

``` bash
hash = HMAC_SHA256('mysecret123', 'GET+/profile+1416157')

GET /profile
HTTP/1.1
Host: test.com
Authentication: hmac <uuid>:<hash>
```

## Example

In one of my projects, I developed HMAC implementation for [spray.io](http://spray.io). The sources can be found at [spray-hmac](https://github.com/mbilski/spray-hmac).

### Server side

 The library provides a directive for HMAC authentication. The user is required to extend *Authentication* trait and implement *accountAndSecret* method, which is supposed to return tuple of account and account's secret for given account identifier.

```
trait Signer {
  def valid(hash: String, secret: String, uri: String): Boolean
  def generate(secret: String, uri: String, timestamp: Long): String
}

case class HmacData(uuid: String, hash: String)

trait Authentication[A] { this: Signer =>
  def authenticate(hmac: HmacData, uri: String): Option[A] = {
    val (account, secret) = accountAndSecret(hmac.uuid)
    for (a <- account; s <- secret if valid(hmac.hash, s, uri)) yield a
  }

  def accountAndSecret(uuid: String): (Option[A], Option[String])
}
```

```
import mbilski.spray.hmac.Directives

case class Account(email: String, secret: String)

val myRoute = path("api") {
  Directives.authenticate[Account] { account =>
    get
      complete(account.email)
    }
  }
}
```

### Client side

In addition to the library, I shared also sample client implementation for AngularJS [spray-hmac-angularjs-example](https://github.com/mbilski/spray-hmac-angularjs-example). The factory HMAC is used to build the hash based on secret, method, path and time.

``` javascript
.factory('HMAC', function($rootScope) {
  var hmac = {
    hash: function(secret, method, path) {
      var time = Math.floor(new Date().getTime() / Math.pow(10, 5));
      var hash = CryptoJS.HmacSHA256(method + "+" + path + "+" + time, secret);
      return hash.toString(CryptoJS.enc.Base64);
    },
    hmac: function(uuid, hash) {
      return "hmac " + uuid + ":" + hash;
    }
  };

  return hmac;
})
```

The *$httpProvider* interceptor can be used to automatically inject HMAC header for each requests.

``` javascript
.config(function ($httpProvider) {
  $httpProvider.interceptors.push(function ($q, $rootScope, HMAC) {
    return {
      request: function(config) {
        if ($rootScope.uuid != undefined && $rootScope.secret != undefined) {
          var hash = HMAC.hash($rootScope.secret, config.method, config.url);
          config.headers['Authentication'] = HMAC.hmac($rootScope.uuid, hash);
        }
        return config || $q.when(config);
      },
      response: function(response) {
        return response || $q.when(response);
      }
    }
  });
})
```

## Summary

HMAC provides simple and scalable alternative for session based authentication and authorization. It is especially useful for mobile applications which call RESTful APIs and do not rely on OAuth2. The user's secret can be stored on a device when a user logs in, and removed when user logs out. It can be easily integrated with device management.
