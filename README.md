# Connectoid

Version: 0.1.0

**Status:** early draft idea about libs and decentralized services as in SMTP.

![joke](backup.jpg)

## What Connectoid does

- Connectoid **connects** users of p2p apps by providing "signaling" layer required for WebRTC, kind of a chat for apps, which just starts as a chat of humans.
- **Simple sharing:** send a link via any messenger to another person or to yourself on another device.
- **Privacy, economy, and backup:** copies of data are stored on devices this data is shared with, not in someone's cloud, not in a public p2p network.
- **Collaboration:** when one user updates data, it gets synced to other users this data is shared with.
- **Battery saving:** apps don't keep connections when they don't need to sync.
- **Security:** each step is encrypted and only the right user can decrypt. Of course there is no such thing as absolute protection, but we do our best, read below how.

## How it works

### Share

- For example Alice uses your app.
- Alice wants to share her shopping list with Bob.
- Alice's app asks the nickname of who to share with.
- Alice enters: "Bob".
- App generates a link:

<details>
<summary>Click here to see details.</summary>

- App passes this nickname to Connectoid lib.
- Lib generates secure (ECC 384) keypair for messages from Bob to Alice. No typo here: from Bob to Alice.
- Private key is stored securely in app's data, neither visible to other apps, nor even to Alice, and never leaves this device.
- Public key is encoded with RFC 4648 base64url to e.g. `123abc`.
- URL of a Connectoid service is encoded in the same way to e.g. `456def`.
- Optional app-specific data (e.g. api key for Connectoid service) is encoded in the same way to e.g. `789ghi`.
- These 2 or 3 values are joined using dot and become `connectoid` value in the link to the app, e.g. https://app.example.com/?connectoid=123abc.456def.789ghi
- Public key is similar to username, Connectoid service URL - to domain, and optional extension - to query string, so it is tempting to use `pub@url?ext` format, but that would require URL encoding of this component, which is already in query string of real URL. Dots are safe to use in URL and are consistent with JWT format.

</details>

- App shows standard "Share" dialog.
- Alice selects a messenger they use with Bob and sends the link.
- App may also encode this link as QR code for cases when no prior messenger contact is possible and maybe users don't want to create one.

### Connect

- Bob receives and clicks this link.
- Link opens or installs and opens the app.
- Apps of Alice and Bob get connected peer to peer and sync the data:

<details>
<summary>Click here to see details.</summary>

- Bob's app passes this link to its Connectoid lib.
- Lib unpacks `connectoid` value found in the link to Alice's public key `pub`, Connectoid service `url`, and optional data `ext`.
- Lib generates `connectoid` value for messages from Alice to Bob in the same way.
- Connectoid lib gets SDP offer from WebRTC lib.
- Connectoid lib creates JSON object: `{"connectoid": "...", "sdpOffer": {...}}`
- See TODO below re signing the message and man in the middle.
- Lib encrypts this JSON string using public key of Alice.
- Lib encodes the result using the same base64url.
- Lib sends this encrypted `data` to Connectoid service, e.g:
```
curl https://connecto.id/ -d '{
"pub": "123abc",
"ext": null,
"action": "send",
"data": "vttf66efd34dttf66fg6fde57gg"
}'
```
- `pub` is a public key of Alice, because it is a message to Alice.
- Responses to expect:
    - `{"ok": true}`
    - `{"ok": false, "error": "Too often"}`
    - `{"ok": false, "error": "Too big"}`
    - Public keys and SDP offers are not that big and don't happen too often, so Connectoid service tries to defend itself from abuse and extra costs as it can afford.
- Alice's app configures Connectoid lib to check Connectoid service e.g. once per minute (depends on the app).
- Lib checks using request like this:
```
curl https://connecto.id/ -d '{
"pub": "123abc",
"ext": null,
"action": "check",
"at": 0
}'
```
- If we find such requests too heavy, we can try 1 UDP packet instead, but let's prove this is a performance bottleneck before we add complexity.
- `at` is an integer timestamp at which Alice already received and processed messages from Bob.
- Responses to expect:
    - `{"ok": true, "found": false}` - no messages are found, please try again later.
    - `{"ok": true, "found": true, "question": "iyrvchgutgr65rffy"}`
