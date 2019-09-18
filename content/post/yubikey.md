---
date: 2019-09-18T00:00:00Z
title: The YubiKey; or, why are you still using a password in 2019?
tags: ["Security", "Cryptography", "YubiKey"]
---

_Note: I am not receiving any compensation either for what I'm saying here or 
for the product links on this page. I think the YubiKey is a great product, and 
I want to share why._

With the upcoming release of macOS Catalina -- and with it, the end of the last 
major browser holdout from the [FIDO](https://fidoalliance.org) standards, I
wanted to take a moment to talk about the YubiKey and the promise of the
standards that it supports for secure internet authentication.

Before we dive into the YubiKey itself, I'd like to explain why this is
important to me. The economic impact of compromised accounts is astronomical; 
according to a 2018 [study](https://www.mcafee.com/us/resources/reports/restricted/economic-impact-cybercrime.pdf?utm_source=Press&utm_campaign=bb9303ae70-EMAIL_CAMPAIGN_2018_02_21&utm_medium=email&utm_term=0_7623d157be-bb9303ae70-)
by the [Center for Strategic and International Studies](https://csis.org) in
collaboration with McAfee, the economic impact of cyber crime is nearly $600
billion per year, or 1% of total global GDP. At the same time, password security
is either misunderstood or ignored by both end users and system administrators
alike; [according to multiple sources](https://en.wikipedia.org/wiki/List_of_the_most_common_passwords),
the most commonly-used password in recent years is `123456`. In addition, 
password requirements vary greatly amongst services, with many still enforcing
maximum length limits and the use of special characters (or limitation of
characters) [despite mathematical evidence](https://xkcd.com/936/) that the use
of easy-to-remember long passphrases is far more secure than requiring multiple
special characters in a shorter password. Simply put - passwords need to be put
out to pasture. 

With what do we replace passwords? Public key cryptography has been widely-used
and battle-tested for decades -- every time you visit a site with the `https`
URL scheme, you are using it! -- and while some implementations have been shown
to have weaknesses over time, the basic concepts are solid. As long as the
private key can be protected, public key cryptograhy is an effective method of
authenticating an end user and encrypting communications, which is where the
YubiKey comes in.

If you are not familiar with the YubiKey, it is a multi-protocol hardware
cryptographic token. The premise behind this kind of token is that the secret 
key is stored on the token and the token itself performs cryptographic
operations using the key without making it accessible to the host system, thus 
preventing the key from being siphoned off by an unscrupulous actor. The latest
generation YubiKey 5 family has support for the following protocols:

* **USB HID**: these protocols take advantage of the capability of the YubiKey 
to act as a keyboard and send data as keystrokes:
  * Static password: The YubiKey can generate a secure password that can be
  combined with an easy-to-remember value to simulate a 2nd-factor auth.
  * Yubico OTP: A custom OTP implementation that relies on a shared key known
  by the YubiKey and the server. Yubico provides this service for all
  fresh-from-factory YubiKeys; if you don't trust Yubico with your key, you can
  run the service by yourself and reprogram your YubiKey with a new shared key.
* **OTP (One-time password)**: These protocols allow for time- or event-based
one-time passwords:
  * OATH-TOTP: This is the same protocol utilized by the nearly-ubiquitous
  mobile authenticators out there. In the case of the YubiKey, the one-time
  password values are read from the key with a special Yubico Authenticator
  application.
  * OATH-HOTP: This one-time password mechanism is not nearly as commonly used
  due to its reliance on counters and susceptibility to synchronization
  issues.
  * HMAC-SHA1 Challenge/Response: This protocol relies on a shared secret. The
  remote party sends a challenge, and the YubiKey calculates the HMAC-SHA1 code
  of the challenge using the shared secret, and returns it to the remote party.
* **CCID (Smart cards)**: These protocols allow the YubiKey to emulate smart
  cards:
  * OpenPGP: The YubiKey acts as an OpenPGP smart card, with three key slots
  available. The YubiKey then performs all sign/encrypt/decrypt operations using
  the stored keys.
  * PIV: The YubiKey acts as a PIV card and can be accessed using the standard
  PKCS#11 protocols. In addition to raw keys, the PIV applet can store
  certificates and generate CSRs, allowing for the keys to be certified and have
  enforced expiration and revocation capabilities.
* **FIDO (web authentication)**: These protocols allow for the usage of public
  key cryptography for authentication in the Web browser:
  * U2F, aka CTAP1: This protocol allows for 2nd-factor authentication using the
  private key stored on the YubiKey. It was never standardized beyond FIDO and 
  thus there are implementation and support issues across common browsers.
  * FIDO2, aka CTAP2: This protocol is backwards-compatible with CTAP1, and
  works in concert with the W3C WebAuthn standard to allow not only 2nd-factor
  authentication, but completely passwordless authentication. This is the future
  of web authentication.

In addition to the fully-featured Yubikey 5 line, Yubico offers a "Security Key"
line which only support the FIDO protocols. For the majority of people out
there, this is sufficient.

[WebAuthn](https://www.w3.org/TR/webauthn/) is a recently ratified standard by
the W3C specifying a JavaScript API for facilitating authentication with public 
key cryptography. This works together with the [CTAP device protocols](https://fidoalliance.org/specs/fido-v2.0-rd-20170927/fido-client-to-authenticator-protocol-v2.0-rd-20170927.html)
to enable the web browser to communicate with a CTAP-compatible USB device and 
use it to securely authenticate a user to a service. Per the [FIDO2](https://fidoalliance.org/fido2)
website: `FIDO2 cryptographic login credentials are unique across every website, never leave the user’s device and are never stored on a server. This security model eliminates the risks of phishing, all forms of password theft and replay attacks.` The FIDO2 protocol also supports requiring a local second
factor such as physical presence (push the button) or a PIN. In the case of a
PIN, this can be a simpler value as it is never transmitted over the Internet
nor stored outside of the device.

I utilize my YubiKey 4 as much as possible. I use the PIV applet to log in 
locally to my computers; the OpenPGP  applet to store my PGP auth, signing, and
encryption keys that I use for signing git commits or remotely logging into 
servers. I utilize Yubico OTP as a second factor on my most sensitive servers as 
well as the secnod factor on my password manager. I utilize FIDO U2F wherever 
possible (Google, AWS, Facebook). I understand I am a power user, and that many
of these use cases are quite advanced, but I simply see too much value in 
eschewing the world of the password and moving exclusively to public-key 
cryptography, now that I have the tools to do it. I only wish more things 
supported it, e.g. LUKS, which still requires a passphrase. 

Regardless of your level of expertise, I hope you are at least using a
[Security Key](https://www.yubico.com/products/security-key-3/). If not, get
one ASAP, and pressure every provider that does not yet support WebAuthn/FIDO2
to build that support as soon as possible. Hopefully we can start seeing that
$600 billion number trend downward as more people adopt YubiKeys or other
FIDO2-compliant devices.