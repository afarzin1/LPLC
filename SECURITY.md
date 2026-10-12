# Security policy

## Reporting a vulnerability

Please report security issues privately through GitHub's
[private vulnerability reporting](https://github.com/afarzin1/LPLC/security/advisories/new) rather
than in a public issue. Include the affected block, the TwinCAT version, and steps to reproduce.
You should get a reply within a week.

## Scope and deployment model

LPLC runs inside the TwinCAT runtime and trusts its own process image. Anything that can write PLC
symbols over ADS (an HMI, a sidecar service, an engineering laptop) can command the machine, so:

- keep ADS routes, the AMS router and the IPC on a segregated machine network;
- treat every ADS client as fully trusted, and restrict who can add routes;
- set the MQTT broker, Modbus and database endpoints the blocks use for your own network; the
  library ships only placeholders.

Safety functions do not belong in this library or in any HMI. They belong in certified safety
hardware and logic.
