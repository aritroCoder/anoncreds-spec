## AnonCreds Revocation Data Flow

AnonCreds includes a mechanism that supports the revocation of verifiable
credentials. This mechanism includes:

- An [[ref: issuer]] setting up to issue revocable credentials.
- An [[ref: issuer]] issuing revocable credentials.
- An [[ref: issuer]] activating or revoking issued credentials.
- A [[ref: verifier]] requesting a presentation to include a non-revocation proof
  for one or more revocable credentials.
- A [[ref: holder]] generating based on the presentation request of the verifier a
  non-revocation proof for attributes derived from revocable credentials.
- A [[ref: verifier]] verifying a non-revocation proof included in a
  presentation from a [[ref: holder]].

A fundamental goal of AnonCreds is to not provide a correlatable identifier for
either a [[ref: holder]] or a credential as part of generation and verification
of an AnonCreds presentation. Applying that goal to revocation means that the
revocation mechanism must support the [[ref: holder]] proving a credential used
in generating a presentation is not revoked without providing a correlatable
identifier for that credential or the holder itself. As such, the AnonCreds
revocation mechanism uses a Zero Knowledge Proof (ZKP) that allows the [[ref:
holder]] to prove a credential they hold is not revoked without revealing an
identifier for their credential or the holder.

### AnonCreds Issuer Setup With Revocation

