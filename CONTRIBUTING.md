# Contributing to LPLC

Thanks for helping.

## Before you start

- Read [`CONVENTIONS.md`](CONVENTIONS.md). Pull requests that break its naming or structure rules
  will be asked to change, so it saves a round trip.
- For anything larger than a fix, open an issue first so the approach can be agreed.

## Making a change

- Build in TwinCAT XAE (3.1.4026 or newer) with no new errors or warnings before opening a pull
  request, and say in the description what you tested and on which hardware.
- Register every new file in `PLCLibrary/PLCLibrary.plcproj` (CONVENTIONS §9).
- Changes to a `.HMI` struct are contract changes for HMIs. List them in `CHANGELOG.md`.
- **No site data.** Never commit real hostnames, IP addresses, credentials, customer or machine
  names, or a machine's own PLC project.

## AI-assisted contributions

Welcome. You are responsible for what you submit: review it, build it in TwinCAT, and make sure the
agent read `AGENTS.md` and `CONVENTIONS.md` first. Keep agent instructions in `AGENTS.md` only; do
not add tool-specific instruction files (`CLAUDE.md`, `.cursorrules`, …).

## Sign-off and license

LPLC is licensed under the [MIT License](LICENSE). By contributing you agree that your contribution
is licensed under the same terms. Please sign off each commit to certify the
[Developer Certificate of Origin](https://developercertificate.org/):

```bash
git commit -s -m "Fix the thing"
```

## Safety

LPLC is not a safety-rated library. Contributions must not present an interlock or permissive as a
safety function.
