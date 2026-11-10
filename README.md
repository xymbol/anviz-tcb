# anviz-tcb

A library with no code for the TC_B protocol that Anviz time clocks speak. It
encodes and decodes frames, commands, users, PINs, punches and dates.

## The idea

The library is a specification and a set of tests. You give both to a coding
agent and it writes the implementation in your language, then runs the tests
until they pass. The approach is inspired by Drew Breunig's
[whenwords](https://github.com/dbreunig/whenwords), described in
[A software library with no code](https://www.dbreunig.com/2026/01/08/a-software-library-with-no-code.html).

It fits this protocol well: the codec is small, pure and unlikely to change,
and the hard part is knowing the bytes, which the spec and tests capture. What
it can't do is keep implementations in step for you: if the spec changes, each
one has to be regenerated and retested.

## Use

1. Copy `SPEC.md` and `tests.yaml` into your project.
2. Paste the prompt in [INSTALL.md](INSTALL.md) into a coding agent, with your
   language and the place for the code.
3. Keep the generated tests: they're how you know the implementation conforms.

## Files

- [SPEC.md](SPEC.md): every function, byte by byte, with the errors to raise,
  how to turn `tests.yaml` into tests, and guidance on the session flow.
- [tests.yaml](tests.yaml): 142 language-neutral cases, many of them bytes
  recorded from the clock.
- [INSTALL.md](INSTALL.md): the prompt for the coding agent.

## Scope

In: the CRC, frame encoding and stream splitting, the commands a server sends,
and decoding what the clock sends back (device identity, dates, punches, users,
PINs, record counts), plus folding names for the clock's display.

Out: networking, sessions, retries, storage. The spec describes how a server
talks to the clock, as guidance; how you build one is up to you.

## Origin

Extracted from a working implementation that runs an Anviz G100-N (firmware
`02.07.A1`) in WiFi-client mode. Every case in `tests.yaml` was either recorded
from that clock or checked against that implementation; a few where the
implementation validates less strictly than the spec were checked by hand.
