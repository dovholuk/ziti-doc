# Edge-to-edge encryption

Edge-to-edge encryption (commonly referred to as end-to-end encryption or E2Ee) ensures that data is encrypted at the
source and only decrypted by the intended recipient. This ensures that data remains secure even if the underlying
transport or physical infrastructure is compromised along the communication path.

## The role of communications security

Communications Security (COMSEC) is the discipline of preventing unauthorized interception of communications. In a
traditional network, security is often fragmented across different layers. OpenZiti focuses primarily on cryptographic
security, which serves as a powerful mitigant for weaknesses in other areas, such as physical or transmission security.

For example, if a transmission link is compromised, cryptographic security ensures that the snooped packets remain
unreadable and unalterable by the attacker.

### Link encryption vs. end-to-edge encryption

It's important to distinguish between *link encryption* and *end-to-edge encryption*:

- **Link encryption**: Encryption occurs between two network routing points, such as TLS between a client and a router.
  While secure, a determined attacker who compromises a router could potentially capture or alter messages while they
  are momentarily decrypted in the router's memory.
- **End-to-edge encryption**: Data is encrypted within the OpenZiti SDK at the source and remains encrypted throughout
  the entire OpenZiti Fabric. It's only decrypted by the destination SDK, meaning even a compromised OpenZiti router
  can't "see" the raw data.

## Technical requirements and implementation

OpenZiti's encryption implementation is designed to be lightweight, avoiding additional network round trips while
maintaining industry-standard security.

### The libsodium library

Rather than implementing custom cryptography, NetFoundry uses [libsodium](https://doc.libsodium.org/), a widely audited,
open-source library used by major platforms like Discord and WireGuard. This provides cross-platform support across all
OpenZiti SDK flavors, including Go, C, Java, and Android.

### Encryption algorithms

OpenZiti uses the XChaCha20-Poly1305 cipher.

- **Performance**: ChaCha20 is significantly faster than AES-256 on devices without hardware-accelerated AES, like
  mobile phones and IoT edge devices.
- **Strength**: It provides 256-bit security, which is equivalent to the strength of AES-256.

:::note
The "X" in XChaCha20-Poly1305 denotes an *eXtended-nonce* (192-bit vs. the standard 96-bit). This allows
OpenZiti to safely use randomly generated nonces for every message without the risk of collision, making it much more
resilient in distributed, stateless environments where central counter management is impractical.
:::

## How the OpenZiti key exchange works

OpenZiti leverages established trust via an ephemeral key exchange to bootstrap a secure session without adding latency.

1. **Server-side binding**: When an SDK server binds to a service, it generates an ephemeral key pair and provides its
   public key to the controller.
2. **Client-side dialing**: When a client dials that service, it generates its own ephemeral key pair and sends its
   public key to the controller.
3. **Controller routing**: The controller routes the request, swapping the public keys between the client and server.
4. **Key derivation**: Both parties derive symmetric session keys from their own private keys and the received public
   key.

### Message exchange and forward secrecy

For every message sent, OpenZiti performs the following:

- **Encryption and authentication**: Data is encrypted and a [message authentication code
  (MAC)](https://grokipedia.com/page/Message_authentication_code) is appended to ensure integrity.
- **Key rotation**: The transmission (tx) and reception (rx) keys are rotated immediately after every message. This
  provides [perfect forward secrecy](https://grokipedia.com/page/Forward_secrecy), ensuring that if a future key is
  somehow compromised, past sessions remain secure.

## Performance and overhead

Encryption introduces a negligible amount of overhead to ensure security:

- **Session start**: One additional 24-byte message is sent at the beginning of a session.
- **Per message**: 17 bytes are added to each message.
