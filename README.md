## [BUG] Panic in SlashAndResetCounters can halt chain when validator not found

## Bug Description

Check the Oracle module and find `panic()` used to handle errors in `SlashAndResetCounters`. Using panic in BeginBlocker is dangerous because it can crash all validators and stop the chain.

The problem is in `x/oracle/keeper/slash.go`:

**Line 49:**
```go
validator, err := k.StakingKeeper.Validator(ctx, operator)
if err != nil {
    panic(err)
}
```

**Line 54:**
```go
consAddr, err := validator.GetConsAddr()
if err != nil {
    panic(err)
}
```

This function is called from BeginBlocker:

```
x/oracle/abci.go BeginBlocker()
    → k.SlashAndResetCounters(ctx)
        → panic(err) kalo validator gak ketemu
```

- **Apa yang rusak?** Error di SlashAndResetCounters pake panic, bukan handle error yang tepat
- **Module mana yang kena?** Oracle module (`x/oracle/keeper/slash.go`)
- **Kenapa ini bug?** Panic di BeginBlocker bakal bikin semua validator crash dan chain berhenti. Best practice Cosmos SDK bilang jangan pake panic di ABCI methods.

---

## Steps to Reproduce

1. validator yang punya VotePenaltyCounter pada oracle module
2. tombstone/hapus validator dari staking (tapi penalty counter masih ada)
3. Tunggu sampe SlashWindow selesai
4. BeginBlocker panggil SlashAndResetCounters
5. Kode coba ambil data validator → error → panic → chain berhenti

Buat verifikasi:
```bash
grep -n 'panic' ~/kiichain/x/oracle/keeper/slash.go
# Output:
# 49:                             panic(err)
# 54:                                     panic(err)
```

Hasil PoC test:
```bash
cd ~/kiichain && go test -v -run TestPanicInSlashAndResetCounters ./x/oracle/keeper/

# Output:
# Panic value:    validator does not exist
# /root/kiichain/x/oracle/keeper/slash.go:49
```

---

## Expected Behavior

Semua validator harusnya handle error dengan baik. Error harusnya di-log terus di-skip, bukan crash node-nya. Sesuai docs Cosmos SDK, ABCI methods (BeginBlocker, EndBlocker) gak boleh panic.

---

## Actual Behavior

Pas `StakingKeeper.Validator()` return error, kode langsung panic:
- Semua validator jalanin kode yang sama di block yang sama
- Semua validator panic barengan
- Semua validator crash
- Chain berhenti total

---

## Environment

- **Kiichain version / commit:** v6.0.0 (commit 4d52338)
- **Network:** testnet oro
- **File:** `x/oracle/keeper/slash.go` lines 49, 54
- **Related:** `x/oracle/abci.go` BeginBlocker

---

## Impact Assessment

**Severity: Critical**

- **Consensus failure:** Iya - semua validator crash di block yang sama
- **Fund loss:** Gak langsung - user gak bisa akses dana selama chain berhenti
- **Security risk:** Iya - bisa di-trigger sengaja
- **Chain halt:** Iya - butuh koordinasi restart

Referensi - bug yang sama ditemuin di audit Nibiru:
https://reports.zellic.io/publications/nibiru/findings/high-xinflation-xoracle-panic-in-endblock-hooks-will-halt-the-chain

---

## Suggested Fix

```go
// Sebelum (vulnerable)
validator, err := k.StakingKeeper.Validator(ctx, operator)
if err != nil {
    panic(err)
}

// Sesudah (safe)
validator, err := k.StakingKeeper.Validator(ctx, operator)
if err != nil {
    ctx.Logger().Error("failed to get validator", "operator", operator.String(), "error", err)
    return false, nil
}
```
