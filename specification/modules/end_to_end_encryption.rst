End-to-End Encryption
=====================

.. _module:e2e:

Matrix optionally supports end-to-end encryption, allowing rooms to be created
whose conversation contents is not decryptable or interceptable on any of the
participating homeservers.

.. WARNING:
  End-to-end crypto is still being designed and prototyped. The following is
  subject to change as the design evolves. Parts of the specification are not
  yet implemented by the reference implementations.

Key Distribution
----------------
Encryption and Authentication in Matrix is based around public-key
cryptography. The Matrix protocol provides a basic mechanism for exchange of
public keys, though an out-of-band channel is required for the inital key 
exchange.

Goals
~~~~~
* No central authority: users should not need to trust a central authority
  when determining the authenticity of keys.

* Easy to add new devices: it should be easy for a user to start using a
  new device.

* Possible to discover MITM: man-in-the-middle attacks should be visible to a
  user.

* Lost devices: it should be possible for a user to recover if they lose all
  their devices.

* No copying keys: private keys should be per device and shouldn't leave the
  device they were created on.


Overview
~~~~~~~~

.. code::

    1) Bob publishes the public keys and supported algorithms for his
       device. This may include long-term identity keys, or single-use

                                          +----------+  +--------------+
                                          | Bob's HS |  | Bob's Device |
                                          +----------+  +--------------+
                                                |              |
                                                |<=============|
                                                  /keys/upload

    2) Alice requests Bob's public key and supported algorithms.

      +----------------+  +------------+  +----------+
      | Alice's Device |  | Alice's HS |  | Bob's HS |
      +----------------+  +------------+  +----------+
             |                  |               |
             |=================>|==============>|
               /keys/query        <federation>

    3) Alice selects an algorithm and claims any one-time keys needed.

      +----------------+  +------------+  +----------+
      | Alice's Device |  | Alice's HS |  | Bob's HS |
      +----------------+  +------------+  +----------+
             |                  |               |
             |=================>|==============>|
               /keys/claim         <federation>

    4) Alice sends an encrypted message to Bob.

      +----------------+  +------------+  +----------+  +--------------+
      | Alice's Device |  | Alice's HS |  | Bob's HS |  | Bob's Device |
      +----------------+  +------------+  +----------+  +--------------+
             |                  |               |              |
             |----------------->|-------------->|------------->|
               /send/             <federation>     <events>


Algorithms
----------

There are two kinds of algorithms: messaging algorithms and key algorithms.
Messaging algorithms are used to securely send messages between devices.
Key algorithms are used for key agreement and digital signatures.

Messaging Algorithm Names
~~~~~~~~~~~~~~~~~~~~~~~~~

Messaging algorithm names use the extensible naming scheme used throughout this
specification. Algorithm names that start with ``m.`` are reserved for
algorithms defined by this specification. Implementations wanting to experiment
with new algorithms are encouraged to pick algorithm names that start with
their domain to reduce the risk of collisions.

Algorithm names should be short and meaningful, and should list the primitives
used by the algorithm so that it is easier to see if the algorithm is using a
broken primitive.

The name ``m.olm.v1.curve25519-aes-sha2`` corresponds to version 1 of the Olm
ratchet, as defined by the `Olm specification`_. This uses:

* Curve25519 for the initial key agreement.
* HKDF-SHA-256 for ratchet key derivation.
* Curve25519 for the DH ratchet.
* HMAC-SHA-256 for the hash ratchet.
* HKDF-SHA-256, AES-256 in CBC mode, and 8 byte truncated HMAC-SHA-256 for authenticated encryption.

.. _`Olm specification`: http://matrix.org/docs/spec/olm.html

A name of ``m.olm.v1`` is too short: it gives no information about the primitives
in use, and is difficult to extend for different primitives. However a name of
``m.olm.v1.ecdh-curve25519-hdkfsha256.hmacsha256.hkdfsha256-aes256-cbc-hmac64sha256``
is too long despite giving a more precise description of the algorithm: it adds
to the data transfer overhead and sacrifices clarity for human readers without
adding any useful extra information.

Key Algorithms
~~~~~~~~~~~~~~

The name ``ed25519`` corresponds to the Ed25519 signature algorithm. The key is
a Base64 encoded 32-byte Ed25519 public key.

The name ``curve25519`` corresponds to the Curve25519 ECDH algorithm. The key is
a Base64 encoded 32-byte Curve25519 public key.

Uploading Keys
~~~~~~~~~~~~~~

Keys are uploaded as a signed JSON object. The JSON object must include an
ed25519 key and must be signed by that key. A device may only have one ed25519
signing key. This key is used as the fingerprint for a device by other clients.

The JSON object is signed using the process given by `Signing JSON`_.

_`Signing JSON`: https://matrix.org/docs/spec/#signing-json

