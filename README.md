# Math
```
Encoder.Encode:
    input[n,l]
    coeff[n,l] // output
    coeff[i] = [w^(-ij)] @ (F_inv * (F * inputFr)[i*l:i*(l+1)]) // i in [0:n], j in [0:l]
    coeff = F * F_inv * (F * input)
          = (F * input)

KzgMultiProofGnarkBackend.ComputeMultiFrameProof:
    proof(f) = F * h(f)
             = F * Toeplitz(f) * s
             = F * (Cyc(f2) * s2)[0:n]
             = F * (F_inv * diag(F * f2) * (F * s2))[0:n]
             = F * (F_inv * (F * f2) * (F * s2))[0:n]
             = F * (F_inv * (coeffStore * FFTPointsT))[0:n]
    // The inner product of two vectors of length l
    (coeffStore * FFTPointsT)[i] = coeffStore[i] @ FFTPointsT[i] // i in [0:2n] 

KzgMultiProofGnarkBackend.computeCoeffStore:
    coeffStore = F * f[2n,l]
    coeffStore.col(j) = F * f2[..., m-j-2l, m-j-l]
SRSTable.Precompute:
    FFTPointsT = F * s[2n,l]
    FFTPointsT.col(j) = F * s2[..., m-j-2l, m-j-l]

variables:
    f: coefficients
constants:
    s: s^0, s^1, ..., s^nl
    F: Fourier Matrix
    l: chunklen
    n: numchunks
theorems:
    proof(f) = F * h(f)
    h(f) = Toeplitz(f) * s
    Cyc(f) = F_inv * diag(F * f) * F
definitions:
    proof(f) = (f(x) - f(s)) / (x - s), x = w, w^2, ..., w^n
```

## Refs on Toeplitz matrix

[1] [Multiplying a Toeplitz matrix by a vector](https://alinush.github.io/2020/03/19/multiplying-a-vector-by-a-toeplitz-matrix.html)

[2] [Fast amortized Kate proofs](https://github.com/khovratovich/Kate/blob/master/Kate_amortized.pdf)

## Refs on proof

[3] [A Universal Verification Equation for Data Availability Sampling](https://ethresear.ch/t/a-universal-verification-equation-for-data-availability-sampling/13240)

[4] [KZG polynomial commitments](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html)

[5] [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)

# Data flow diagram

Recommended VS Code extention: [markdown-mermaid](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid).

```mermaid
flowchart TB

disperser -- []byte --> blobstore

blobstore -- []byte --> encserver
blobstore --> relay.Server.GetBlob

encserver -- []byte --> prover
prover -- []fr.Element --> encoder
encoder -- []coeffs --> prover
prover -- []fr.Element --> prover1
prover1 -- []bn254.G1Affine --> prover
prover -- []encoding.Frame --> encserver

encserver --> encmgr

relay -- []encoding.Frame --> node

dispatcher -- []v2.BlobHeader --> node
node -- Signature --> dispatcher

encserver -- []encoding.Frame --> chunkstore
chunkstore -- []encoding.Frame --> relay

node -- []v2.BlobHeader --> v2.ValidateBatchHeader 
node -- []BlobCommitment.Commitment []encoding.Frame --> v2.ValidateBlobs --> Verifier.UniversalVerify
node -- []BlobHeader.BlobCommitment --> verify[VerifyBlobLength VerifyCommitEquivalenceBatch]
node --> batchstore --> ServerV2.GetChunks

encmgr -- []v2.BlobHeader --> headerstore
headerstore -- []v2.BlobHeader --> newbatch

newbatch -- []v2.BlobHeader --> dispatcher
dispatcher --> Dispatcher.HandleSignatures

newbatch[Dispatcher.NewBatch]
dispatcher[Dispatcher.HandleBatch]
prover[Prover.GetFrames]
prover1[ComputeMultiFrameProof]
encoder[Encoder.Encode]
encmgr[EncodingManager.HandleBatch]
encserver[EncoderServerV2.handleEncodingToChunkStore]

relay[relay.Server.GetChunks]
node[*node* ServerV2.StoreChunks]
batchstore[(node.StoreV2)]

blobstore[(blobstore)]
chunkstore[(chunkstore)]
headerstore[(BlobMetadataStore)]
```

# Data structures
```
encoding.Frame: 
    Proof:  bn254.G1Affine
    Coeffs: fr.Element

v2.Batch:
    ReferenceBlockNumber: uint64
    BlobCertificates:     []v2.BlobHeader

v2.BlobHeader:
    Commitment:       []byte
    LengthCommitment: []byte
    LengthProof:      []byte
    Length:           uint32
```

# Finite field elements
```
fr.Element: [4]uint64
fp.Element: [4]uint64
bn254.G1Affine: [2]fp.Element
```
