# #37134 \[BC-Insight] Improper secp256k sanitization

**Submitted on Nov 26th 2024 at 17:13:52 UTC by @br0nz3p1ck4x3 for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37134
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/prysmaticlabs/prysm>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

Hey Prysm, Ethereum Foundation, Triagers,

We stumbled upon something during our research and we hope this adds some value to the Prysm team.

## Description

Inside `crypto/ecdsa/utils.go::ConvertToInterfacePubkey()`:

```go
func ConvertToInterfacePubkey(pubkey *ecdsa.PublicKey) (crypto.PubKey, error) {
	xVal, yVal := new(btcec.FieldVal), new(btcec.FieldVal)
	if xVal.SetByteSlice(pubkey.X.Bytes()) {
		return nil, errors.Errorf("X value overflows")
	}
	if yVal.SetByteSlice(pubkey.Y.Bytes()) {
		return nil, errors.Errorf("Y value overflows")
	}
	// @audit does not verify that the pubkey is on the secp256k1 curve
	newKey := crypto.PubKey((*crypto.Secp256k1PublicKey)(btcec.NewPublicKey(xVal, yVal)))
	// Zero out temporary values.
	xVal.Zero()
	yVal.Zero()
	return newKey, nil
}
```

We see that a new public key is created here:

```go
newKey := crypto.PubKey((*crypto.Secp256k1PublicKey)(btcec.NewPublicKey(xVal, yVal)))
```

However, when we look at the implementation of `NewPublicKey()`, we see the following:

```go
// NewPublicKey instantiates a new public key with the given x and y
// coordinates.
//
// It should be noted that, unlike ParsePubKey, since this accepts arbitrary x
// and y coordinates, it allows creation of public keys that are not valid
// points on the secp256k1 curve.  The IsOnCurve method of the returned instance
// can be used to determine validity.
func NewPublicKey(x, y *FieldVal) *PublicKey {
	return secp.NewPublicKey(x, y)
}
```

This function does not check if x and y coordinates are valid points on the secp256k1 curve. This checks out because if we look the `secp.NewPublicKey(x,y)`:

```go
func NewPublicKey(x, y *FieldVal) *PublicKey {
	var pubKey PublicKey
	pubKey.x.Set(x)
	pubKey.y.Set(y)
	return &pubKey
}
```

As we can see from the Proof of Concept, the key is not on the curve but it still passes the initial validation upon communicating with other p2p peers. Even though we could not currently identify a place where this secp256k1 key gets validated such that `IsOnCurve()` is called upon it and it errors, we hope this adds value to the Prysm team as it can prevent nasty bugs.

## Recommended Patch

Inside `ConvertToInterfacePubkey()`, ensure that the key is on the `secp256k1 curve` to ensure that all methods called upon that variable will work and that there will not be any unexpected errors.

## Proof of Concept

## Proof of Concept

Replace the`crypto/ecdsa/utils_test.go` file with the following code:

```go
package ecdsa_test

import (
	"crypto/ecdsa"
	"fmt"
	"math/big"
	"testing"

	gcrypto "github.com/ethereum/go-ethereum/crypto"
	"github.com/libp2p/go-libp2p/core/peer"
	p2p "github.com/prysmaticlabs/prysm/v5/beacon-chain/p2p"
	prysm_ecdsa "github.com/prysmaticlabs/prysm/v5/crypto/ecdsa"
	"github.com/prysmaticlabs/prysm/v5/testing/require"
)

func TestConvertPeerIDToNodeID_InvalidCurvePoint(t *testing.T) {
	// Create coordinates that represent a point not on the secp256k1 curve
	x := big.NewInt(1)
	y := big.NewInt(2) // This y value doesn't satisfy y² = x³ + 7

	pubkey := &ecdsa.PublicKey{
		Curve: gcrypto.S256(),
		X:     x,
		Y:     y,
	}

	// Verify point is not on curve before we start
	require.Equal(t, false, pubkey.Curve.IsOnCurve(pubkey.X, pubkey.Y))

	// Convert to interface pubkey
	interfacePubKey, err := prysm_ecdsa.ConvertToInterfacePubkey(pubkey)
	fmt.Print("Interface pubkey:", interfacePubKey, "\n")
	require.NoError(t, err)

	// Check if its on the curve
	isOnCurve := pubkey.Curve.IsOnCurve(pubkey.X, pubkey.Y)
	fmt.Print("Is the key on the curve?", isOnCurve, "\n")

	// Create peer ID from invalid key
	peerID, err := peer.IDFromPublicKey(interfacePubKey)
	require.NoError(t, err)
	fmt.Print("Peer ID:", peerID, "\n")

	// Try to convert this invalid peer ID to node ID
	nodeID, err := p2p.ConvertPeerIDToNodeID(peerID)
	fmt.Print("Node ID:", nodeID, "\n")
	require.NoError(t, err)

	// Show that we got a valid-looking node ID despite using an invalid curve point
	require.NotEqual(t, [32]byte{}, nodeID)
	t.Logf("Generated node ID %x from invalid public key", nodeID)
}
```

`cd` into `crypto/ecdsa/` and run:

* `go test -timeout 30s -run ^TestConvertPeerIDToNodeID_InvalidCurvePoint`

This will print out the following:

```zsh
Interface pubkey:&{{[1 0 0 0 0 0 0 0 0 0]} {[2 0 0 0 0 0 0 0 0 0]}}
Is the key on the curve?false
Peer ID:16Uiu2HAkuRfynyeQUyaKG6D44mPBuzAaiqVCWqAW9GHmv9rSiQ3z
Node ID:c0d54f3e9a77589a14146c473344c6f6eeca588be132142db0a32c71abfaae7b
PASS
ok  	github.com/prysmaticlabs/prysm/v5/crypto/ecdsa	0.674s
```
