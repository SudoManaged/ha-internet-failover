# Security policy

## Reporting a vulnerability

Please report security problems **privately**: on this repository's **Security** tab choose
**Report a vulnerability**. Do not open a public issue for them.

Include the blueprint, your Home Assistant version, what an attacker could do, and the steps to reproduce it.
We aim to reply within a few days and to publish a fix, with credit if you want it, once it is ready.

## Scope

These blueprints run entirely inside your own Home Assistant. They use no webhooks and open nothing to the
network; the public address is looked up by the DNS IP integration (or the optional command line sensor in the
README). Reports about Home Assistant itself belong with the Home Assistant project.
