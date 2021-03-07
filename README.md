# fulcio - A New Kind of Root CA For Code Signing

fulcio is a free Root-CA for CodeSignign certs- issuing certificates based on an OIDC email address.
fulcio only signs short-lived certificates that are valid for under 20 minutes.

The fulcio root cert is:

```
-----BEGIN CERTIFICATE-----
  MIIB+DCCAX6gAwIBAgITNVkDZoCiofPDsy7dfm6geLbuhzAKBggqhkjOPQQDAzAq
  MRUwEwYDVQQKEwxzaWdzdG9yZS5kZXYxETAPBgNVBAMTCHNpZ3N0b3JlMB4XDTIx
  MDMwNzAzMjAyOVoXDTMxMDIyMzAzMjAyOVowKjEVMBMGA1UEChMMc2lnc3RvcmUu
  ZGV2MREwDwYDVQQDEwhzaWdzdG9yZTB2MBAGByqGSM49AgEGBSuBBAAiA2IABLSy
  A7Ii5k+pNO8ZEWY0ylemWDowOkNa3kL+GZE5Z5GWehL9/A9bRNA3RbrsZ5i0Jcas
  taRL7Sp5fp/jD5dxqc/UdTVnlvS16an+2Yfswe/QuLolRUCrcOE2+2iA5+tzd6Nm
  MGQwDgYDVR0PAQH/BAQDAgEGMBIGA1UdEwEB/wQIMAYBAf8CAQEwHQYDVR0OBBYE
  FMjFHQBBmiQpMlEk6w2uSu1KBtPsMB8GA1UdIwQYMBaAFMjFHQBBmiQpMlEk6w2u
  Su1KBtPsMAoGCCqGSM49BAMDA2gAMGUCMH8liWJfMui6vXXBhjDgY4MwslmN/TJx
  Ve/83WrFomwmNf056y1X48F9c4m3a3ozXAIxAKjRay5/aj/jsKKGIkmQatjI8uup
  Hr/+CxFvaJWmpYqNkLDGRU+9orzh5hI2RrcuaQ==
  -----END CERTIFICATE-----
  ```

## Status

Fulcio is a work in progress.
There's working code and a running instance and a plan, but you should not attempt
to try to actually use it for anything.

## API

The API is defined via OpenAPI.
The defintiion is [here](openapi.yaml).

## Transparency

Fulcio will publish issued certificates to a unique CT-log.
That log will be hosted by the Sigstore project.
We encourage auditors to monitor this log, and aim to help people the data.

A simple example would be a service to email users (on a different address)when ceritficates
have been issued on their behalf.
This can be used to detect bad behavior or other compromises.

## Parameters

The fulcio root CA is running on GCP Private CA with the EC_P384_SHA384 algorithm.

## Security Model

* Fulcio assumes that a valid OIDC token is a sufficient "proof of ownership" of an email address.
* To mitigate against this: Fulcio uses a Transparency log to help protect against OIDC
  compromises.This means:
      * Fulcio MUST publish all certificates to the log.
      * Clients MUST NOT trust certificates that are not in the log.
    * This means users can detect any mis-issued certificates.
* Combined with `rekor's` signature transparency, artifacts signed with stolen certificates can even
  be identified.

### Revocation, Rotation and Expiry

There are two main approaches to code signing:
Long-term certs and trusted time-stamping.

#### Long-term certs

These certificates are typically valid for years.
All old code must be re-signed with a new çert before an old cert expires.

Typically this works with long deprecation periods, dual-signing and planned rotations.
This approach assumes users can keep acess to private keys and keep them secret over
log periods of time.

This other big problem with this approach is revocation.
Revocation is hard and doesn't work well.

#### Fulcio's Model

Fulcio is designed to avoid revocation, by issuing short-lived certificates.
What really matters for CodeSigning is to know that an artifact was signed while the
certificate was valid.

This can be done a few ways:

* Third-party Timestamp Authorities (RFC3161)
* Transparency Logs
* Both (Fulcio's Model)

#### RFC3161 Timestamp Servers

RFC3161 defines a protocol for Trusted Timestamps.
Parties can send a payload to an RFC3161 service and, the service digitally signs that payload with
its own timestamp.

This is the equivalent of posting a hash to Twitter - you are getting a third-party attestation that
you had a particular piece of data at a particular time, as observed by that same third-party.

The downside is that users need to interact with another service.
They must timestamp all signatures and check the timestamps of all signatures - adding
another dependency and set of keys they must trust (the timestamp servers).

We could provide one for free, but if people don't trust the clock in our transparency ledger they might
not trust another service we run.

#### Transparency Logs

The `rekor` service provides a transparency log of software signatures.
As entries are appended into this log, `rekor` periodically signs the full tree along with a timestamp.
An entry in Rekor provide a single-party attestation that a piece of data existed prior to a certain time.
These tiemstamps cannot be tampered with later, providing long-term trust - but their strength gains from
being monitored.

Transparency Logs make it hard to forge timestamps long-term, but in short time-windows it would be much easier for
the Rekor oeprator to fake or forge timestamps. 

More skeptical parties don't have to trust Rekor at all!
Rekor's timestamps and STH's are signed - a valid signed tree hash contains a non-repudiadable timestamp.
They can save these signed timestamp tokens as evidence in case Rekor's clock changes in the future.

#### Why Not Both!?!?!?

Like usual, we can combine these approaches and do a bit better.
Rekor can interact with third-party TSAs automatically, allowing users to skip this step.

Timestamp authorities provide signatures over pieces of data and a timestamp.
Rekor can get its own STH (including the timestamp) signed by one or many third-party TSAs regularly.

Each timestamp attestation in the Rekor log provides a fixed "fencepost" in time.
Rekor, the client and a third-party can all provide evidence of the state of the world at a point in time.

Fenceposts every ten minutes protect all data in between.
Auditors can monitor Rekor's log to ensure these are added, shifting the complexity burden from users
uditors.

## Info

`Fulcio` is developed as part of the [`sigstore`](https://sigstore.dev) project.
Come on over to our [slack channel](https://sigstore.slack.com)!
