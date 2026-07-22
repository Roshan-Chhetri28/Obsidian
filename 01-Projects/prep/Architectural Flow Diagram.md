```mermaid
%%{init:{'flowchart':{'htmlLabels':true,'nodeSpacing':38,'rankSpacing':40,'useMaxWidth':true}}}%%
flowchart TD

A([Client request]) --> B["Gateway auth<br/>mTLS / signed JWT"]

B --> C[Mint service token]

C --> D{"Reserve tokens<br/>idempotency key"}

D -->|insufficient| X1[/402 out of tokens/]

D -->|reserved| E{"Semantic cache<br/>hit?"}

E -->|hit| V["Verifier LLM<br/>cheap model"]

V --> VQ{"Answers<br/>this query?"}

VQ -->|yes| CACHE["Return cached<br/>+ cheap paraphrase"]

CACHE --> M

VQ -->|no · stale| F

E -->|miss| F["Model gateway<br/>policy picks model"]

F --> G["Agent runtime<br/>bounded loop"]

G <-->|retrieve| R[("Shared<br/>vector store")]

G -->|LLM call| P{"Provider call<br/>timeout + breaker"}

P -->|ok| G

P -->|fail| FB["Fallback model<br/>retry backoff"]

FB --> G

G -->|done / max iters| H[Compose response]

H --> C2["Commit tokens<br/>real usage"]

C2 --> W["Cache write<br/>+ persist"]

W --> M["Emit traces<br/>+ cost metrics"]

M --> Z([Response streamed])

classDef term fill:#1F4445,stroke:#123233,color:#fff;
classDef dec fill:#f6ebd9,stroke:#b26a12,color:#5a3708;
classDef led fill:#eae4f6,stroke:#6a4bb0,color:#33235e;
classDef store fill:#dfeceb,stroke:#2F6767,color:#123f3f;
classDef gw fill:#e2e9f3,stroke:#385a86,color:#1f3a5f;
classDef fail fill:#f7e4e2,stroke:#b23b32,color:#6e211b;
classDef ok fill:#e2f2ec,stroke:#157a5b,color:#0d3d2c;

class A,Z term;
class D,E,P,VQ dec;
class C2 led;
class R store;
class B,C,F,V gw;
class X1,FB fail;
class CACHE ok;