.. code:: http

    POST /_matrix/client/v2_alpha/keys/upload/<device_id> HTTP/1.1
    Content-Type: application/json

    {
      "device_keys": {
        "user_id": "<user_id>",
        "device_id": "<device_id>",
        "valid_after_ts": 1234567890123,
        "valid_until_ts": 2345678901234,
        "algorithms": [
          "<chat_algorithm>",
        ],
        "keys": {
          "<key_algorithm>:<device_id>": "<key_base64>",
        },
        "signatures": {
          "<user_id>": {
            "<key_algorithm>:<device_id>": "<signature_base64>"
      } } },
      "one_time_keys": {
        "<key_algorithm>:<key_id>": "<key_base64>"
    } }

.. code:: http

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "one_time_key_counts": {
        "<key_algorithm>": 50
      }
    }


Downloading Keys
~~~~~~~~~~~~~~~~

Keys are downloaded as a collection of signed JSON objects. There
will be one JSON object per device per user. If one of the user's
devices doesn't support end-to-end encryption then their
homeserver must synthesise a JSON object without any device keys
for that device.

The JSON must be signed by both the homeserver of
the user querying the keys and by the homeserver of the device
being queried. This provides an audit trail if either homeserver
lies about the keys a user owns.

.. code:: http

    POST /keys/query HTTP/1.1
    Content-Type: application/json

    {
      "device_keys": {
        "<user_id>": ["<device_id>"]
    } }


.. code:: http

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "device_keys": {
        "<user_id>": {
          "<device_id>": {
            "user_id": "<user_id>",
            "device_id": "<device_id>",
            "valid_after_ts": 1234567890123,
            "valid_until_ts": 2345678901234,
            "algorithms": [
              "<chat_algorithm>",
            ],
            "keys": {
              "<algorithm>:<device_id>": "<key_base64>",
            },
            "signatures": {
              "<user_id>": {
                "<key_algorithm>:<device_id>": "<signature_base64>"
              },
              "<local_server_name>": {
                "<key_algorithm>:<key_id>": "<signature_base64>"
              },
              "<remote_server_name>": {
                "<key_algorithm>:<key_id>": "<signature_base64>"
    } } } } } }


Clients use ``/_matrix/client/v2_alpha/keys/query`` on their own homeservers to
query keys for any user they wish to contact. Homeservers will respond with the
keys for their local users and forward requests for remote users to
``/_matrix/federation/v1/user/keys/query`` over federation to the remote
server.


Claiming One Time Keys
~~~~~~~~~~~~~~~~~~~~~~

Some algorithms require one-time keys to improve their secrecy and deniability.
These keys are used once during session establishment, and are then thrown
away. In order for these keys to be useful for improving deniability they
must not be signed using the ed25519 key for a device.

A device must generate a number of these keys and publish them onto their
homeserver. A device must periodically check how many one-time keys their
homeserver still has. If the number has become too small then the device must
generate new one-time keys and upload them to the homeserver.

Devices must store the private part of each one-time key they upload. They can
discard the private part of the one-time key when they receive a message using
that key. However it's possible that a one-time key given out by a homeserver
will never be used, so the device that generates the key will never know that
it can discard the key. Therefore a device could end up trying to store too
many private keys. A device that is trying to store too many private keys may
discard keys starting with the oldest.

A homeserver should rate-limit the number of one-time keys that a given user or
remote server can claim. A homeserver should discard the public part of a one
time key once it has given that key to another user.


.. code:: http

    POST /keys/claim HTTP/1.1
    Content-Type: application/json

    {
      "one_time_keys": {
        "<user_id>": {
          "<device_id>": "<key_algorithm>"
    } } }

.. code:: http

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "one_time_keys": {
        "<user_id>": {
          "<device_id>": {
            "<key_algorithm>:<key_id>": "<key_base64>"
    } } } }


Clients use ``/_matrix/client/v2_alpha/keys/claim`` on their own homeservers to
claim keys for any user they wish to contact. Homeservers will respond with the
keys for their local users and forward requests for remote users to
``/_matrix/federation/v1/user/keys/claim`` over federation to the remote
server.

Sending a Message
~~~~~~~~~~~~~~~~~

Encrypted messages are sent in the form.

.. code:: json

    {
      "type": "m.room.encrypted",
      "content": {
        "algorithm": "<chat_algorithm>",
        "<algorithm_specific_keys>": "<algorithm_specific_data>"
    } }


Using Olm
+++++++++

Devices that support olm must include "m.olm.v1.curve25519-aes-sha2" in their
list of supported chat algorithms, must list a Curve25519 device key, and
must publish Curve25519 one-time keys.

.. code:: json

    {
      "type": "m.room.encrypted",
      "content": {
        "algorithm": "m.olm.v1.curve25519-aes-sha2",
        "sender_key": "<sender_curve25519_key>",
        "ciphertext": {
          "<device_curve25519_key>": {
            "type": 0,
            "body": "<base_64>"
    } } } }

