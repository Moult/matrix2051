# Matrix2051

An IRC server backed by Matrix. You can also see it as an IRC bouncer that
connects to Matrix homeservers instead of IRC servers.

Goals:

1. Make it easy for IRC users to join Matrix seamlessly
2. Support existing relay bots, to allows relays that behave better on IRC than
   existing IRC/Matrix bridges
3. Bleeding-edge IRCv3 implementation
4. Very easy to install. This means:
   1. as little configuration and database as possible (ideally zero)
   2. small set of depenencies.

Non-goals:

1. Being a hosted service (it would require spam countermeasures, and that's a lot of work).
2. TLS support (see previous point). Just run it on localhost. If you really need it to be remote, access it via a VPN or a reverse proxy.
3. Connecting to multiple accounts per IRC connection or to other protocols (à la [Bitlbee](https://www.bitlbee.org/)). This conflicts with goals 1 and 4.
4. Implementing any features not natively by **both** protocols (ie. no need for service bots that you interract with using PRIVMSG)

## Usage

* Install system dependencies. For example, on Debian: `sudo apt install elixir otp erlang-dev erlang-inets erlang-xmerl`
* Install Elixir dependencies: `mix deps.get`
* Run tests to make sure everything is working: `mix test`
* Run: `mix run --no-halt matrix2051.exs`
* Connect a client to `localhost:2051`, with the following config:
  * no SSL/TLS
  * SASL username: your full matrix ID (`user:homeserver.example.org`)
  * SASL password: your matrix password

## Roadmap

* [x] password authentication (using [SASL](https://ircv3.net/specs/extensions/sasl-3.1) on the IRC side)
* [x] registration (using the [draft/account-registration](https://github.com/ircv3/ircv3-specifications/pull/435) spec)
  * [x] on Synapse with no email verification
  * [ ] on Synapse with email verification
  * [ ] on other homeservers (when the Matrix spec defines how to)
* [x] retrieving the room list and joining clients to it
* [x] retrieving the member list and sending it to clients
* [x] support JOIN from clients
* [x] sending messages from IRC clients
* [x] receiving messages from Matrix
  * [x] basics
  * [x] split at 512 bytes
* [ ] support PART from IRC clients
* [ ] direct messages
* [x] rewrite mentions/highlights
  * [x] Matrix -> IRC
  * [x] IRC -> Matrix
* [x] rewrite formatting and colors
  * [x] Matrix -> IRC
  * [x] IRC -> Matrix
* [x] show JOIN/PART events from other members
* [ ] figure out images / files
  * [x] Matrix -> IRC
  * [ ] IRC -> Matrix
* [x] [display names](https://github.com/ircv3/ircv3-specifications/pull/452)
* [ ] [chat history](https://ircv3.net/specs/extensions/chathistory)
* [ ] connection via [websockets](https://github.com/ircv3/ircv3-specifications/pull/342)
* [x] [multiline](https://ircv3.net/specs/extensions/multiline) messages
  * [x] Matrix -> IRC
  * [x] IRC -> Matrix
* [ ] [reacts](https://ircv3.net/specs/client-tags/reply)
  * [ ] Matrix -> IRC
  * [ ] IRC -> Matrix
* [x] [replies](https://ircv3.net/specs/client-tags/reply)
  * [x] Matrix -> IRC
  * [x] IRC -> Matrix
* [ ] message edition/deletion
* [ ] invites
* [ ] translate channel modes / room privileges
  * [ ] Matrix -> IRC
  * [ ] IRC -> Matrix
* [x] use [channel renaming](https://ircv3.net/specs/extensions/channel-rename)
  * [x] when the canonical name changes
  * [ ] in case of joining duplicate aliases of the same room?

In the far future:

* [ ] support authentication with non-password flows
* [ ] end-to-"end" encryption (Matrix2051 would be the end, unless there is a way
  to make IRC clients understand Olm)
* [ ] Matrix P2P

## Architecture

* `matrix2051.exs` starts Matrix2051, which starts Matrix2051.Supervisor, which
  supervises:
  * `config.ex`: global config agent
  * `irc_server.ex`: a `DynamicSupervisor` that receives connections from IRC clients.

Every time `irc_server.ex` receives a connection, it spawns `irc_conn/supervisor.ex`,
which supervises:

* `irc_conn/state.ex`: stores the state of the connection
* `irc_conn/writer.ex`: genserver holding the socket and allowing
  to write lines to it (and batches of lines in the future)
* `irc_conn/handler.ex`: task busy-waiting on the incoming commands
  from the reader, answers to the simple ones, and dispatches more complex
  commands
* `matrix_client/state.ex`: keeps the state of the connection to a Matrix homeserver
* `matrix_client/client.ex`: handles one connection to a Matrix homeserver, as a single user
* `matrix_client/sender.ex`: sends events to the Matrix homeserver and with retries on failure
* `matrix_client/poller.ex`: repeatedly asks the Matrix homeserver for new events (including the initial sync)
* `irc_conn/reader.ex`: task busy-waiting on the incoming lines,
  and sends them to the handler

Utilities:

* `matrix/raw_client.ex`: low-level Matrix client / thin wrapper around HTTP requests
* `irc/command.ex`: IRC line manipulation, including "downgrading" them for clients
  that don't support some capabilities.
* `irc/word_wrap.ex`: generic line wrapping
* `format/`: Convert between IRC's formatting and `org.matrix.custom.html`
