# Annotated MT `.fin` Samples

Real carbon/TPRL samples with tag-by-tag breakdown. Live sample files are typically found under a project's `tmp_extract/` or `documents/` folder — search for `*.fin` or `Dien mau` to locate them in the current workspace.

## MT598.301 — Register carbon trading account (TVLK→VSDC)

```
{1:F01VSDHSCXXASTP0001000205}{2:I598VSDSVN03XXXN}{4:
:20:DKGDTPRLHSC005                     ← unique request id
:12:301                                ← sub-code 301 = register carbon account
:77E:NORMAL                            ← semantic mode
:16R:GENL
:23G:NEWM                              ← new message (ACLS-flow would still be NEWM)
:22H::ACCT//AOPN                       ← account action: open (ACLS = close)
:98A::PREP//20230424
:16S:GENL
:16R:REGDET
:97A::SAFE//011C001143                 ← trading account number
:95Q::INVE//Cao Hoanh Anh              ← investor name (telex-encode if diacritics)
:95S::ALTE//VISD/IDNO/VN/007082003064  ← ID: type/country/number
:98A::ISSU//20100910                   ← ID issue date
:94G::ISSU//CATPHCM                    ← issuing place
:94G::EMAI//viet.va(at)hsc.com.vn      ← email uses (at), NOT ?
:94G::PHON//01246325813
:94G::ADDR//Lau 5,6 so 76 Le Lai...
:70E::ADTX//011C001143/TYPE//DOMIND    ← account + investor type DOMIND/FORIND/DOMCORP/FORCORP/GOVT
:22F::TPTY//EMIT                       ← party type: EMIT / PROJ / ORGA
:22F::ACTP//QUOT                       ← account type: QUOT (quota) / CRDT (credit)
:16S:REGDET
-}{5:{MAC:00000000}{CHK:F1DBCA886BBF}{TNG:}}
```
Reply: same structure, `:16R:STAT` `:25D::IPRC//PACK` (accept) or `REJT` + `:70D::REAS//`, `:20C::RELA//` echoes the request `:20:`.

## MT518 — Trade result + settlement obligation (VSDC→TVLK)

```
{1:F01VSDSVN03AXXX2222123456}{2:O518...VSDVDSXX...N}{4:
:16R:GENL
:20C::SEME//0126206435                 ← VSDC trade ref
:23G:NEWM                              ← :23G:CANC = trade removed/adjusted
:22F::TRTP//TRAD
:16R:LINK
:20C::TRRF//VNCA0012508092025          ← exchange match id
:16S:LINK
:16S:GENL
:16R:CONFDET
:98C::TRAD//20250908141214             ← trade datetime
:98A::SETT//20250908                   ← settlement date (carbon = T+0)
:90B::DEAL//ACTU/VND100000             ← price per unit
:19A::SETT//VND1000000                 ← total settlement amount
:22H::BUSE//BUYI                       ← side: BUYI / SELL
:22H::PAYM//APMT                       ← APMT = DVP, FREE = free of payment
:16R:CONFPRTY
:95R::BUYR//VSDVDSXX                   ← buyer BIC
:16S:CONFPRTY
:36B::CONF//UNIT/100                   ← quantity
:35B:/VN/VNCA00125                     ← carbon code
:16S:CONFDET
:16R:SETDET
:22F::SETR//TRAD
:16R:SETPRTY
:95P::REAG//VSDVDSXX                   ← receiving agent
:20C::PROC//VNCA00125080920251         ← process / order id
:70D::REGI//021CD12581                 ← counterparty account
:16S:SETPRTY
:16S:SETDET
-}{5:...}
```

## MT598.305 — Confirm settlement obligation (TVLK→VSDC, response to MT518)

```
:20:DKGDTPRLHSC005
:12:305
:77E::PROC//TRADE
:16R:GENL
:23G:NEWM
:16R:LINK
:20C::PREV//<MT518 id>                 ← references the obligation
:16S:LINK
:16R:STAT
:20C:RELA//<exchange match id>
:25D:STAT//CONF                        ← CONF = confirm, REJT = reject (+ :70D::REAS//)
:16S:STAT
:16S:GENL
```

## MT544 — Settlement credit confirmation (VSDC→TVLK buyer)

Key sections: `:13A::LINK//518` links to the obligation; `:16R:FIAC` `:36B::SETT//UNIT/100` `:97A::SAFE//<buyer account>` = units credited; `:16R:SETDET` `:22F::STCO//PHYS` (physical/full transfer) or `NPAR`; `:95P::DEAG//<seller BIC>` = delivery source. MT546 mirrors this as a debit on the seller side.

## MT598.000 — Invalid trade (VSDC→TVLK, error)

```
:20:0126216441
:12:000
:77E:PROC//ERRTRADE
:16R:GENL
:23G:CANC                              ← trade cancelled
:22F::TRTR//TRAD
:16R:LINK
:20C::TRRF//VNCA00125080920256         ← original trade ref
:70D::REAS//1                          ← reason code/text
:16S:LINK
:16S:GENL
```
Sent to both buy and sell TVLK. TVLK should expect this for both sides.

## MT900 — Bank payment confirmation (NHTT→VSDC)

```
:20:NHTTXN100004                       ← payment reference
:21:001669Q16                          ← related reference
:25:VSDTCBSX                           ← account number
:32A:250908VND50000                    ← value date(YYMMDD) + currency + amount
:52A:VCBVVNVX                          ← sending bank BIC
:72:105C123953/VNCA0012508092025       ← instruction: account / trade ref
```

## MT598.010 — Allocation notification (TVLK→VSDC)

`:12:010` (or `308` variant), `:77E:CASH`. `:70E::SPRO//` is **multiline** — subsequent lines carry: allocation date / settlement date / quantity / unit price / total value. Non-standard SWIFT but allowed by VSDC carbon spec; parsers must read the continuation lines.

See `swift-mt-tags.md` for tag formats, `vsdc-encoding-rules.md` for telex encoding of names/addresses.