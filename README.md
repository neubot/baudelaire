# Baudelaire: a Neubot Master Server

Baudelaire is an implementation of Neubot's master server. It is written in Go
and it should be compatible with version v0.4.16.x of Neubot's client. It has
been mainly tested with Go v1.7.1 but should also work with other versions.

## Usage

```bash
GOPATH=$HOME go get -u -v
$HOME/bin/baudelaire --version # Get version number
$HOME/bin/baudelaire           # Run the server for testing
```

Log messages are written on the system log by default.

## The rendezvous protocol

Neubot uses the `rendezvous` protocol to communicate with its master server
periodically. Historically this communication used XML but since Neubot v0.4.10
released on March 15, 2012, JSON is used. Earlier clients using XML are not
supported by this Neubot master server anymore.

A Neubot client connects to Neubot's master server and sends the following
message to the `/rendezvous` API endpoint (the method SHOULD be `POST`, but
the code also accepts requests with the `GET` method, for historical reasons).

```JSON
 {
     "accept": [
         "speedtest",
         "bittorrent"
     ],
     "privacy_can_collect": 0,
     "privacy_can_share": 0,
     "privacy_informed": 0,
     "version": "0.4.17.0"
 }
```

The *accept* key should list the test names that the client implements and for
which it would accept suggestions indicating with which server such tests should
be performed. The *privacy_xx* keys contain the client's privacy settings and
they must all be nonzero otherwise Neubot should not perform any test (the
rationale being that it only performs tests if the privacy permissions allow
us to save tests results and put them in the public domain thus allowing other
people to study our results). The *version* key contains the version number
of the Neubot client.

Upon receiving this message a generic Neubot master server SHOULD:

1. check privacy permissions and return an error unless all of them are nonzero

2. use the connecting client's IP address to determine the best server to run
   tests with for each known test name listed in *accept*

The *version* key is currently unused. It was used to determine whether to
notify the user that he was running an obsolete version of Neubot, but this
feature is not needed anymore because now Neubot automatically auto updates.

The response sent by the server to the client MUST be like the following:

```JSON
 {
     "available": {
         "bittorrent": [
             "http://neubot.mlab.mlab1.mil01.measurement-lab.org:8080/"
         ],
         "speedtest": [
             "http://neubot.mlab.mlab1.mil01.measurement-lab.org:8080/speedtest"
         ]
     },
     "update": {
         "uri": "http://neubot.org/",
         "version": "0.4.15.6"
     }
 }
```

The server MAY choose to include *update* information if it has knowledge of the
most recent Neubot version and it would like to notify the client just in case,
but doing that is not mandatory (and, in fact, Baudelaire *never* provides the
client with any update information whatsoever). Otherwise, *update* should be
included as an empty JSON object (i.e. `{}`).

The server SHOULD fill the *available* map with (key, value) pairs where the key
should be the name of a test specified in the incoming *available* message and
the values should be lists of URLs to be used to run specific tests.

For historical reasons, given the FQDN `$x` of an host where a Neubot server
is accepting tests from Neubot client instances, the algorithm to make the URL
for each test is as follows:

- if the test is `speedtest`, `"http://" + $x + ":8080/speedtest"`

- otherwise, `"http://" + $x + ":8080/"

## Motivation and implementation details

Baudelaire uses [mlab-ns](https://docs.google.com/document/d/1eJhS75EZHDLmC6exggStr_b1euiR24_MVBJc1L6eH2c/view),
a name server implemented by [M-Lab](http://www.measurementlab.net/) as a
backend to map client's requests to test URLs.

Of the four available Neubot tests at the moment of this writing, two (`raw`
and `dash`) already use mlab-ns directly. However, `bittorrent` and `speedtest`
do not, and instead they are using the now obsolete
[neubot-master-server code](https://github.com/neubot/neubot/tree/3db1a1309a0eebacbf0f3c8df95c7d7ee68d8c59/neubot/rendezvous).

The plan is to update Neubot clients and stop using the master server for
discovering test servers (and perhaps start using it for telemetry) but until
then, there is need to have an implementation of the master server that
automatically updates the list of available servers. In fact, the current
neubot-server implementation needs to be updated manually and the whole process
is fragile, so I am not running such process very often (read: basically never).

Enters Baudelaire, which basically will translate a request for the master
server into an mlab-ns request, parse the response, prepare a rendezvous
response, and send the result back to the client.

Baudelaire only handles the `/rendezvous` URL with GET and POST. It MAY work
with other methods, but that's not guaranteed. Any other requested URL will
cause a `404` error to be returned.

Specifically, in its `/rendezvous` handler, Baudelaire will use the client's
IP address `$ip` to send an HTTP GET request for `/neubot?ip=$ip` at
`mlab-ns.appspot.com` on port 80. Note that Baudelaire does not support
IPv6 (and Neubot's master server is a IPv4-only machine).

The response received by mlab-ns would look like this:

```JSON
 {
     "city": "Turin",
     "url": "http://neubot.mlab.mlab1.trn01.measurement-lab.org:8080",
     "ip": ["194.116.85.211", "2001:7f8:23:307::211"],
     "fqdn": "neubot.mlab.mlab1.trn01.measurement-lab.org",
     "site": "trn01",
     "country": "IT"
 }
```

Of this response, Baudelaire will only care of the `fqdn` field and will in
particular use it to assemble a rendezvous response as indicated above,
by properly filling the *available* map.

In case of success, Baudelaire will respond with HTTP code equal to `200`
and a JSON-formatted body compatible with the rendezvous response described
above. Otherwise, if any operation fails, Baudelaire will set code equal
to `500` and send as body an empty JSON object (i.e. `{}`).

## Deployment

The Baudelaire binary can be compiled from any machine with Go installed
by running `go build` or `make` (if `make` is installed). It can be cross
compiled for Linux and x86_systems by running:

```
GOPATH=$HOME GOARCH=amd64 GOOS=linux go build
```

Once the binary has been compiled, just run `sudo make install` on the
target system, which MUST be a Debian system (we tested it with Debian v8.6).
Specifically, this command performs the following actions:

1. installs `baudelaire` under `/usr/local/bin`

2. installs `baudelaire.service` (the systemd unit) under the location
   where user installed units should be (`/usr/local/lib/systemd/system`)

3. installs a custom `rc.local` that:

    1. sets up ports redirection (Baudelaire uses by default port `8080` but
       the old server used also ports `80` and `9773`)

    2. starts Baudelaire using `systemctl`

After you have run `sudo make install`, you should reboot the system to make
sure the changes have had effect. The install command should be idempotent so
you should be able to also use it to upgrade Baudelaire. To get the version
of the currently installed Baudelaire, run:

```
baudelaire --version
```

Note that systemd will take care of running Baudelaire in the background
and with the privileges of the `nobody` user. This explains why Baudelaire
itself does not contain code to drop privileges and go background.

As already stated, Baudelaire uses by default the system log, thus you
can see its log messages by running this command:

```
sudo tail -f /var/log/syslog
```
