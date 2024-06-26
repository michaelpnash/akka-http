# Server HTTPS Support

Akka HTTP supports TLS encryption on the server-side as well as on the @ref[client-side](../client-side/client-https-support.md).

The central vehicle for configuring encryption is the @apidoc[HttpsConnectionContext], which can be created using
the static method `ConnectionContext.httpsServer` which is defined like this:

Scala
:  @@snip [ConnectionContext.scala](/akka-http-core/src/main/scala/akka/http/scaladsl/ConnectionContext.scala) { #https-server-context-creation }

Java
:  @@snip [ConnectionContext.scala](/akka-http-core/src/main/scala/akka/http/javadsl/ConnectionContext.scala) { #https-server-context-creation }

On the server-side, the @apidoc[ServerBuilder] defines a method `enableHttps` with an `httpsContext` parameter,
which can receive the HTTPS configuration in the form of an `HttpsConnectionContext` instance.

For detailed documentation for client-side HTTPS support refer to @ref[Client-Side HTTPS Support](../client-side/client-https-support.md).

## Obtaining SSL/TLS Certificates

In order to run an HTTPS server a certificate has to be provided, which usually is either obtained from a signing
authority or created by yourself for local or staging environment purposes. You can either use PEM-files or Java keystores
with certificate and private key.

Signing authorities often provide instructions on how to create a Java keystore (typically with reference to Tomcat
configuration). If you want to generate your own certificates, the official Oracle documentation on how to generate a
keystore using the JDK keytool utility can be found [here](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html).

SSL-Config provides a more targeted guide on generating certificates, so we recommend you start with the guide
titled [Generating X.509 Certificates](https://lightbend.github.io/ssl-config/CertificateGeneration.html).

<a id="using-https"></a>
## Using HTTPS

Once you have obtained the server certificate, using it is as simple as preparing an @apidoc[HttpsConnectionContext]
and passing it to `enableHttps` when binding the server.

The below example shows how setting up HTTPS works.
First, you create and configure an instance of @apidoc[HttpsConnectionContext].

If you have certificate files in PEM-files (often with extensions `.pem`, `.key` and `.crt`), you can use the convenience
factories in @apidoc[SSLContextFactory$] for loading them:

Scala
:  @@snip [HttpsServerExampleSpec.scala](/docs/src/test/scala/docs/http/scaladsl/server/HttpsServerExampleSpec.scala) { #convenience-cert-loading }

Java
:  @@snip [HttpsServerExampleTest.scala](/docs/src/test/java/docs/http/javadsl/server/HttpsServerExampleTest.java) { #convenience-cert-loading }

For mTLS (mutual TLS) the trusted CA cert list needs to contain the CA that issues the certs which clients that the server trusts
connect with.

If there are more low level aspects you need to configure, or if you are loading certificates from a Java key store:

Scala
:  @@snip [HttpsServerExampleSpec.scala](/docs/src/test/scala/docs/http/scaladsl/server/HttpsServerExampleSpec.scala) { #imports #low-level-default }

Java
:  @@snip [SimpleServerApp.java](/akka-http-tests/src/main/java/akka/http/javadsl/server/examples/simple/SimpleServerApp.java) { #https-http-config }



After that you can pass it to `enableHttps`, like displayed below:

Scala
:  @@snip [HttpsServerExampleSpec.scala](/docs/src/test/scala/docs/http/scaladsl/server/HttpsServerExampleSpec.scala) { #bind-low-level-context }

Java
:  @@snip [SimpleServerApp.java](/akka-http-tests/src/main/java/akka/http/javadsl/server/examples/simple/SimpleServerApp.java) { #bind-low-level-context }

## Running both HTTP and HTTPS

If you want to run HTTP and HTTPS servers in a single application, you first create an instance of `HttpsConnectionContext` as explained above
and then create two server bindings for different ports, one with https enabled and one without:

Scala
:  @@snip [HttpsServerExampleSpec.scala](/docs/src/test/scala/docs/http/scaladsl/server/HttpsServerExampleSpec.scala) { #both-https-and-http }

Java
:  @@snip [SimpleServerHttpHttpsApp.java](/akka-http-tests/src/main/java/akka/http/javadsl/server/examples/simple/SimpleServerHttpHttpsApp.java) { #both-https-and-http }

## Mutual authentication

To require clients to authenticate themselves when connecting, using mTLS (mutual TLS), you must set this on the `SSLEngine`:

Scala
:  @@snip [HttpsServerExampleSpec.scala](/docs/src/test/scala/docs/http/scaladsl/server/HttpsServerExampleSpec.scala) { #require-client-auth }

Java
:  @@snip [HttpsServerExampleTest.scala](/docs/src/test/java/docs/http/javadsl/server/HttpsServerExampleTest.java) { #require-client-auth }

For further (custom) certificate checks, you can access the `javax.net.ssl.SSLSession` via the `sslSession` attribute.

At this point dynamic renegotiation of the certificates to be used is not implemented. For details see [issue #18351](https://github.com/akka/akka/issues/18351)
and some preliminary work in [PR #19787](https://github.com/akka/akka/pull/19787).

## Rotating certificates

It is often important to rotate the certificates without having to redeploy/restart a server. This is possible with Akka
HTTP through the @scala[`ConnectionContext.httpsServer(() => SSLEngine)`]@java[`ConnectionContext.httpsServer(akka.japi.function.Creator[SSLEngine])`]
`ConnectionContext` factory. The function passed to `httpsServer` will be invoked on each connection so can return differently
configured `SSLEngine`s over time.

The function for creating the `SSLEngine` can be manually implemented, but for convenience a few utilities are provided in the @apidoc[SSLContextFactory$] class.
Here is an example using those utilities providing a cached `SSLEngine` that is periodically reloaded to pick up updated certificates:

Scala
:  @@snip [HttpsServerExampleSpec.scala](/docs/src/test/scala/docs/http/scaladsl/server/HttpsServerExampleSpec.scala) { #rotate-certs }

Java
:  @@snip [HttpsServerExampleTest.scala](/docs/src/test/java/docs/http/javadsl/server/HttpsServerExampleTest.java) { #rotate-certs }


## Further reading

The topic of properly configuring HTTPS for your web server is an always changing one,
thus we recommend staying up to date with various security breach news and of course
keep your JVM at the latest version possible, as the default settings are often updated by
Oracle in reaction to various security updates and known issues.

We also recommend having a look at the [Play documentation about securing your app](https://www.playframework.com/documentation/2.5.x/ConfiguringHttps#ssl-certificates),
as well as the techniques described in the Play documentation about setting up a [reverse proxy to terminate TLS in
front of your application](https://www.playframework.com/documentation/2.5.x/HTTPServer) instead of terminating TLS inside the JVM, and therefore Akka HTTP, itself.

Other excellent articles on the subject:

 * [Oracle Java SE 8: Creating a Keystore using JSSE](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html#CreateKeystore)
 * [Java PKI Programmer's Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/security/certpath/CertPathProgGuide.html)
 * [Fixing X.509 Certificates](https://tersesystems.com/2014/03/20/fixing-x509-certificates/)
