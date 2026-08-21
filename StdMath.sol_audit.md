## Executive Summary

`stdMath` is a pure Solidity library that provides mathematical utilities:
- `abs` for the absolute value of an `int256`
- `delta` for unsigned difference between two `uint256` or two `int256` values
- `percentDelta` for percentage difference relative to a base value

The library is intended to be inherited/used by a smart contract wallet. Static analysis with Slither only reported an informational pragma issue. Mythril found no issues. Manual review, however, identifies arithmetic edge cases around overflow, division by zero, and compiler-version-dependent behavior. Overall risk is **Medium** — no direct external attack surface, but the library can revert or produce incorrect results for extreme inputs if used without safeguards.

---

## Vulnerability Findings

### 1. Integer overflow in `delta(int256, int256)`
- **Severity:** HIGH
- **Title:** `int256` subtraction can overflow for extreme values
- **Location:** `delta(int256 a, int256 b)`
- **Description:**  
  The `delta(int256, int256)` function computes the absolute difference between two signed integers, likely via a branch such as `a > b ? a - b : b - a` or by using `abs(a - b)`. For extreme values (e.g., `a = type(int256).max`, `b = type(int256).min`), the intermediate subtraction exceeds the `int256` range.  
  - With Solidity `>=0.8.0`, checked arithmetic causes a revert.
  - With Solidity `0.6.x` / `0.7.x`, the subtraction silently wraps around, potentially producing correct results only in some implementations, but this is compiler-dependent and fragile.

- **Impact:**  
  Callers using large `int256` values may experience unexpected reverts or mathematically incorrect results, leading to denial of service or incorrect accounting in wallet logic.

- **Remediation:**  
  Use a safe absolute-difference implementation that converts both operands to `uint256` without intermediate signed overflow, for example via OpenZeppelin’s `SignedMath.abs` or a well-tested math library. Avoid relying on implicit signed overflow behavior. Pin the compiler to a version with checked arithmetic and ensure the code is compiled under that version.

---

### 2. Division by zero in `percentDelta`
- **Severity:** MEDIUM
- **Title:** `percentDelta` does not validate non-zero base
- **Location:** `percentDelta(uint256 a, uint256 b)` and `percentDelta(int256 a, int256 b)`
- **Description:**  
  Both overloads compute a quotient using `a` (for `uint256`) or `abs(a)` (for `int256`) as the divisor. If the base value `a` is zero, the division reverts. No check exists to prevent a zero divisor.

- **Impact:**  
  Any caller that passes `a = 0` will cause a revert. If an attacker can influence the base value to be zero, they can block functionality that relies on the library.

- **Remediation:**  
  Add an explicit check at the beginning of each `percentDelta` function:
  ```solidity
  require(a != 0, "stdMath: zero base");
  ```
  Or use a custom error and document the pre-condition clearly.

---

### 3. Multiplication overflow in `percentDelta`
- **Severity:** MEDIUM
- **Title:** `delta * 1e18` can overflow `uint256`
- **Location:** `percentDelta(uint256 a, uint256 b)` and `percentDelta(int256 a, int256 b)`
- **Description:**  
  The percentage calculation likely uses `delta * 1e18 / a` (or `delta * 1e18 / absA`). If `delta` is large enough, the multiplication exceeds `2^256 - 1`, causing a revert (with Solidity 0.8) or silent wraparound (with earlier versions). In the `int256` overload, the final cast to `int256` may also overflow if the result exceeds `type(int256).max`.

- **Impact:**  
  Large input values can cause unexpected reverts or incorrect percentage results, undermining the reliability of the library.

- **Remediation:**  
  Use a safe `mulDiv` implementation (e.g., OpenZeppelin’s `Math.mulDiv`) or rearrange the arithmetic to avoid intermediate overflow:
  ```solidity
  uint256 result = (delta / a) * 1e18 + (delta % a) * 1e18 / a;
  ```
  For the signed overload, validate that the final result fits in `int256` before casting.

---

### 4. Complex Solidity version pragma
- **Severity:** LOW
- **Title:** Broad compiler version range
- **Location:** Line 1 (`pragma solidity >=0.6.2 <0.9.0`)
- **Description:**  
  The pragma allows compilation with Solidity 0.6.x, 0.7.x, and 0.8.x. These versions have fundamentally different arithmetic overflow handling (wrapping vs. reverting) and language features.

- **Impact:**  
  Inconsistent deployed bytecode and behavior between compiler versions. The same source can be vulnerable or safe depending on the compiler used, making security auditing and deployment error-prone.

- **Remediation:**  
  Pin the compiler to a specific version, e.g., `pragma solidity 0.8.20;` or use a restrictive patch `^0.8.20`. Ensure all code is compiled and tested with that exact version.

---

### 5. State variable should be `constant`
- **Severity:** INFO
- **Title:** `INT256_MIN` not marked as a constant in SSIR
- **Location:** `[STATE] int256 private INT256_MIN`
- **Description:**  
  The library declares `int256 private INT256_MIN`. In Solidity, libraries cannot have non-constant state variables; this should be a constant. If it is not declared as `constant`, the library may fail to deploy or incur unnecessary storage access overhead.

- **Impact:**  
  Deployment failure or inefficient storage reads. The code likely intends a constant, but it is worth verifying.

- **Remediation:**  
  Declare as:
  ```solidity
  int256 private constant INT256_MIN = type(int256).min;
  ```

---

## Risk Rating

**Overall score: 6 / 10 — Medium Risk**

**Justification:**  
The library is a pure arithmetic utility with no external calls, no delegatecall, and no storage-modifying logic. The main risks are edge-case arithmetic failures (overflow, division by zero) and the broad compiler pragma. No critical vulnerabilities such as reentrancy, access control flaws, or unauthorized fund movement were identified. However, the arithmetic issues can cause denial of service or incorrect calculations if used with extreme values, and the compiler-version ambiguity increases the chance of subtle bugs.

---

## Recommended Actions

1. **Pin Solidity version** to `0.8.x` (e.g., `0.8.20`) and recompile/test with that exact version.
2. **Replace custom signed integer arithmetic** with a well-tested math library (e.g., OpenZeppelin’s `SignedMath` and `Math`).
3. **Add explicit zero-base checks** in both `percentDelta` overloads.
4. **Use safe `mulDiv` or equivalent** to avoid multiplication overflow in percentage calculations.
5. **Add unit tests for extreme values** including `type(int256).min`, `type(int256).max`, `type(uint256).max`, and zero bases.
6. **Verify that `INT256_MIN` is a constant** and not a regular state variable.
7. **Document preconditions and postconditions** for each function so integrators understand the required invariants.

Note: Review with a human auditor before deploying contracts holding significant value.