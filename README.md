# pop3-nv

POP3, the Post Office Protocol version 3, lets a mail client fetch the
messages waiting in a mailbox on a server and delete them there. It is
specified in [RFC 1939](https://www.rfc-editor.org/rfc/rfc1939) and
extended by RFC 2449 (capabilities and response codes), RFC 5034 (AUTH)
and RFC 2595 (STLS). This package brings the client side of POP3 to
novo-lang. [mail-nv](https://novo-lang.org/packages/mail-nv) reads the
messages it fetches.

## What it is

POP3 is a conversation of lines over a TCP connection. The client sends a
command such as `RETR 1`. The server answers with a line that begins
`+OK` or `-ERR`, and for some commands follows it with lines of data that
end at a line holding one dot. A line of data that starts with a dot has
a second dot put in front of it, and the reader takes it out again.

The package has no socket and no clock. A program that uses it owns the
connection, and the TLS around it, and moves the bytes. The package says
which bytes to send and what the bytes it received mean. Three things
follow.

**The session is a step.** A `Pop3Client` is a value the caller holds in
a `var`. `send` takes a command and answers the bytes to write. `feed`
takes the bytes that arrived and answers a list of events. A read may hold
half a reply or three; the session keeps what is not yet a whole line.

**A message is bytes.** `RETR` and `TOP` answer a `Pop3Message` event whose
`body` is the message as a list of bytes, each line ending in CRLF, with
the byte-stuffing removed. The package reads no header and decodes no
MIME. mail-nv does that.

**A login mechanism is the caller's.** `AUTH` carries the bytes of a SASL
mechanism in base64. The package sends them and hands back the server's
challenges decoded. The mechanism, for instance
[scram-nv](https://novo-lang.org/packages/scram-nv), is another package.
The same holds for `STLS`: the package sends the command, says when the
server agreed, and waits to be told that TLS has started.

## Install

```
novo pkg add pop3-nv
```

## Example

This logs in with USER and PASS and reads the first message. The server's
lines are written out in the program so that it runs anywhere. In a
program that talks to a server, the bytes passed to `feed` are what the
socket read, and the bytes `send` answers are what the socket writes.

```novo
use std.bytes
use pop3session

fn bytes_of(s: Str) -> [u8]
    bytes.to_byte_list(bytes.from_str(s))

fn main() [io]
    var c = pop3session.new_client(pop3session.POP3_BODY_DEFAULT)

    // The greeting.
    let _g = pop3session.feed(var c, bytes_of("+OK POP3 server ready\r\n"))

    // USER, and the server's answer.
    let _u = pop3session.send(var c, Pop3CmdUser("mrose"))
    let _a = pop3session.feed(var c, bytes_of("+OK\r\n"))

    // PASS, and the server's answer.
    let _p = pop3session.send(var c, Pop3CmdPass("secret"))
    let _b = pop3session.feed(var c, bytes_of("+OK mailbox ready\r\n"))
    println(pop3session.phase_name(c.phase))   // transaction

    // RETR 1, and the message.
    match pop3session.send(var c, Pop3CmdRetr(1))
        Ok(line) => print(bytes.byte_list_to_str(line))   // RETR 1
        Err(e)   => println(e.message())
    let events = pop3session.feed(var c, bytes_of("+OK 38 octets\r\nSubject: hi\r\n\r\nhello\r\n.\r\n"))
    match events[0]
        Pop3Message(_, number, body, _) => println("message ${number}: ${list.len(body)} bytes")   // message 1: 22 bytes
        _ => println("no message")
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `pop3wire` | The lines of the protocol: a reply line with its response code, the command encoder, byte-stuffing, and the readers for STAT, LIST, UIDL and CAPA lines. |
| `pop3apop` | The timestamp in a greeting and the APOP digest. |
| `pop3session` | The client as a step: `send`, `feed`, the phases, the events and the commands. |
| `pop3err` | The refusals a line, a command or a call can meet, each with a sentence. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/pop3-nv) is that
reference.

## The commands and their events

| Command | Allowed in | Event on `+OK` |
| --- | --- | --- |
| `Pop3CmdUser` | authorization | `Pop3Reply` |
| `Pop3CmdPass` | authorization, after an accepted USER | `Pop3Authenticated` |
| `Pop3CmdApop` | authorization | `Pop3Authenticated` |
| `Pop3CmdAuth`, `Pop3CmdAuthInitial` | authorization | `Pop3Challenge` for each `+`, then `Pop3Authenticated` |
| `Pop3CmdAuthReply`, `Pop3CmdAuthCancel` | while a challenge waits | none |
| `Pop3CmdStls` | authorization | `Pop3TlsReady` |
| `Pop3CmdCapa` | either | `Pop3Capabilities` |
| `Pop3CmdStat` | transaction | `Pop3Stat` |
| `Pop3CmdList`, `Pop3CmdUidl` | transaction | `Pop3Listing`, `Pop3UidListing` |
| `Pop3CmdRetr`, `Pop3CmdTop` | transaction | `Pop3Message` |
| `Pop3CmdDele`, `Pop3CmdNoop`, `Pop3CmdRset` | transaction | `Pop3Reply` |
| `Pop3CmdQuit` | either | `Pop3Bye` |

A `-ERR` reply is a `Pop3Refused` event for any command. It carries the
response code of RFC 2449 section 8 when the server sent one.

## The rules a user needs

1. **One command at a time.** POP3 has no tags, so a reply belongs to the
   command sent last. `send` refuses a second command until the first is
   answered, with `Pop3Busy`. The two AUTH answers are the exception.
2. **A command belongs to a phase.** Before the greeting nothing may be
   sent. USER, PASS, APOP, AUTH and STLS belong to authorization; STAT,
   LIST, RETR, DELE, NOOP, RSET, TOP and UIDL to the transaction phase;
   CAPA and QUIT to either (RFC 1939 section 3). `send` refuses the rest
   and names the phase.
3. **The updates happen at QUIT.** RFC 1939 section 6 deletes the messages
   marked with DELE only when the server enters the update state after
   QUIT. A session that ends any other way deletes nothing.
4. **The reply to a multi-line command is one event.** `feed` answers it
   when the closing dot line has arrived. A limit, given to `new_client`
   in octets, bounds it, and a longer response ends the session with
   `Pop3BodyTooLong`.
5. **A line may end in CRLF or a bare LF.** Servers that send a bare LF
   are read. The bytes of a message body come out with CRLF.
6. **A status line is at most 512 octets** with its CRLF, and a command at
   most 255 (RFC 2449 section 4). Longer lines are refused.
7. **A fault ends the session.** A line that is not a status line, a
   STAT answer that is not two numbers and a line when nothing was asked
   each give a `Pop3Failed` event, and every later call is refused.
   A `-ERR` is not a fault.
8. **APOP needs the timestamp.** The greeting carries it, and `Pop3Greeting`
   names it. A server without one cannot be sent APOP. MD5, the digest
   APOP uses, is broken as a hash, and APOP protects a password from a
   passive observer only.
9. **Capabilities go stale.** The session forgets the list after a login
   and after `tls_started`, because the server may answer differently.
   Ask for CAPA again.
10. **A password is sent as it is.** USER and PASS arguments may contain
    spaces. They may not be empty, and may not contain CR, LF or NUL.

## What is not included

- **Sockets, TLS and timers.** The caller moves the bytes and decides when
  a server has been silent too long.
- **Reading a message.** The body is bytes. mail-nv parses it.
- **A SASL mechanism.** `AUTH` carries one. The mechanism is another
  package.
- **`AUTH` with no mechanism.** RFC 5034 allows it to list the mechanisms;
  CAPA's `SASL` line lists them here.
- **Pipelining.** RFC 2449 names a PIPELINING capability. The session
  sends one command at a time.
- **The server side.** Only the client is implemented.
- **The `UTF8` and `LANG` extensions** of RFC 6856.

## Related packages

- [mail-nv](https://novo-lang.org/packages/mail-nv) parses the message a
  `RETR` answers.
- [scram-nv](https://novo-lang.org/packages/scram-nv) is a SASL mechanism
  `AUTH` can carry.
- [tls-nv](https://novo-lang.org/packages/tls-nv) is the TLS a caller
  starts after `STLS`, or before the greeting on port 995.
- [imap-nv](https://novo-lang.org/packages/imap-nv) is the other protocol
  for reading a mailbox, one that keeps the messages on the server.

## Tests

The suite is two files under `tests/`. What they assert comes from the
RFCs.

`tests/pop3_wire_tests.nv` holds the lines one at a time: the status
lines, response codes, continuation lines, byte-stuffing, the STAT, LIST,
UIDL and CAPA lines with the values RFC 1939 and RFC 2449 print, the limits
of 512 and 255 octets, and the APOP digest
`c4c9334bac560ecc979e58001b3e22fb` that RFC 1939 section 7 gives for the
timestamp `<1896.697170952@dbc.mtview.ca.us>` and the secret `tanstaaf`.

`tests/pop3_session_tests.nv` plays the server by hand. It follows the
dialogues of RFC 1939 sections 5 to 7 line by line: STAT, LIST, RETR,
DELE, NOOP, RSET, TOP, UIDL and QUIT, with the `-ERR` answers the RFC
prints. It adds the greeting, CAPA, STLS, AUTH with a challenge, a cancel
and an initial response, replies cut at every byte, and each way a server
or a caller can break the rules above.

`bash tests/coverage.sh` merges the coverage of the suites and reports
the lines of `src/`.

## Licence

Apache-2.0. See `LICENSE`.
