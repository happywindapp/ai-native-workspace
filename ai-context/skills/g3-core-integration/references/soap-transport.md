# G3SB SOAP Transport

How HSC's OMS code (Bond + Carbon) physically talks to G3 today. The operation catalog is in `api-operations.md`; error codes in `error-codes.md`.

## Two transports

| Transport | Where | Status |
|---|---|---|
| **Direct SOAP/XML** to `G3SBApi_Url`, op `messageTransfer` | BondOMS `handler-core-api.go`, Carbon-OMS `handler-core-api.go` | What the code uses **today**. |
| **Core API Gateway (REST/JSON)** at `{{coreAPIHost}}`, `/equity/*` namespace | designed target | Carbon settlement spec is written against it; OMS not yet migrated (TODO — confirm with IT.Service). |

The Core API Gateway (`CoreApiGW`, module path e.g. `C:\_core_api_gateway\CoreApiGateway`) sits *in front of* G3 — it is not G3's source. G3 itself is a closed third-party AFE product.

## SOAP envelope (`messageTransfer`)

```xml
POST {G3SBApi_Url}
Content-Type: text/xml
SOAPAction: urn:messageTransfer

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="...">
  <soapenv:Body>
    <web:messageTransfer>
      <web:CompanyID>HSC</web:CompanyID>
      <web:UserID>{G3SBApi_Username}</web:UserID>
      <web:UserPassword>{G3SBApi_Password}</web:UserPassword>
      <web:RequestXML>
        <![CDATA[<REQUEST ServiceApp="" ID="{randSeq}">
          <ZIP PlainTextMode="Y">
            <REQUEST Type="CreateCashHold" ID="{randSeq}">
              <VALUEDATE>yyyy-MM-dd</VALUEDATE>
              <TRANSACTIONREFERENCE>yyyyMMddHHmmss{randSeq}</TRANSACTIONREFERENCE>
              <ACCOUNTID>{accountNo}</ACCOUNTID>
              <HOLDTYPE>D</HOLDTYPE>
              <AMOUNT TYPE="DECIMAL">{amount}</AMOUNT>
              <REMARK>Cash hold | {accountNo} | {amount}</REMARK>
              <AUTOAPPROVALFLAG>Y</AUTOAPPROVALFLAG>
            </REQUEST>
          </ZIP>
        </REQUEST>]]>
      </web:RequestXML>
    </web:messageTransfer>
  </soapenv:Body>
</soapenv:Envelope>
```

Layers, outermost → innermost:
1. **SOAP envelope** — standard `soapenv:Envelope` / `Body`.
2. **`messageTransfer`** — the single G3SB SOAP operation. Carries auth + `RequestXML`.
3. **Auth** — `CompanyID` (always `HSC`), `UserID`, `UserPassword`. From env `G3SBApi_Username` / `G3SBApi_Password`.
4. **`RequestXML`** — a CDATA-wrapped inner XML document: outer `<REQUEST ServiceApp="" ID="">` wrapping a `<ZIP>` block.
5. **`<ZIP>` block** — `PlainTextMode` attribute controls encoding (below).
6. **Inner `<REQUEST Type="...">`** — the actual operation; `Type` selects `CreateCashHold` / `CreateInstrumentHold` / `CreateCashRelease` / `CreateInstrumentRelease` / `CreateAccountContract` / `CreateAccount` / etc.

## ZIP / Base64 encoding

The `<ZIP>` element name is literal; the `PlainTextMode` attribute decides whether content is actually compressed:

| `PlainTextMode` | Meaning |
|---|---|
| `"Y"` | **Plain text** — inner `<REQUEST>` sits as-is inside `<ZIP>`. No compression. This is what the OMS code sends. |
| `"N"` | **Compressed** — inner XML → PKZIP → Base64 → text content of `<ZIP>`. The g3sb-api.md v1.09 spec describes a `<Zip>{Base64 of PKZIP of XML}</Zip>` form for this mode. |

> g3sb-api.md v1.09 documents the *compressed* form as "mandatory ZIP + Base64". The live OMS code uses `PlainTextMode="Y"` (no compression). Treat plain-text mode as the current reality; be ready to compress if a G3 environment requires `PlainTextMode="N"`.

Optional message signing: HMAC-SHA256 or RSA, carried in a `<Signature>` element. Optional in HSC's usage.

## Field conventions

| Field | Rule |
|---|---|
| `ID` (both REQUEST levels) | random sequence (`randSeq`, ~10 chars). Correlation id. |
| `VALUEDATE` | `yyyy-MM-dd`. **Must equal G3 core business date** — see `integration-rules.md` Rule 4. |
| `TRANSACTIONREFERENCE` | `yyyyMMddHHmmss{randSeq}`. Unique per day per account. Drives idempotency. Keep `time.Now()` for this even when `VALUEDATE` is core-date — it is an identifier, not a business date. |
| `ACCOUNTID` | investor account number. Bond: `\d{3}[A-Z]{3}_[A-Z]{2}`. Carbon: `REGEX_VALIDATE_BP_ACCOUNT`. |
| `HOLDTYPE` | `D` cash · `T` instrument temp · `B` block · `7` taxable bonus. |
| `AMOUNT TYPE="DECIMAL"` | integer VND string — `fmt.Sprintf("%.0f", amount)`. VND has no sub-unit. |
| `AUTOAPPROVALFLAG` | `Y` — auto-approve the hold/release without a separate G3 approval step. |

## HTTP client / TLS

- Carbon-OMS `netClient`: timeout **60s**, dialer 60s, TLS handshake 60s.
- `InsecureSkipVerify: true` is set — flagged for PROD review (do not rely on it being safe).
- A SOAP call that cannot reach G3 surfaces as `connection refused` / `dial tcp ...` — an infra/DevOps issue, not a code bug (see `error-codes.md` mode=1).

## Config (env vars)

| Env var | Purpose |
|---|---|
| `G3SBApi_Url` | SOAP endpoint URL. **Trim it** — k8s/kustomize env files do not strip quotes; a stray trailing `"` yields a URL ending `/%22`. Carbon-OMS does `strings.Trim` at `main.go` startup + logs the effective URL. |
| `G3SBApi_Username` | SOAP `UserID`. |
| `G3SBApi_Password` | SOAP `UserPassword`. |
| `REGEX_VALIDATE_BP_ACCOUNT` | account-number validation regex (default `^[a-zA-Z0-9]{1,}$`). |
| `ENABLE_G3` | Carbon-OMS — `false` skips all G3 calls (test workaround). |
| `ENABLE_G3_DATE` | Carbon-OMS — `true` auto-fetches G3 core business date for `VALUEDATE`. |

The OMS also holds a direct read-only DB pool to G3's MSSQL (`dbG3SB`) for queries such as account-contract / fee lookup and business-date — see `integration-rules.md`.

> The OMS code may log the SOAP request including `UserID` / `UserPassword`. Credentials seen in logs (e.g. `UserID=TPRL` / `UserPassword=1234`) must be verified as the correct per-environment creds and never echoed.