The details of [[ref: issuer]] setting up revokable credential types are covered
in the [issuer setup](#issuer-create-and-publish-revocation-registry-objects)
section of this specification. Note the [warning and recommendation against the
use of `ISSUANCE_ON_DEMAND`](#recommend-not-using-issuanceondemand) in that part
  of the specification.

### AnonCreds Issuance with Revocation

The details of an [[ref: issuer]] issuing a revokable credential to a [[ref:
holder]] are covered in the [issuance data
flow](issue-credential) section of this specification.

### AnonCreds Credential Activation/Revocation and Publication

When an [[ref: issuer]] decides to revoke a previously issued credential (or
activate a previously inactive/revoked credential), they do so by publishing
another instance of the [[ref: RevRegEntry]] object. Recall from the issuer
setup section, the specification about [creating and publishing the first [[ref:
RevRegEntry]]](data_flow_setup.md#creating-the-initial-revocation-registry-entry-object)
for a [[ref: RevReg]]. In that process, the accumulator for the initial state of
the [[ref: RevReg]] is published. When subsequent [[ref: RevRegEntry]]
transactions are published to the ledger, each includes an updated value of the
accumulator. The update of the accumulator is necessary with each revocation or
(re)activation of a credential or set of credentials since the last published
[[ref: RevRegEntry]]. This is because only the factors (all factors are listed
in the respective tails file) of credentials which are active (meaning not being
revoked) contribute to the accumulator. Therefore in addition to the updated
accumulator value, every [[ref: RevRegEntry]] contains lists of indices of
credential factors which have been either revoked or (re)activated within the
[[ref: RevRegEntry]]. This list of factor indices is a so called [[ref: Witness
Delta]] and enables the [[ref: Holder]] to successfully generate a proof of non
revocation. 

An example of the
data in the [[ref: RevRegEntry]] is shown in the following example of a [[ref:
RevRegEntry]], pulled from [this transaction on the Sovrin
MainNet](https://indyscan.io/tx/SOVRIN_MAINNET/domain/140326).

``` json
"data": {
    "revocDefType": "CL_ACCUM",
    "revocRegDefId": "4xE68b6S5VRFrKMMG1U95M:4:4xE68b6S5VRFrKMMG1U95M:3:CL:59232:default:CL_ACCUM:4ae1cc6c-f6bd-486c-8057-88f2ce74e960",
    "value": {
        "accum": "21 116...567",
        "prevAccum": "21 128...C3B",
        "issued": [
        ],
        "revoked": [
            172
        ]
    }
},
```

In the above:

- `revocDefType`: is defined by the comparable entry in the [[ref: RevReg]], and for this
  version of AnonCreds is hard-coded to `CL_ACCUM`.
- `revocRegDefId`: is the Id of the [[ref: RevReg]] to which this entry is
  being added.
- `accum`: is the new value of the accumulator based on the state of the
  credentials in the [[ref: RevReg]], including those listed in this transaction. The
  value is calculated by the Issuer based on the credential state changes, and
  submitted as part of the transaction, and verified by the ledger before being
  published.
- `prevAccum`: is the previous value of the accumulator. The value is supplied
  by the Issuer in the transaction request, and verified by the ledger before
  being published.
- `issued`: an array (possibly empty or not supplied) of the indices within the
  [[ref: RevReg]] of the credentials whose state has changed to `active` (also known as
  `not revoked`) since the last [[ref: RevRegEntry]] was published to the ledger.
- `revoked`: an array (possibly empty or not supplied) of the indices within the
  [[ref: RevReg]] of the credentials whose state has changed to `revoked` (also known as
  `not active`) since the last [[ref: RevRegEntry]] was published to the ledger.

In the example transaction above no credentials are `issued` (meaning changed
from status `revoked` to `issued`) and only one, the credential with index
`172`, is changed to `revoked`. Both lists can have an arbitrary number of
entries, up to the total number of credentials in the [[ref: RevReg]].

The algorithm to calculate the value of a [[ref: RevRegEntry]] accumulator at
any time is the same: determine the (modulo) product of the primes for each
non-revoked credential in the [[ref: Revocation Registry]], as described
[here](#creating-the-initial-revocation-registry-entry-object).

::: note

The [[ref: issuer]] MUST track of the revocation status of all of the
credentials within a [[ref: RevReg]] so that it can both calculate the correct
accumulator and send to the [[ref: VDR]] accurate lists (`issued` and `revoked`)
of the indices of the credentials whose status has changed since the last [[ref:
RevRegEntry]] was published. If the list and accumulator published to [[ref: VDR]] get out of
sync  a [[ref: holder]] will not be able to generate a valid NRP.

A [[ref: VDR]] publishing a [[ref: RevReg]] MAY perform its own calculation of
the accumulator based on the list updates received in a [[ref: RevRegEntry]]
transaction to ensure that the calculation of the accumulator after all of the
revocation status updates to the credentials within the [[ref: RevReg]] have
been applied, rejecting the transaction if the calculated accumulator does not
match that from the [[ref: issuer]].

If an [[ref: issuer]]'s local revocation information gets out of sync with what
is in the VDR, the [[ref: issuer]] MUST rationalize the differences and produce
a [[ref: RevRegEntry]] transaction that accounts for both the last published
[[ref: RevRegEntry]] published in the [[ref: VDR]] and the desired revocation
status of all of the credentials in the [[ref: RevReg]].

:::

The [[ref: holder]] is not involved in the process of revoking a credential.
There is no technical requirement for an [[ref: issuer]] to notify the [[ref:
holder]] that a credential they were issued has been revoked. That said, it is a
courtesy that may improve the user experience of the [[ref: holder]]. [Aries RFC
0183 Revocation
Notification](https://github.com/hyperledger/aries-rfcs/tree/main/features/0183-revocation-notification)
is an example of how that can be done. Even if not notified by the [[ref:
issuer]] of the revocation of a credential, the [[ref: holder]] can detect their
credential has been revoked when they retrieve the list of revoked credentials
from the [[ref: VDR]] and discover the index of their credential in the list.

### AnonCreds Presentation Request with Revocation

Carrying out an AnonCreds presentation with revocation is a two-step process, beginning with a
request from the [[ref: verifier]] asking the [[ref: holder]] to include a
non-revocation proof (NRP) in the presentation, and then the [[ref: holder]]
creating the NRP and including it in the presentation sent to the [[ref:
verifier]].

The verifier requesting a non-revocation proof, and the [[ref: holder]]
generating the non-revocation proof are covered in the sections of this
specification about [requesting](#request-non-revocation-proofs) and
[generating](#generate-non-revocation-proofs) presentations, respectively.

### AnonCreds Verification with Revocation

A [[ref: verifier]] receives the presentation from the [[ref: holder]] and
processes the [non-revocation-related parts of the presentation](#verify-presentation) and
the [revocation-related parts of the presentation](#verify-non-revocation-proof)
(if any) in the presentation. The resulting status of the presentation combines the
verification outcomes from processing all proofs within the presentation. If
verification of one or more of the embedded proofs is unsuccessful, the
presentation is rejected as unverifiable.
