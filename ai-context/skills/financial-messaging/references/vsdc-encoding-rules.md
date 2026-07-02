# VSDC — Vietnamese Encoding, Cut-off Times, Validation

Operational rules for producing VSDC-acceptable `.fin` messages.

## Vietnamese → Latin telex encoding

SWIFT FIN allows only: `a-zA-Z0-9 / - ? : ( ) . , ' +` plus `<CR>` and `<Space>`. Vietnamese diacritics are encoded telex-style, wrapped in `?…?` markers.

### Vowel modifiers
| VN | Encoding |
|---|---|
| Ă | `?AW?` |
| Ơ | `?OW?` |
| Ư | `?UW?` |
| Â | `?AA?` |
| Ê | `?EE?` |
| Ô | `?OO?` |
| Đ | `?DD?` |

### Tone marks (telex letter inside the `?…?`)
`f`=huyền, `s`=sắc, `r`=hỏi, `x`=ngã, `j`=nặng.
- `CÔNG TY SỮA` → `C?OO?NG TY S?UWX?A`
- Tone applies within the marker: Ữ = Ư + ngã → `?UWX?`.

### Special characters
`/` → `?_?` · `&` → `?_38?` · `#` → `?_35?` · `%` → `?_37?` · `\` → `?_92?`

### Email
`@` → `(at)` — **no `?` markers** for email addresses specifically (e.g. `viet.va(at)hsc.com.vn`).

### CRITICAL — length validation
Field length limits (e.g. `35x`) are counted on the **Latin form AFTER conversion**. `KHÓA` (4 chars) becomes `KH?OS?A` (7 chars). Always encode first, then validate length.

## Cut-off times (VSDC Quy chế Ver03, Phụ lục I)

| Time | Event |
|---|---|
| 09:00–16:00 | TVLK deposits cash to NHTT |
| **15:30** | TVLK confirms/rejects trade result (MT598.305); submits Mẫu 11/TTCB to remove payment |
| **15:45** | VSDC processes post-trade errors + sends adjusting MT518 |
| **16:30** | TVLK notifies cash+carbon allocation (MT598.010); cut-off for payment removal |
| **17:00** | VSDC sends FileAct reports (`.par` + `.csv`); TVLK reconciles |

## Gateway validation

- **ISN uniqueness:** Block 1 ISN must be unique within a session. Duplicate → NAK `[REQUESTID: duplicate]`.
- **Tag 20 uniqueness:** `:20:` transaction reference must be unique per partner.
- **Block 5 MAC/CHK:** checksum validated; invalid → reject at gateway.
- **Character set:** only the allowed set above; everything else must be telex-encoded.
- **`:23G:DUPL`** marks an intentional retransmission of an already-sent message.

## Status & processing codes

| Code | Field | Meaning |
|---|---|---|
| NEWM / CANC / DUPL / INST | 23G | New / Cancel / Duplicate / Instruction |
| PACK | 25D::IPRC | Package accepted (account/instruction) |
| ACPT | 25D::IPRC | Accepted (type adjustment) |
| REJT | 25D::IPRC or STAT | Rejected — reason in 70D/70E |
| CONF | 25D::STAT | Settlement confirmed |
| AVAL / PLED | 93A | Available / Pledged (locked) balance |

## FileAct `.par` parameter file

Accompanies each FileAct `.csv` report. Key fields:
```
SwiftTime=<timestamp>
Requestor=o=<SENDER_BIC>, o=swift
Responder=o=<VSDC_BIC>, o=swift
Service=camt.xxx.fisp.rep
RequestRef=<query id>           # MT598.003 query id
TransferRef=<event code>        # e.g. BS001
LogicalName=<report>.txt
PossibleDuplicate=TRUE
Size=<bytes>
```

## Reconciliation report schemas (Quy chế Ver03)

- **Mẫu 05/LKCB** (monthly total balance): Mã HN/TC | Mệnh giá | Loại TK | SL GD | SL chờ TT | SL tạm giữ | Tổng.
- **Mẫu 06/LKCB** (monthly detail balance): STT | Mã HN/TC | Số ĐKSH | Ngày cấp | Loại NĐT | Số TK | Loại TK | SL.
- **Mẫu 10/TTCB (QS001)** (daily EOD trade summary, 16 cols): STT | SHL | Số XN | Mã SP | Loại SP (1=HN,2=TC) | Số TK | Mã TV | Mã NHTT | Giá | Mua SL/GT | Bán SL/GT | TV đối ứng | NHTT đối ứng | Trạng thái.

See `vsdc-mt-catalog.md` for message types, `vsdc-stp-flows.md` for the flows these rules govern.