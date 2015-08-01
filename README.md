NightMiner
==========

A very simple pure Python implementation of a CryptoCurrency stratum CPU mining client. Currently supports _scrypt_ (litecoin) and _SHA256d_ (bitcoin).

At a Glance
-----------

* Simple, one file
* Supports Scrypt (litecoin, dogecoin, etc) and SHA256d (bitcoin, namecoin, etc)
* Stratum (and only stratum)
* Zero dependencies (beyond standard Python libraries)
* 100% pure Python implementation
* Attempts to detect faster implementations of scrypt (pure Python is SLOW)
* Enable protocol chatter (-P) to see messages to and from the server

Command Line Interface
----------------------

    python nightminer.py [-h] [-o URL] [-u USERNAME] [-p PASSWORD]
                         [-O USERNAME:PASSWORD] [-a {scrypt,sha256d}] [-B] [-q]
                         [-P] [-d] [-v]

    -o URL, --url=              stratum mining server url
    -u USERNAME, --user=        username for mining server
    -p PASSWORD, --pass=        password for mining server
    -O USER:PASS, --userpass=   username:password pair for mining server

    -a, --algo                  hashing algorithm to use for proof of work (scrypt, sha256d)

    -B, --background            run in the background as a daemon

    -q, --quiet                 suppress non-errors
    -P, --dump-protocol         show all JSON-RPC chatter
    -d, --debug                 show extra debug information

    -h, --help                  show the help message and exit
    -v, --version               show program's version number and exit


    Example:
        python nightminer.py -o stratum+tcp://foobar.com:3333 -u user -p passwd
                                                                                                                                              

API
---
The API can be used by anyone wishing to create their own modified miner to learn more about the protocol, test their own pool or experiment with new algorithms.

```python
import nightminer
```


### Selecting a scrypt implementation (optional)

By default, the fastest detected library will be used; but if you wish to force a specific implementation:

```python
nightminer.set_scrypt_library(library = nightminer.SCRYPT_LIBRARY_AUTO)
print nightminer.SCRYPT_LIBRARY
```


### Subscription
After connecting to a stratum server, there is a small level of handshaking and then occasional messages to maintain state. The `Subscription` class manages this subscription state with the server.

**Properties:**
* `id` - The subscription ID
* `worker_name` - The name of the authenticated worker
* `difficulty`, `target` - The result of the proof of work must be less than `target`
* `extranounce1` - The extranounce1 
* `extranounce2_size` - The size of the binary extranounce2 (in bytes)

**set_subscription(subscription_id, extranounce1, extranounce2_size)**
Sets up the subscription details. Reply from the server to `mining.subscribe`.

**set_difficulty(difficulty)**
Sets the current difficulty. Sent from the server as a `mining.set_difficulty` message.

**set_worker_name(worker_name)**
Sets the worker's name after the server has authenticated the username/password. Reply from the server to `mining.authorize`.

**create_job(job_id, prevhash, coinb1, coinb2, merkle_branches, version, nbits, ntime)**
Creates a new job. Sent from the server as a `mining.notify` message.


### Job

When the server has a new job to work on it sends a `mining.notiffy` message. The `Job` class manages all the paameters required to perform work and performs the actual mining.

**Properties:**
* `id` - The job ID
* `prevhash` - The previous hash
* `coinb`, `coinb2` - The coinbase prefix and suffix
* `merkle_branches` - The Merkle branches
* `version` - The version
* `nbits`, `ntime` - The network bits and network time
* `target`, `extranounce`, `extranounce2_size` - See `Subscription` class above
* `hashrate` - The rate this miner has been hashing at

**merkle_root_bin(extranounce2_bin)**
Calculate the Merkle root, as a binary string.

**mine(nounce_start = 0, nounce_stride = 1)**
Iterates over all solutions for this job. This will run for an extrememly long time, likely far longer than ntime would be valid, so you will likely call `stop()` at some point and start on a new job.

**stop()**
Causes the `mine()` method to finish immediately for any thread inside.


### Miner

This is a sub-class of `SimpleJsonRpcClient` which connects to the stratum server and processes work requests from the server updating a `Subscription` object.

**Properties:**
* `url` - The stratum server URL
* `username`, `password` - The provided username and password

**serve_forever()**
Connect to the server, handshake and block forever while handling work from the server.

Use Cases
---------

### Create a standard miner
```python
miner = Miner('stratum+tcp://foobar.com:3333', 'username', 'password')
miner.server_forever()
```

### Experimenting with a new algorithm...

For this example, we will create a CryptoCoin based on MD5.

```python
import hashlib

# Create the Subscription object (proof-of-work should be 32 bytes long)
class SubscriptionMd5(nightminer.Subscription):
  def ProofOfWork(self, header):
    return hashlib.md5(header).digest() + ('0' * 16)
```

If you wish to manually find a few valid shares:
```python
# Create a subscription (and fill it in a bit with what a proper server would give us)
subs = SubscriptionMd5()
subs.set_subscription('my_subs_id', '00000000', 4)
subs.set_difficulty(1.0 / (2 ** 16))
subs.set_worker_name('my_fake_worker')

# Create a job
job = subs.create_job('my_job', ('0' * 64), ('0' * 118), ('0' * 110), [ ], '00000002', 'deadbeef', '01234567')

# Search for 5 shares
share_count = 0
for valid_share in job.mine():
  print "Found a valid share:", valid_share
  share_count += 1
  if share_count == 5: break

print "Hashrate:", job.hashrate
```

Or if you already have a server ready to go with your algorithm:
```python
# Register the Subscription
SubscriptionByAlgorithm['my_algo'] = SubscriptionMd5

# Start a miner
miner = Miner('stratum+tcp://localhost:3333', 'username', 'password', 'my_algo')
miner.server_forever()
```

FAQ
---

**Why would you do this?**
I was trying to tinker around with Litecoin, but found it difficult to find a simple, complete example of how to decode the endianness of the provided parameters and build the _block header_. So, the obvious next step is to create a full client to experiment with.

**Why is this so slow?**
It is written in Python. It is not meant to be fast, more of a reference solution or something that can be easily hacked into to test your own pool.

On my MacBook Air, with one thread I get around 3,000 hashes/s using the `ltc_scrypt` libary but less than 2 hashes/s using the built-in pure Python scrypt.

**What is this ltc_scrypt you speak of?**
It is a Python C-binding for a C implementation of scrypt found in p2pool (https://github.com/forrestv/p2pool). To add to your own system:

    > # Download the source
    > curl -L https://github.com/forrestv/p2pool/archive/13.4.tar.gz > p2pool-13.4.tar.gz

    > # Untar
    > tar -xzf p2pool-13.4.tar.gz
    
    > # Build and install
    > cd p2pool-13.4/litecoin_scrypt/
    > python setup.py build
    > sudo python setup.py install
    
After this is installed, this miner will be about 2,000 times faster. 

**Why am I am only getting rejected shares?**
Make sure you are using the correct algoithm, that means `--algo=scrypt` _(the default)_ for Litecoin or `--algo=sha256d` for Bitcoin.

**How do I get a question I have added?**
E-mail me at nightminer@ricmoo.com with any questions, suggestions, comments, et cetera.

**Can I give you my money?**
Umm... Ok? :-)

_Bitcoin_  - `1LNdGsYtZXWeiKjGba7T997qvzrWqLXLma`
_Litecoin_ - `LXths3ddkRtuFqFAU7sonQ678bSGkXzh5Q`
_Namecoin_ - `N6JLCggCyYcpcUq3ydJtLxv67eEJg4Ntk2`

