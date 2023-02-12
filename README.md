# nostr-tools

Tools for developing [Nostr](https://github.com/fiatjaf/nostr) clients.

Only depends on _@scure_ and _@noble_ packages.

## Usage

### Generating a private key and a public key

```js
import {generatePrivateKey, getPublicKey} from 'nostr-tools'

let sk = generatePrivateKey() // `sk` is a hex string
let pk = getPublicKey(sk) // `pk` is a hex string
```

### Creating, signing and verifying events

```js
import {
  validateEvent,
  verifySignature,
  signEvent,
  getEventHash,
  getPublicKey
} from 'nostr-tools'

let event = {
  kind: 1,
  created_at: Math.floor(Date.now() / 1000),
  tags: [],
  content: 'hello',
  pubkey: getPublicKey(privateKey)
}

event.id = getEventHash(event)
event.sig = signEvent(event, privateKey)

let ok = validateEvent(event)
let veryOk = verifySignature(event)
```

### Interacting with a relay

```js
import {
  relayInit,
  generatePrivateKey,
  getPublicKey,
  getEventHash,
  signEvent
} from 'nostr-tools'

const relay = relayInit('wss://relay.example.com')
await relay.connect()

relay.on('connect', () => {
  console.log(`connected to ${relay.url}`)
})
relay.on('error', () => {
  console.log(`failed to connect to ${relay.url}`)
})

// let's query for an event that exists
let sub = relay.sub([
  {
    ids: ['d7dd5eb3ab747e16f8d0212d53032ea2a7cadef53837e5a6c66d42849fcb9027']
  }
])
sub.on('event', event => {
  console.log('we got the event we wanted:', event)
})
sub.on('eose', () => {
  sub.unsub()
})

// let's publish a new event while simultaneously monitoring the relay for it
let sk = generatePrivateKey()
let pk = getPublicKey(sk)

let sub = relay.sub([
  {
    kinds: [1],
    authors: [pk]
  }
])

sub.on('event', event => {
  console.log('got event:', event)
})

let event = {
  kind: 1,
  pubkey: pk,
  created_at: Math.floor(Date.now() / 1000),
  tags: [],
  content: 'hello world'
}
event.id = getEventHash(event)
event.sig = signEvent(event, sk)

let pub = relay.publish(event)
pub.on('ok', () => {
  console.log(`${relay.url} has accepted our event`)
})
pub.on('seen', () => {
  console.log(`we saw the event on ${relay.url}`)
})
pub.on('failed', reason => {
  console.log(`failed to publish to ${relay.url}: ${reason}`)
})

let events = await relay.list([{kinds: [0, 1]}])
let event = await relay.get({
  ids: ['44e1827635450ebb3c5a7d12c1f8e7b2b514439ac10a67eef3d9fd9c5c68e245']
})

await relay.close()
```

### Interacting with multiple relays

```js
import {SimplePool} from 'nostr-tools'

const pool = new SimplePool()

let relays = ['wss://relay.example.com', 'wss://relay.example2.com']

let relay = await pool.ensureRelay('wss://relay.example3.com')

let subs = pool.sub([...relays, relay], {
  authors: ['32e1827635450ebb3c5a7d12c1f8e7b2b514439ac10a67eef3d9fd9c5c68e245']
})

subs.forEach(sub =>
  sub.on('event', event => {
    // this will only be called once the first time the event is received
    // ...
  })
)

let pubs = pool.publish(relays, newEvent)
pubs.forEach(pub =>
  pub.on('ok', () => {
    // ...
  })
)

let events = await pool.list(relays, [{kinds: [0, 1]}])
let event = await pool.get(relays, {
  ids: ['44e1827635450ebb3c5a7d12c1f8e7b2b514439ac10a67eef3d9fd9c5c68e245']
})

let relaysForEvent = pool.seenOn(
  '44e1827635450ebb3c5a7d12c1f8e7b2b514439ac10a67eef3d9fd9c5c68e245'
)
// relaysForEvent will be an array of URLs from relays a given event was seen on
```

### Querying profile data from a NIP-05 address

```js
import {nip05} from 'nostr-tools'

let profile = await nip05.queryProfile('jb55.com')
console.log(profile.pubkey)
// prints: 32e1827635450ebb3c5a7d12c1f8e7b2b514439ac10a67eef3d9fd9c5c68e245
console.log(profile.relays)
// prints: [wss://relay.damus.io]
```

To use this on Node.js you first must install `node-fetch@2` and call something like this:

```js
nip05.useFetchImplementation(require('node-fetch'))
```

### Encoding and decoding NIP-19 codes

```js
import {nip19, generatePrivateKey, getPublicKey} from 'nostr-tools'

let sk = generatePrivateKey()
let nsec = nip19.nsecEncode(sk)
let {type, data} = nip19.decode(nsec)
assert(type === 'nsec')
assert(data === sk)

let pk = getPublicKey(generatePrivateKey())
let npub = nip19.npubEncode(pk)
let {type, data} = nip19.decode(npub)
assert(type === 'npub')
assert(data === pk)

let pk = getPublicKey(generatePrivateKey())
let relays = [
  'wss://relay.nostr.example.mydomain.example.com',
  'wss://nostr.banana.com'
]
let nprofile = nip19.nprofileEncode({pubkey: pk, relays})
let {type, data} = nip19.decode(nprofile)
assert(type === 'nprofile')
assert(data.pubkey === pk)
assert(data.relays.length === 2)
```

### Encrypting and decrypting direct messages

```js
import {nip04, getPublicKey, generatePrivateKey} from 'nostr-tools'

// sender
let sk1 = generatePrivateKey()
let pk1 = getPublicKey(sk1)

// receiver
let sk2 = generatePrivateKey()
let pk2 = getPublicKey(sk2)

// on the sender side
let message = 'hello'
let ciphertext = await nip04.encrypt(sk1, pk2, message)

let event = {
  kind: 4,
  pubkey: pk1,
  tags: [['p', pk2]],
  content: ciphertext,
  ...otherProperties
}

sendEvent(event)

// on the receiver side
sub.on('event', event => {
  let sender = event.tags.find(([k, v]) => k === 'p' && v && v !== '')[1]
  pk1 === sender
  let plaintext = await nip04.decrypt(sk2, pk1, event.content)
})
```

### Performing and checking for delegation

```js
import {nip26, getPublicKey, generatePrivateKey} from 'nostr-tools'

// delegator
let sk1 = generatePrivateKey()
let pk1 = getPublicKey(sk1)

// delegatee
let sk2 = generatePrivateKey()
let pk2 = getPublicKey(sk2)

// generate delegation
let delegation = nip26.createDelegation(sk1, {
  pubkey: pk2,
  kind: 1,
  since: Math.round(Date.now() / 1000),
  until: Math.round(Date.now() / 1000) + 60 * 60 * 24 * 30 /* 30 days */
})

// the delegatee uses the delegation when building an event
let event = {
  pubkey: pk2,
  kind: 1,
  created_at: Math.round(Date.now() / 1000),
  content: 'hello from a delegated key',
  tags: [['delegation', delegation.from, delegation.cond, delegation.sig]]
}

// finally any receiver of this event can check for the presence of a valid delegation tag
let delegator = nip26.getDelegator(event)
assert(delegator === pk1) // will be null if there is no delegation tag or if it is invalid
```

Please consult the tests or [the source code](https://github.com/fiatjaf/nostr-tools) for more information that isn't available here.

### Using from the browser (if you don't want to use a bundler)

```html
<script src="https://unpkg.com/nostr-tools/lib/nostr.bundle.js"></script>
<script>
  window.NostrTools.generatePrivateKey('...') // and so on
</script>
```

## Plumbing

1. Install [`just`](https://just.systems/)
2. `just -l`

## License

Public domain.