- `ok` means no errors happened and that the check is noticed.
- If there are no checks for 1 month, Connectoid service may delete the message box of this public key.
- `question` is a challenge from Connectoid service: a random string, encrypted with Alice's public key, encoded with the same base64url, stored at the Connectoid service side for 1 minute.
- Lib decodes and decrypts the `question` using Alice's private key, result is the `answer` to prove that Alice has right to receive a message.
- Lib sends another request:
```
curl https://connecto.id/ -d '{
"pub": "42abc123def456",
"ext": null,
"answer": "hghy64egyu7yguy5"
"action": "receive",
"at": 0,
"limit": 1
}'
```
- `limit` is how many messages we want to receive.
- It is safe to bulk process messages, as they are deleted after 1 hour, not after being received, because an app could fail to process them. That's why `at` is updated after the app processes each event.
- There is no sense to increase this 1 hour to something bigger, because messages contain SDP offers, which expire quickly, e.g. when external IP address changes.
- Also bigger limit would expose a Connectoid service to bigger costs to store messages.
- Response: `{"ok": true, "at": 20221231235959123456, "data": "vttf66efd34dttf66fg6fde57gg"}`
- Alice's lib decrypts the `data` using private key.
- Connectoid lib gets SDP offer and passes it to WebRTC lib.
- WebRTC lib generates SDP answer.
- Alice's Connectoid lib sends this answer to Bob's Connectoid lib in the same way.
- Bob's lib receives this answer in the same way.
- Bob's Connectoid lib passes SDP answer to WebRTC lib.
- Apps get connected peer to peer.
- Connectoid libs send challenge-response to each other directly in the same way to be completely sure.
- Libs generate and start using new public key of Alice to prevent other people from reusing this link.
- If bad person Eve manages to intercept the link from a private chat between Alice and Bob and to use it before Bob did, then Bob will fail to use this link and his lib/app will show an error that the link was intercepted.
- The same can happen to any one time code, verification email, etc.
- Apps sync their data directly in their own app-specific way, e.g. for each `id` the latest `updatedAt` timestamp wins, or read-only sharing, etc.
- To reduce number of connections, N apps can agree on chain or circle routing as in DHT. This can become a feature of next version of Connectoid lib, not in MVP.

</details>

- Apps disconnect to save batteries.
- When Alice or Bob updates the data, their app automatically reconnects with another app, syncs the data and disconnects again.

### Unshare

- On the first connect with Alice, Bob's app also asked the nickname of the other side.
- Bob entered: "Alice".
- Now both Alice and Bob have each other in "Shared" list of this app.
- Each nickname in this list has "Unshare" button which deletes related keypair on both sides, stopping the sync.
- Bob's app may also delete the data, which is no longer shared with Bob.

## That's all for now

- Thank you for reading!
- Contributions are welcome.

## TODO

- Request review of this draft.
- Solve message signing and man in the middle:
    - E.g. Connectoid lib signs this JSON string using private key of Bob, to avoid replacing of message by Connectoid service.
    - But Connectoid service may impersonate Bob for Alice, and impersonate Alice for Bob, creating its own keypairs for both directions, allowing them to communicate, but can read and update all.
    - Alice <---> Service as Bob, Service as Alice <---> Bob.
    - Learn [E2EE](https://en.wikipedia.org/wiki/End-to-end_encryption) deeper re how to prevent [MITM](https://en.wikipedia.org/wiki/Man-in-the-middle_attack).
    - PGP uses either via web of trust (very complex to implement) or CA, which can be tampered too.
    - Telegram shows emoji fingerprints for users to validate using the voice they recognize, etc.
    - Maybe we could show something to our users too, learn more to find better solution.
    - Select default signing algorithm, add "alg" field as in JWT.
    - Lib adds this signature as "sig" field to the same JSON and check later - add missing steps to the spec.
- Solve availability as in https://github.com/nostr-protocol/nostr#how-does-it-solve-the-problems-the-networks-above-cant
    - "A relay can block a user from publishing anything there, but that has no effect on them as they can still publish to other relays. Since users are identified by a public key, they don't lose their identities and their follower base when they get banned. ... All of the above is valid too for when a relay ceases its operations."
- Consider how to add incentive for Connectoid service hosts by proposing user to select an option:
    - Paid service.
    - Free for ads.
    - Mining (reduces battery) without ads.
- Implement Connectoid service as a Python or TypeScript lib with example of deploy to AWS free tier.
- Implement Connectoid lib for Flutter.
- Use in a real app.
- Request wider review and libs for other app platforms.
