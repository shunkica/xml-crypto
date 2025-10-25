# XmlDSigVerifier Usage Guide

`XmlDSigVerifier` provides a focused, secure, and easy-to-use API for verifying XML signatures. It is designed to replace direct usage of `SignedXml` for verification scenarios, offering built-in security checks and a simplified configuration.

## Features

- **Type-Safe Configuration:** Explicit options for different key retrieval strategies (Public Certificate, KeyInfo, Shared Secret).
- **Enhanced Security:** Built-in checks for certificate expiration, truststore validation, and limits on transform complexity.
- **Flexible Error Handling:** Choose between throwing errors or returning a result object.

## Installation

Ensure you have `xml-crypto` installed:

```bash
npm install xml-crypto
```

## Quick Start

### 1. Verifying with a Public Certificate

If you already have the public certificate or key and want to verify a document signed with the corresponding private key:

```typescript
import { XmlDSigVerifier } from "xml-crypto";
import * as fs from "fs";

const xml = fs.readFileSync("signed_document.xml", "utf-8");
const publicCert = fs.readFileSync("public_key.pem", "utf-8");

const result = XmlDSigVerifier.verifySignature(xml, {
  keySelector: { publicCert: publicCert },
});

if (result.success) {
  console.log("Signature valid!");
  // Access the signed content securely
  console.log("Signed references:", result.signedReferences);
} else {
  console.error("Verification failed:", result.error);
}
```

### 2. Verifying using KeyInfo (with Truststore)

When the XML document contains the certificate in a `<KeyInfo>` element, you can verify it while ensuring the certificate is trusted and valid.

```typescript
import { XmlDSigVerifier, SignedXml } from "xml-crypto";
import * as fs from "fs";

const xml = fs.readFileSync("signed_with_keyinfo.xml", "utf-8");
const trustedRootCert = fs.readFileSync("root_ca.pem", "utf-8");

const result = XmlDSigVerifier.verifySignature(xml, {
  keySelector: {
    // Extract the certificate from KeyInfo
    getCertFromKeyInfo: (keyInfo) => SignedXml.getCertFromKeyInfo(keyInfo),
  },
  security: {
    // Ensure the certificate is trusted by your root CA
    truststore: [trustedRootCert],
    // Automatically check if the certificate is expired
    checkCertExpiration: true,
  },
});

if (result.success) {
  console.log("Signature is valid and trusted.");
} else {
  console.log("Verification failed:", result.error);
}
```

## Advanced Usage

### Reusing the Verifier Instance

For better performance when verifying multiple documents with the same configuration, create an instance of `XmlDSigVerifier`.

```typescript
const verifier = new XmlDSigVerifier({
  keySelector: { publicCert: myPublicCert },
  // Global security options
  security: { maxTransforms: 2 },
});

const result1 = verifier.verifySignature(xml1);
const result2 = verifier.verifySignature(xml2);
```

### Verification Options

The `verifySignature` method accepts an options object with the following structure:

```typescript
interface XmlDSigVerifierOptions {
  // STRATEGY: Choose one of the following key selectors
  keySelector:
    | { publicCert: string | Buffer } // Direct public key/cert
    | { getCertFromKeyInfo: (node) => string | null } // Extract from XML
    | { sharedSecretKey: string | Buffer }; // HMAC

  // CONFIGURATION
  idAttributes?: string[]; // e.g., ['Id', 'ID']
  throwOnError?: boolean; // Default: false (returns result object)

  // SECURITY
  security?: {
    maxTransforms?: number; // Limit transforms (DoS protection)
    checkCertExpiration?: boolean; // Check NotBefore/NotAfter (KeyInfo only)
    truststore?: (string | Buffer)[]; // List of trusted CAs (KeyInfo only)

    // Algorithm allow-lists
    signatureAlgorithms?: Record<string, any>;
    hashAlgorithms?: Record<string, any>;
    // ...
  };
}
```

### Error Handling

By default, `verifySignature` returns a result object. If you prefer to handle exceptions:

```typescript
try {
  const result = XmlDSigVerifier.verifySignature(xml, {
    keySelector: { publicCert },
    throwOnError: true, // Will throw Error on failure
  });
  // If code reaches here, signature is valid
} catch (e) {
  console.error("Signature invalid:", e.message);
}
```

### Handling Multiple Signatures

If a document contains multiple signatures, you must specify which one to verify by passing the signature node.

```typescript
import { DOMParser } from "@xmldom/xmldom";

const doc = new DOMParser().parseFromString(xml, "application/xml");
const signatures = doc.getElementsByTagNameNS("http://www.w3.org/2000/09/xmldsig#", "Signature");

// Verify the second signature
const result = XmlDSigVerifier.verifySignature(
  xml,
  {
    keySelector: { publicCert },
  },
  signatures[1],
);
```
