# Crypto Engines

The Go standard library crypto library is very well done and widely used.
The problem, however, comes when a user wants to swap out the implementation
for another. This has already happened even upstream with the dev.boringcrypto [0]
branch. Use cases for using an alternate implementation include FIPS compliance
(where a user may need to call into a FIPS certified crypto library), and
interfacing with external crypto devices (e.g. accelerator cards, specialty
hardware).

It would be easy enough to maintain a package that does this already, however
the problem comes when trying to swap crypto implementation in a project
you do not own when building against 3rd party code. This would also ensure
consistency so that there is a formal way to declare that all or part of the
crypto operations provided by the standard library should be implemented
by some outside package.

This proposal outlines a way for alternate implementations of the
crypto functionality to be registered via a function call, and allow the
alternate crypto implementation to live in a separate package maintained
outside of the Go tree.

## Engine implementation

Users should be able to register a single global engine which if present
will be used instead of the standard library implementation. The design allows
engines to replace all *or* part of the standard library implementations.

The loading and registering of an engine should be simple and straighforward.
The user is presented with 2 options:

1. Load an engine dynamically by having the engine source code and dynamically
   registering the engine struct itself, or
2. Load an engine via Go plugin functionality [1]

The basic outline for these functions would look like:

```
package crypto

var engine Engine

// RegisterEngine registers the engine specified by `e`.
// This engine will be used for any crypto functionality
// it provides, falling back to the standard library for
// any functionality not provided by the engine.
func RegisterEngine(e Engine) {
	if engine != nil {
		panic("engine already registered")
	}
	engine = e
}

// RegisterEnginePlugin will load the plugin located
// at `path` and subsequently create and register 
// an engine from it.
func RegisterEnginePlugin(path string) {
	...
}
```

It would also be very easy to unregister an Engine, if need be, via:

```
func EngineCleanup() {
	...
}
```

The standard library code will switch based on the engine and specific
implementation being non-null. For example:

```
// NewCipher creates and returns a new cipher.Block.
// The key argument should be the AES key,
// either 16, 24, or 32 bytes to select
// AES-128, AES-192, or AES-256.
func NewCipher(key []byte) (cipher.Block, error) {
	k := len(key)
	switch k {
	default:
		return nil, KeySizeError(k)
	case 16, 24, 32:
		break
	}
	if engine != nil && engine.AES() != nil {
		return engine.AES().NewCipher(key)
	}
	return newCipher(key)
}
```

### Engine interface

All engines to be registered internally must conform to a specific interface.
The interfaces will be tiered and focus on specific areas of cryptography.

```
type Engine interface {
	AES()    AESEngine
	Cipher() CipherEngine
	DES()    DESEngine
	...
	TLS() TLSEngine
}

type AESEngine interface {
	NewCipher(key []byte) (cipher.Block, error)
}

...
```

The goal would be that all crypto operations could be handled by an engine
implementation, including TLS specific functionality (PRF, et al).

### Real world use cases

The upstream Go repository currently has a branch [0] that is being maintained
separate from the main `master` branch which adds support for calling into
BoringCrypto [2]. Instead of maintaining an entirely separate branch, the
functionality could be rewritten as an engine and included in any project
that may need it.

Additionally, if a Go user has a requirement that their code meet FIPS
requirements and would not like to use (or cannot use) the 
dev.boringcrypto branch, this new functionality would allow the user to
pick their crypto library of choice (OpenSSL, LibreSSL, et al).

Finally, if there are users who wish to take advantage of certain hardware
for crypto, that too can be accomplished via engines.

## Summary

Engines would provide an easy way to swap all or part of the standard library
crypto functionality with one provided by the user. There are already real
world use cases and examples of folks using other crypto libraries either
by creating their own package, or even by upstream maintaining a separate
branch to support BoringSSL.

This proposal suggests a way to remove the maintenance burdon and make the
Go standard library crypto implementation more extensible.

---

[0] https://github.com/golang/go/tree/dev.boringcrypto
[1] https://golang.org/pkg/plugin/
[2] https://boringssl.googlesource.com/boringssl/
