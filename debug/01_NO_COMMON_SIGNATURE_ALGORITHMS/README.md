# 01: NO_COMMON_SIGNATURE_ALGORITHMS

(2023-05-19)

I ran into the error `NO_COMMON_SIGNATURE_ALGORITHMS`:

> [2023-05-19 22:03:12.236][62180][debug][connection] [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:232] [C0] remote address:127.0.0.1:53344,TLS error: 268435709:SSL routines:OPENSSL_internal:NO_COMMON_SIGNATURE_ALGORITHMS
> [2023-05-19 22:03:12.236][62180][debug][connection] [external/envoy/source/common/network/connection_impl.cc:250] [C0] closing socket: 0
> [2023-05-19 22:03:12.236][62180][debug][connection] [external/envoy/source/extensions/transport_sockets/tls/ssl_socket.cc:232] [C0] remote address:127.0.0.1:53344,TLS error: 268435709:SSL routines:OPENSSL_internal:NO_COMMON_SIGNATURE_ALGORITHMS

And I got a hint from this file `envoy/test/extensions/transport_sockets/tls/integration/ssl_integration_test.cc`:

```cpp
// Server has only an ECDSA certificate, client is only RSA capable, leads to a connection fail.
// Test the access log.
TEST_P(SslCertficateIntegrationTest, ServerEcdsaClientRsaOnlyWithAccessLog) {
  useListenerAccessLog("DOWNSTREAM_TRANSPORT_FAILURE_REASON=%DOWNSTREAM_TRANSPORT_FAILURE_REASON% "
                       "FILTER_CHAIN_NAME=%FILTER_CHAIN_NAME%");
  server_rsa_cert_ = false;
  server_ecdsa_cert_ = true;
  initialize();
  auto codec_client =
      makeRawHttpConnection(makeSslClientConnection(rsaOnlyClientOptions()), absl::nullopt);
  EXPECT_FALSE(codec_client->connected());

  auto log_result = waitForAccessLog(listener_access_log_name_);
  if (tls_version_ == envoy::extensions::transport_sockets::tls::v3::TlsParameters::TLSv1_3) {
    EXPECT_THAT(log_result,
                StartsWith("DOWNSTREAM_TRANSPORT_FAILURE_REASON=TLS_error:_268435709:SSL_routines:"
                           "OPENSSL_internal:NO_COMMON_SIGNATURE_ALGORITHMS"));
  } else {
    EXPECT_THAT(log_result, StartsWith("DOWNSTREAM_TRANSPORT_FAILURE_REASON=TLS_error:_268435640:"
                                       "SSL_routines:OPENSSL_internal:NO_SHARED_CIPHER"));
  }
}
```

The comment says the server "has only an ECDSA certificate" but the client "is only RSA capable". So I'll need to check a few things:
- What was the certificate I made for the server?
- Is `curl` only RSA capable?

A quick search seems to reveal that `curl` does not quite support ECDSA certificates:
- [curl does not support ECDSA certificates](https://bugzilla.redhat.com/show_bug.cgi?id=1058776)
- [ECDSA certificates fail validation with curl and Firefox](https://serverfault.com/q/932983)