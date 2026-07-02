# SWIFT MT — FIN Blocks & Tag Reference

ISO 15022 `.fin` structure and the SWIFT tags used in VSDC carbon messages.

## FIN block structure

```
{1:BASIC HEADER}{2:APPLICATION HEADER}{4:TEXT BLOCK}{5:TRAILER}
```
Block 3 (user header) is omitted in carbon samples.

### Block 1 — Basic header
- TVLK→VSDC input: `{1:F01<SenderBIC 8c>A<SessionNo 4n><ISN 6n>}`
- VSDC→TVLK output: `{1:F01<ReceiverBIC 8c>A<SessionNo 4n><OSN 6n>}`
- ACK/NAK: `{1:A21…}` — service ID `21` = protocol acknowledgment
- `F`=SWIFT, `01`=business message, `21`=ACK/NAK. `A`=logical terminal. ISN/OSN auto-increment, **unique per session**.

### Block 2 — Application header
- Input: `{2:I<MT 3n><ReceiverBIC 8c>X<Priority 1a><DM 1n><OP 3!n>}` — e.g. `{2:I598VSDSVN03XXXN}`
- Output: `{2:O<MT 3n><SenderBIC 8c>X…}`
- Priority `U`(urgent)/`N`(normal). DM (delivery monitoring) `1`/`3` if U, `2`/null if N. OP (obsolescence) `020` if present.
- Receiver BIC ending `03` = carbon market routing; `01` = legacy bond.

### Block 4 — Text block
- Tags `:XXX:` or `:XXX::QUAL//VALUE`, newline-separated, block closes with `-`.
- `16R`/`16S` delimit logical sections: GENL, LINK, REGDET, CONFDET, CONFPRTY, SETDET, SETPRTY, TRADDET, FIAC, STAT.

### Block 5 — Trailer
`{5:{MAC:00000000}{CHK:<checksum>}{TNG:}}` — MAC = message auth code, CHK = integrity checksum.

### ACK/NAK (service ID 21)
`:177:` timestamp · `:451:` `0`=ACK / `1`=NAK · `:405:` error code (e.g. `H80`, or free-text business reason).

## Tag reference

### Structure & references
| Tag | Format | Meaning |
|---|---|---|
| 16R / 16S | — | Begin / End section block |
| 13A | `:4!c//<MT 3n>` | Link to parent MT (`:13A::LINK//518`) |
| 13B | `:4!c//<code>` | Report/statistic code (`:13B::STAT//BS001`) |
| 20 | `[N]16x` | Transaction reference — unique per partner |
| 20C | `:4!c//<value 16x>` | Qualified ref: SEME (semantic), RELA (related), PREV (previous), TRRF (trade ref), PROC (process) |
| 23G | `4!c` | Function: NEWM, CANC, DUPL, INST |
| 12 | `3!n` | MT598 sub-message type |
| 77E | `[N]78x` | MT598 semantic mode: NORMAL/ISIN/DLST/BALANCE/CASH/MODE/ERRTRADE/TRADE |

### Dates & amounts
| Tag | Format | Meaning |
|---|---|---|
| 98A | `:4!c//8!n` | Date YYYYMMDD: PREP, SETT, ISSU, DBIR, DFON |
| 98C | `:4!c//14!n` | DateTime YYYYMMDDHHMMSS: TRAD (trade time) |
| 36B | `:4!c//4!c/15d` | Quantity: SETT/CONF/ESTT + UNIT (`:36B::SETT//UNIT/100`) |
| 90B | `:4!c//4!c/<ccy><amt>` | Price: `:90B::DEAL//ACTU/VND100000` |
| 19A | `:4!c//[N]3!a15d` | Amount: `:19A::SETT//VND1000000` |

### Parties & accounts
| Tag | Format | Meaning |
|---|---|---|
| 95P | `:4!c//<BIC 8c>` | Party BIC: ACOW (acct owner), PSET (place of settlement), DEAG (deliverer), REAG (receiver) |
| 95Q | `:4!c//<name>` | Free-text party name: INVE (investor) |
| 95R | `:4!c//<role>//<id>` | BUYR, SELL, AFFM (affirming party) |
| 95S | `4!c/8c/4!c/2!a/30x` | Alt ID `VISD/<type>/VN/<id>`; type ∈ IDNO/CCPT/CORP/TXID/FIIN/ARNU/OTHR/GOVT |
| 97A | `:4!c//<account>` | Account: SAFE (safe-keeping), OWND (owned) |

### Trade / settlement qualifiers
| Tag | Format | Meaning |
|---|---|---|
| 22F | `:4!c//4!c` | TRTP//TRAD, SETR//TRAD, STCO//{PHYS/NPAR/DLWM}, TRTR//TRAD, **TPTY//{EMIT/PROJ/ORGA}**, **ACTP//{QUOT/CRDT}** |
| 22H | `:4!c//4!c` | ACCT//{AOPN/AOPE/ACLS/TBAC/TWAC/MODE}, BUSE//{BUYI/SELL}, PAYM//{APMT/FREE} |
| 25D | `:4!c//4a` | IPRC//{PACK/ACPT/REJT}, STAT//{CONF/REJT} |
| 35B | `[ISIN]/[/2!a/32x]` | Carbon code: `:35B:/VN/VN2027` or `:35B::ISIN//<code>` |
| 93A | `:4!c//4!c` | Balance: FROM//AVAL, TOBA//PLED (MT508 lock/unlock) |
| 12A | `:4!c//4!c/1!n` | CLAS//NORM/1 (`1`=standard, `2`=restricted transfer) |

### Narrative
| Tag | Format | Meaning |
|---|---|---|
| 70C/70D/70E | `:4!c//[N]35x` | REGI, ADTX (address), FIAN, SPRO (special processing), REAS (reason), TPRO (cancel reason) |
| 94G | `:4!c//[2*35x]` | ADDR, EMAI (email — use `(at)` not `?`), PHON, ISSU |
| 94C | `:4!c//2!a` | Country ISO code |

See `vsdc-encoding-rules.md` for Vietnamese encoding and length rules, `mt-samples.md` for tags in context.