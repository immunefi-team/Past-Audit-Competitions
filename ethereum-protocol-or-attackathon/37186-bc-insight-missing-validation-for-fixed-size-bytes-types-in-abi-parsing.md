# #37186 \[BC-Insight] Missing Validation for Fixed-Size bytes Types in ABI Parsing

**Submitted on Nov 27th 2024 at 22:43:54 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37186
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

In the NewType function within type.go, there is a lack of validation for the size of fixed-length bytes types.

## Vulnerability Details

According to the argument encoding for the solidity, the bytes should limited to 32 bytes:\
<https://docs.soliditylang.org/en/v0.8.23/abi-spec.html>

\| bytes: binary type of M bytes, 0 < M <= 32.

However, such limitation is missing in the accounts/abi/type.go:

```
   case "bytes":
       if varSize == 0 {
           typ.T = BytesTy
       } else {
           typ.T = FixedBytesTy
           typ.Size = varSize
       }
```

Also, it should be noted the validation has been added in Geth: <https://github.com/ethereum/go-ethereum/pull/26075>

## Impact Details

Without validation, the code might accept invalid bytes types with sizes outside the allowed range and break the spec.

## References

* <https://docs.soliditylang.org/en/v0.8.23/abi-spec.html>
* <https://github.com/ethereum/go-ethereum/pull/26075>

## Proof of Concept

## Proof of Concept

```
func TestNewFixedBytesOver32(t *testing.T) {
   _, err := NewType("bytes64", "", nil)
   if err == nil {
       t.Errorf("fixed bytes with size over 32 is mistakenly allowed")
   }
}
```

```
--- FAIL: TestNewFixedBytesOver32 (0.00s)
   /Users/xxx/erigon/accounts/abi/type_test.go:376: fixed bytes with size over 32 is mistakenly allowed
FAIL
FAIL    github.com/erigontech/erigon/accounts/abi   0.321s
FAIL
```
