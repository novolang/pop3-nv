# Changelog

All notable changes to pop3-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`pop3wire`**, the lines of POP3. A reply line is read into `+OK`,
  `-ERR` or the `+` continuation of AUTH, with the response code of
  RFC 2449 split off the text and a limit of 512 octets. Commands are
  built with the limit of 255 octets and a refusal of empty arguments and
  of CR, LF and NUL. The STAT, LIST, UIDL and CAPA lines are read, a
  byte-stuffed line is unstuffed, and the base64 lines of AUTH are encoded
  and decoded.
- **`pop3apop`**, the timestamp of a greeting and the APOP digest, checked
  against the example of RFC 1939 section 7.
- **`pop3session`**, the client as a step: `send` and `feed` over a
  `Pop3Client`, thirteen commands plus the AUTH answers, the three phases
  of RFC 1939 section 3, one command in flight, multi-line responses
  gathered into one event with a size limit, a bare LF accepted as a line
  end, the capabilities kept and forgotten when a login or TLS may have
  changed them, and a fault in the bytes ending the session.
- **`pop3err`**, the refusals, each with a sentence.
- Two test suites: `tests/pop3_wire_tests.nv` (the lines, with the values
  of RFC 1939 and RFC 2449) and `tests/pop3_session_tests.nv` (the
  dialogues of RFC 1939 sections 5 to 7, and AUTH, STLS and CAPA).
  `tests/coverage.sh` merges the coverage: 100% of `src/`.

### Not included

- Sockets, TLS and timers. The caller moves the bytes.
- Reading a message. The body is handed over as bytes.
- A SASL mechanism, `AUTH` with no mechanism, pipelining, the server side,
  and the `UTF8` and `LANG` extensions of RFC 6856.
