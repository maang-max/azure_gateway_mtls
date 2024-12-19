# azure_gateway_mtls
mTLS (Mutual TLS) with Azure Gateway

MTLS stands for Mutual Transport Layer Security. It is an extension of the Transport Layer Security (TLS) protocol that provides an additional layer of security by requiring both the client and the server to authenticate each other using digital certificates. This ensures that both parties in a communication are who they claim to be, and it helps prevent man-in-the-middle attacks.

Below is a simplified diagram illustrating the interaction between a client and a server using mTLS:

+-------------------+                        +---------------------+
|    Client         |                        |     Server          |
|                   |                        |                     |
| 1. Client Hello   | --------->           | 2. Server Hello       |
|                   |                        |  +-----+ Cert       |
|                   |                        |  | CA  |            |
|                   |                        |  +-----+            |
|                   |                        | 3. Certificate      |
|                   |                        |    Verify           |
|                   |                        | 4. Client Key       |
|                   |                        |    Exchange         |
|                   |                        | 5. Server Key       |
|                   |                        |    Exchange (if     |
|                   |                        |    needed)          |
|                   |                        | 6. Change Cipher    |
|                   |                        |    Spec             |
|                   |                        | 7. Finished         |
| 8. Change Cipher  | <---------             |                     |
|    Spec           |                        | 9. Finished         |
| 10. Application   | --------->             | 11. Application     |
|    Data           |                        |    Data             |
+-------------------+                        +---------------------+