The ciphertext is a mapping from device curve25519 key to an encrypted payload
for that device. The ``body`` is a base64 encoded message body. The type is an
integer indicating the type of the message body: 0 for the initial pre-key
message, 1 for ordinary messages.

Olm sessions will generate messages with a type of 0 until they receive a
message. Once a session has decrypted a message it will produce messages with
a type of 1.

When a client receives a message with a type of 0 it must first check if it
already has a matching session. If it does then it will use that session to
try to decrypt the message. If there is no existing session then the client
must create a new session and use the new session to decrypt the message. A
client must not persist a session or remove one-time keys used by a session
until it has successfully decrypted a message using that session.

The plaintext payload is of the form:

.. code:: json

   {
     "type": "<type of the plaintext event>",
     "content": "<content for the plaintext event>",
     "room_id": "<the room_id>",
     "fingerprint": "<sha256 hash of the currently participating keys>"
   }

The type and content of the plaintext message event are given in the payload.
Encrypting state events is not supported.

We include the room ID in the payload, because otherwise the homeserver would
be able to change the room a message was sent in. We include a hash of the
participating keys so that clients can detect if another device is unexpectedly
included in the conversation.

Clients must confirm that the ``sender_key`` belongs to the user that sent the
message.


A Possible Design for Group Chat using Olm
------------------------------------------

Protecting the secrecy of history
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each message sent by a client has a 32-bit counter. This counter increments
by one for each message sent by the client. This counter is used to advance a
ratchet. The ratchet is split into a vector of four 256-bit values,
:math:`R_{n,j}` for :math:`j \in {0,1,2,3}`. The ratchet can be advanced as
follows:

.. math::
    \begin{align}
    R_{2^24n,0} &= H_0\left(R_{2^24(i-1),0}\right) \\
    R_{2^24n,1} &= H_1\left(R_{2^24(i-1),0}\right) \\
    R_{2^24n,2} &= H_2\left(R_{2^24(i-1),0}\right) \\
    R_{2^24n,3} &= H_3\left(R_{2^24(i-1),0}\right) \\
    R_{2^16n,1} &= H_1\left(R_{2^16(i-1),1}\right) \\
    R_{2^16n,2} &= H_2\left(R_{2^16(i-1),1}\right) \\
    R_{2^16n,3} &= H_3\left(R_{2^16(i-1),1}\right) \\
    R_{2^8i,2}  &= H_2\left(R_{2^8(i-1),2}\right) \\
    R_{2^8i,3}  &= H_3\left(R_{2^8(i-1),2}\right) \\
    R_{i,3}     &= H_3\left(R_{(i-1),3}\right)
    \end{align}

Where :math:`H_0`, :math:`H_1`, :math:`H_2`, and :math:`H_3`
are different hash functions. For example
:math:`H_0` could be :math:`HMAC\left(X,\text{"\textbackslash x00"}\right)` and
:math:`H_1` could be :math:`HMAC\left(X,\text{"\textbackslash x01"}\right)`.

So every :math:`2^24` iterations :math:`R_{n,1}` is reseeded from :math:`R_{n,0}`.
Every :math:`2^16` iterations :math:`R_{n,2}` is reseeded from :math:`R_{n,1}`.
Every :math:`2^8` iterations :math:`R_{n,3}` is reseeded from :math:`R_{n,2}`.

This scheme allows the ratchet to be advanced an arbitrary amount forwards
while needing only 1024 hash computations.

This the value of the ratchet is hashed to generate the keys used to encrypt
each mesage.

A client can decrypt chat history onwards from the earliest value of the
ratchet it is aware of. But cannot decrypt history from before that point
without reversing the hash function.

This allows a client to share its ability to decrypt chat history with another
from a point in the conversation onwards by giving a copy of the ratchet at
that point in the conversation.

A client can discard history by advancing a ratchet to beyond the last message
they want to discard and then forgetting all previous values of the ratchet.

Proving and denying the authenticity of history
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Client sign the messages they send using a Ed25519 key generated per
conversation. That key, along with the ratchet key, is distributed
to other clients using 1:1 olm ratchets. Those 1:1 ratchets are started using
Triple Diffie-Hellman which provides authenticity of the messages to the
participants and deniability of the messages to third parties. Therefore
any keys shared over those keys inherit the same levels of deniability and
authenticity.

Protecting the secrecy of future messages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A client would need to generate new keys if it wanted to prevent access to
messages beyond a given point in the conversation. It must generate new keys
whenever someone leaves the room. It should generate new keys periodically
anyway.

The frequency of key generation in a large room may need to be restricted to
keep the frequency of messages broadcast over the individual 1:1 channels
low.
