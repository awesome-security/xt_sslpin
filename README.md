# xt_sslpin 
_netfilter/xtables module: match SSL/TLS certificate finger prints_

## SYNOPSIS

    iptables -I <chain> .. -m sslpin [!] --fpl <finger print list id> [--debug] ..

## DESCRIPTION

For an introduction to SSL/TLS certificate pinning refer to the [OWASP pinning cheat sheet](https://www.owasp.org/index.php/Pinning_Cheat_Sheet). xt_sslpin lets you do certificate validation/pinning at the netfilter level. xt_sslpin will match certificate finger prints in SSL/TLS connections (with minimal performance impact). Applications are expected to do further certificate chain validation and signature checks (i.e. normal SSL/TLS processing).

## EXAMPLE

Drop connections matching on list `0`:

```shell
iptables -I INPUT -p tcp --sport 443 \
    -m conntrack --ctstate ESTABLISHED \
    -m sslpin --debug --fpl 0 \
    -j DROP
```

Add https://github.com certificate to list `0`

```shell
echo \
| openssl s_client -connect github.com:443 2>/dev/null \
| openssl x509 -outform DER \
| sha1sum > /sys/kernel/xt_sslpin/0_add
```

## INSTALLATION

Prerequisites

* linux kernel > 3.7
* kernel-headers
* iptables-dev
* gcc
* git

then:

    git clone https://github.com/Enteee/xt_sslpin.git
    cd xt_sslpin
    sudo apt-get install iptables-dev # xtables.h

Build and install:

```shell
make
sudo make install
```

Verify install:

```shell
iptables -m sslpin -h
```

### Uninstalling

Clean source/build directory:

```shell
make clean
```

Uninstall:

```shell
sudo make uninstall
```

## OPTIONS

Options preceded by an exclamation mark negate the comparison: the rule will match if the presented SSL/TLS certificate fingerprint does NOT match the specified public key.

### `[!] --fpl <list id>` 

If a "Certificate" message is seen, match if the one of the certificates matches a finger print in the given list. (_The `--debug` option can be used to get the public key for a server._)

### `--debug`

Verbose logging.

```
kernel: [ 154.806189] xt_sslpin 1.0 (SSL/TLS pinning)
kernel: [ 156.976209] xt_sslpin: 1 connection (0 actively monitored)
kernel: [ 156.127355] xt_sslpin: 1 connection (1 actively monitored)
kernel: [ 157.127367] xt_sslpin: sslparser: ServerHello handshake message (len = 85)
kernel: [ 157.127370] xt_sslpin: sslparser: Certificate handshake message (len = 2193)
kernel: [ 157.127373] xt_sslpin: sslparser: cn = "example.com"
kernel: [ 157.127378] xt_sslpin: sslparser: pubkey_alg = { name:"rsa", oid_asn1_hex:[2a864886f...] }
kernel: [ 157.127387] xt_sslpin: sslparser: pubkey = [00000000000000000000000000000000...]
kernel: [ 159.129145] xt_sslpin: sslparser: ServerDone handshake message (len = 0)
kernel: [ 159.285698] xt_sslpin: sslparser: ChangeCipherSpec record
kernel: [ 159.285714] xt_sslpin: rule not matched (cn = "example.com")
kernel: [ 159.344721] xt_sslpin: 1 connection (0 actively monitored)
```

## LIST API

The list API is exposed under: `/sys/kernel/xt_sslpin/`.

| Operation | Command |
| --------- | ------- |
| ADD       | `echo finger-print-sha1 > /sys/kernel/xt_sslpin/<list id>_add` |
| REMOVE    | `echo finger-print-sha1 > /sys/kernel/xt_sslpin/<list id>_rm`  |
| LIST      | `ls /sys/kernel/xt_sslpin/<list id>` |

## IMPLEMENTATION NOTES

Per connection, the incoming handshake data is parsed once across all -m sslpin iptables rules;
upon receiving the SSL/TLS handshake ChangeCipherSpec message, the parsed certificates are checked by all rules.

After this, the connection is marked as "finished", and xt_sslpin will not do any further checking.
(Re-handshaking will not be checked in order to incur minimal overhead, and as the server has already proved
its identity).

Up until the ChangeCipherSpec message is received, xt_sslpin will drop out-of-order TCP segments to
parse the data linearly without buffering. Conntrack takes care of IP fragment reassembly up-front, but packets
can still have non-linear memory layout; see skb_is_nonlinear().

If SYN is received on a time-wait state conn/flow, conntrack will destroy the old cf_conn
and create a new cf_conn. Thus, our per-conn state transitions are simply new->open->destroyed (no reopen).

## TODO

* ECParameters/namedCurve pinning in addition to current alg+pubkey pinning
* Optional buffering for reordered TCP segments during handshake (no RTT penalty / overhead)
* TCP Fast Open (TFO) support
* Restrict TCP Options / TCP stack passthrough
* IPv6 support

## Acknowledgment

This module is a fork from [xt_sslpin by fredburger (github.com/fredburger)](https://github.com/fredburger/xt_sslpin).

Thank you!

## LICENSE

xt_sslpin is Copyright (C) 2016 Enteee (duckpond.ch).

This program is free software; you can redistribute it and/or modify it under the terms of the
GNU General Public License as published by the Free Software Foundation; version 2 of the License.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program; if not, write to
the Free Software Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301, USA.
