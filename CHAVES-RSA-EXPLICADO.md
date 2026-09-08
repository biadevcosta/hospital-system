# `private.pem` e `public.pem` explicado do zero

Sem pressa, com analogia, sem assumir que você já sabe cripto. Se em algum ponto parecer óbvio
demais, pula pro próximo título.

---

## TL;DR (a versão de 20 segundos)

- Quando você faz **login** no `identity-service`, ele te devolve um **token** (um crachá digital).
- O `identity-service` **carimba** esse crachá com a `private.pem` (a chave **secreta**).
- O `scheduling-service` (e depois o `history-service`) **conferem o carimbo** com a `public.pem`
  (a chave **pública**), sem precisar falar com o identity.
- Só quem tem a `private.pem` consegue produzir um carimbo válido. Qualquer um com a `public.pem`
  consegue *verificar*, mas **não consegue falsificar**.

É isso. O resto do documento é o "por quê" e o "como".

---

## 1. Que problema isso resolve?

Imagina que o `scheduling-service` recebe um pedido: *"agendar consulta, sou a Dra. Ana, role
DOCTOR"*. Como ele **sabe** que é verdade? Três opções:

1. **Acreditar no que o pedido diz.** Péssimo — qualquer um manda um JSON dizendo que é ADMIN.
2. **Perguntar pro `identity-service` a cada request:** *"esse token é de verdade?"*. Funciona, mas
   deixa tudo lento e acopla os serviços (se o identity cai, o resto para).
3. **O identity assina o token de um jeito impossível de imitar, e cada serviço verifica sozinho.**
   É essa. Chama **JWT assinado com RS256**. As duas chaves `.pem` são o coração dela.

---

## 2. O que é uma "chave" em criptografia

Não é um arquivo com senha. É um **número gigante** (centenas de dígitos) com propriedades
matemáticas especiais. O arquivo `.pem` é só esse número escrito em texto.

Existem dois estilos de usar chaves:

| Estilo | Como é | Analogia |
|---|---|---|
| **Simétrico** | **uma** chave só, a mesma pra trancar e destrancar | uma fechadura comum: quem tranca e quem abre usam a mesma chave |
| **Assimétrico** | um **par**: uma privada + uma pública, que "combinam" entre si | cadeado e a única chave que o abre |

Nosso caso é **assimétrico** (RSA). O par tem uma sacada genial:

> O que a chave **privada** faz, só a chave **pública** correspondente consegue *conferir* —
> e vice-versa. E tendo só a pública, **não dá** pra descobrir a privada.

---

## 3. A analogia do carimbo de cartório

- O `identity-service` é o **cartório**. Ele tem **um carimbo único** (`private.pem`). Só ele tem.
- Todo mundo tem uma **foto em alta resolução do carimbo** (`public.pem`). Com a foto você
  **reconhece** se um documento foi carimbado por aquele cartório...
- ...mas com a foto você **não consegue esculpir** um carimbo igual. Falsificar é
  computacionalmente inviável (é aí que entra o "2048 bits" — o tamanho do número que torna isso
  impossível na prática).

Quando você faz login, o cartório (identity) **carimba seu crachá**. Quando você chega no
`scheduling-service` com esse crachá, ele **olha a foto do carimbo** (a `public.pem` que ele tem
guardada) e confirma: *"sim, esse carimbo é do cartório, o crachá é legítimo, e diz que ela é
DOCTOR"*. Não precisou ligar pro cartório.

Outra forma de pensar: **assinar** = usar a privada. **Verificar a assinatura** = usar a pública.

---

## 4. O que é RSA, "2048 bits" e PEM

- **RSA** = o algoritmo de criptografia assimétrica que a gente usa (nome dos 3 criadores: Rivest,
  Shamir, Adleman, anos 70). Ele define como gerar o par e como assinar/verificar.
- **2048 bits** = o "tamanho" da chave (o número tem ~617 dígitos decimais). Quanto maior, mais
  difícil de quebrar. 2048 é o mínimo recomendado hoje; 1024 já é considerado fraco.
- **RS256** = "**RS**A signature + SHA-**256**". SHA-256 é a função que resume o conteúdo do token
  num "digest" antes de assinar. É o `alg` que aparece no cabeçalho do JWT.
- **PEM** = só um **formato de arquivo de texto**. Pega o número binário da chave, codifica em
  Base64, e embrulha com cabeçalho/rodapé:
  ```
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSk...   (várias linhas de Base64)
  -----END PRIVATE KEY-----
  ```
  - `private.pem` → `-----BEGIN PRIVATE KEY-----` (formato PKCS#8)
  - `public.pem`  → `-----BEGIN PUBLIC KEY-----` (formato X.509 SubjectPublicKeyInfo)
  - Existe também `-----BEGIN RSA PRIVATE KEY-----` (formato antigo, PKCS#1). O nosso é o **sem**
    "RSA" no meio.

---

## 5. O que é um JWT e onde a chave entra

Um **JWT** (JSON Web Token) é aquele textão que o login devolve:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9 . eyJpc3MiOiJob3NwaXRhbC1pZGVudGl0eS...  . oMKbjy_P1soIxOQMn...
└──────────── header ────────────────┘ └──────────── payload ──────────────┘ └──── signature ────┘
```

São **3 pedaços** separados por ponto, cada um em Base64:

1. **Header** — diz o algoritmo: `{"typ":"JWT","alg":"RS256"}`
2. **Payload** — os dados (as *claims*). No nosso identity:
   - `iss`: `hospital-identity` (quem emitiu)
   - `aud`: `hospital-services` (pra quem serve)
   - `sub`: o id do usuário
   - `role`: `DOCTOR` / `NURSE` / `PATIENT` / `ADMIN`
   - `iat` / `exp`: emitido em / expira em (**1 hora** depois)
   - `patientId`: só quando o usuário é PATIENT
3. **Signature** — o resultado de assinar `header.payload` com a **`private.pem`**.

O header e o payload **não são secretos** — qualquer um faz Base64-decode e lê (cola em
<https://jwt.io>). O que impede fraude é a **assinatura**: se você mudar um único caractere do
payload (ex.: trocar `"role":"PATIENT"` por `"role":"ADMIN"`), a assinatura não bate mais, e a
`public.pem` rejeita na hora.

> Então o JWT não **esconde** os dados, ele **garante que ninguém mexeu neles** e que quem emitiu
> foi mesmo o identity.

---

## 6. No NOSSO sistema: quem assina, quem verifica

```
┌────────────────────┐   POST /auth/login (email + senha)
│  você / Insomnia   │ ─────────────────────────────────────────►┐
└────────────────────┘                                           │
                                                                 ▼
                                              ┌─────────────────────────────────────┐
                                              │        identity-service             │
                                              │                                     │
                                              │  1. confere email + senha (Argon2)  │
                                              │  2. monta as claims (sub, role...)  │
                                              │  3. ASSINA com  private.pem  (RS256)│
                                              │     → NimbusJwtTokenIssuer          │
                                              └─────────────────────────────────────┘
                                                                 │  devolve o JWT
   ◄─────────────────────────────────────────────────────────────┘
   │
   │  agora você manda o JWT no header:  Authorization: Bearer eyJ...
   │
   ├──────────────────────────────►┌─────────────────────────────────────┐
   │                               │        scheduling-service           │
   │                               │                                     │
   │                               │  VERIFICA a assinatura com          │
   │                               │  public.pem  (NimbusJwtDecoder)     │
   │                               │  - assinatura bate?                 │
   │                               │  - iss == hospital-identity?        │
   │                               │  - exp ainda no futuro?             │
   │                               │  → se ok, lê role e libera/nega     │
   │                               └─────────────────────────────────────┘
   │
   └──────────────────────────────►  history-service  (mesma verificação, mesma public.pem)
```

- **Só o `identity-service` tem a `private.pem`.** É o único que **cria** tokens.
- **Todos os serviços têm a `public.pem`.** Cada um **verifica** por conta própria, offline.
- É por isso que não existe "gateway" nem sessão compartilhada: o token se auto-comprova.

---

## 7. O caminho do arquivo dentro do código (por onde isso passa)

### No identity-service

1. **`src/main/resources/application.yaml`** aponta os caminhos:
   ```yaml
   security:
     jwt:
       private-key: classpath:private.pem   # "classpath:" = procure dentro dos resources do app
       public-key:  classpath:public.pem
       issuer: hospital-identity
       audience: hospital-services
       access-ttl-seconds: 3600             # 1 hora
   ```
2. **`infrastructure/config/RsaKeys.java`** lê os `.pem`, tira o cabeçalho `-----BEGIN...-----`,
   faz Base64-decode e transforma em objetos Java `RSAPrivateKey` / `RSAPublicKey`
   (via `KeyFactory("RSA")` + `PKCS8EncodedKeySpec` / `X509EncodedKeySpec`).
3. **`infrastructure/security/NimbusJwtTokenIssuer.java`** recebe a `RSAPrivateKey` e usa a lib
   **Nimbus** (`RSASSASigner`, `SignedJWT`, header `alg=RS256`) pra **assinar** o token em
   `POST /auth/login` e `POST /auth/refresh`.
4. **`infrastructure/security/SecurityConfig.java`** monta um `NimbusJwtDecoder.withPublicKey(...)`
   com a `RSAPublicKey` pra **verificar** o token na própria rota protegida do identity
   (`POST /users`, que é só de ADMIN).

### No scheduling-service (e history-service)

- **`application.yaml`**: `security.jwt.public-key: classpath:public.pem` (**só a pública** — esses
  serviços nunca assinam nada).
- **`infrastructure/security/SecurityConfig.java`**: lê a `public.pem`, monta o `NimbusJwtDecoder`,
  e cada `POST /graphql` passa por essa verificação antes de chegar no código.

### No Docker

O `.pem` está em `src/main/resources/`. Quando a imagem é buildada (`mvn package`), ele entra
**dentro do `.jar`**. O `Dockerfile` copia o jar pronto. Ou seja: **trocar a chave exige
`docker compose up --build`** (rebuild), não adianta só reiniciar o container. (A gente bateu
nesse detalhe hoje.)

---

## 8. Como VOCÊ gerou as chaves

Dois comandos `openssl` (do `identity-service/README.md`), rodados na raiz do projeto:

```bash
# 1) gera a chave PRIVADA (RSA, 2048 bits, formato PKCS#8) no arquivo keys/private.pem
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out keys/private.pem

# 2) DERIVA a chave pública a partir da privada e salva em keys/public.pem
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

Traduzindo pedaço por pedaço:

| Pedaço | O que faz |
|---|---|
| `openssl genpkey` | subcomando "gerar uma chave nova" |
| `-algorithm RSA` | do tipo RSA |
| `-pkeyopt rsa_keygen_bits:2048` | com 2048 bits de tamanho |
| `-out keys/private.pem` | escreve o resultado nesse arquivo |
| `openssl rsa -in keys/private.pem` | lê a chave privada... |
| `-pubout` | ...e cospe **só a parte pública** dela |
| `-out keys/public.pem` | nesse arquivo |

O ponto-chave do passo 2: a **pública é calculada a partir da privada**. Elas nascem juntas, do
mesmo "número mãe". É por isso que uma verifica o que a outra assina — e por isso que uma
`public.pem` de um par **não** funciona com a `private.pem` de outro.

### Onde a gente copiou cada uma (hoje, ao gerar o par novo)

```
keys/private.pem                                  ← fonte da verdade (fica só na sua máquina)
keys/public.pem                                   ← fonte da verdade

identity-service/src/main/resources/private.pem   ← ASSINA os tokens        (git-ignored!)
identity-service/src/main/resources/public.pem    ← verifica a rota ADMIN   (commitado)
scheduling-service/src/main/resources/public.pem  ← verifica os tokens      (commitado)
history-service/src/main/resources/public.pem     ← verifica os tokens      (commitado)
```

O par que está valendo agora tem "impressão digital" (`modulus`) `23fe4ef9...` nos quatro lugares.

---

## 9. Por que `private.pem` NÃO vai pro Git e `public.pem` VAI

- **`private.pem` é o carimbo do cartório.** Se vaza, qualquer pessoa passa a emitir tokens
  válidos dizendo que é ADMIN, DOCTOR, qualquer coisa. É **a** credencial mais sensível do sistema.
  Por isso está no `.gitignore`:
  ```
  # identity-service/.gitignore, linha 60
  src/main/resources/private.pem
  ```
  Num clone novo do repositório **ela não existe** — você regenera com os 2 comandos acima.

- **`public.pem` é a foto do carimbo.** Não dá pra fazer nada malicioso com ela — só *verificar*.
  Pode commitar tranquilo e distribuir pros outros serviços. Precisa estar no repo justamente pra
  os serviços conseguirem validar sem configuração manual.

> Regra geral de programação: **segredo (privado) nunca entra no controle de versão.** Senha, token
> de API, chave privada — tudo isso fica fora do Git (via `.gitignore`, variável de ambiente, ou um
> cofre de segredos).

---

## 10. O par TEM que casar (o bug que a gente viveu)

Mais cedo, a `private.pem` local e a `public.pem` commitada eram de **pares diferentes** (alguém
regerou uma e esqueceu a outra). Resultado:

- login funcionava (o identity assinava com a privada dele),
- mas **todo** request com `Authorization: Bearer ...` dava **401**, porque a `public.pem` do outro
  par não reconhecia aquela assinatura.

Como conferir se um par casa (as duas linhas têm que dar **o mesmo hash**):

```bash
openssl rsa -in  private.pem -noout -modulus | openssl md5     # ex: 23fe4ef9...
openssl rsa -pubin -in public.pem -noout -modulus | openssl md5 # tem que ser IGUAL
```

O `modulus` é o número gigante que os dois compartilham. Moduli diferentes = pares diferentes =
não funciona.

---

## 11. As outras chaves: `src/test/resources/test-private.pem` / `test-public.pem`

Essas são **de propósito** commitadas. Servem só pros testes automatizados (`./mvnw verify`):
o teste assina tokens falsos com a `test-private.pem` e o app valida com a `test-public.pem`. Como
são um par isolado e sem valor em produção, tudo bem versionar. **Nunca** use essas em produção.

---

## 12. Se a `private.pem` vazar (ou você quiser trocar)

1. Gere um par novo (os 2 comandos do passo 8).
2. Copie a `private.pem` nova pra `identity-service/src/main/resources/`.
3. Copie a `public.pem` nova pros **três** `src/main/resources/` (identity, scheduling, history).
4. `docker compose up --build` em cada serviço (a chave está dentro do jar).
5. **Todos os tokens antigos param de valer na hora** — os usuários fazem login de novo. Isso é
   uma vantagem: trocar a chave é a forma de "revogar tudo" de uma vez.

---

## 13. Comandos úteis pra inspecionar

```bash
# ver os detalhes da chave pública (expoente, modulus, tamanho)
openssl rsa -pubin -in public.pem -text -noout

# ver os detalhes da chave privada
openssl rsa -in private.pem -text -noout

# confirmar que um par casa (mesmo hash nas duas linhas)
openssl rsa -in private.pem -noout -modulus | openssl md5
openssl rsa -pubin -in public.pem -noout -modulus | openssl md5

# decodificar o payload de um JWT sem verificar (só pra ler as claims)
echo 'eyJpc3MiOiJob3NwaXRhbC1pZGVudGl0eS...' | base64 -d      # o pedaço do meio do token
# ou cole o token inteiro em https://jwt.io
```

---

## 14. Glossário de bolso

| Termo | Em uma frase |
|---|---|
| **Chave privada** | número secreto que **assina**; só o identity tem |
| **Chave pública** | número que **verifica** a assinatura; todo serviço tem |
| **Par de chaves** | privada + pública que nasceram juntas e se correspondem |
| **RSA** | o algoritmo de criptografia assimétrica usado |
| **RS256** | assinatura RSA + hash SHA-256 (o `alg` do JWT) |
| **PEM** | formato de arquivo texto (Base64 + `-----BEGIN...-----`) que guarda a chave |
| **JWT** | crachá digital: `header.payload.signature`, tudo em Base64 |
| **Claim** | cada campo dentro do payload do JWT (`sub`, `role`, `exp`...) |
| **Assinar** | gerar a `signature` a partir de `header.payload` + chave privada |
| **Verificar** | checar, com a chave pública, se a `signature` corresponde ao conteúdo |
| **modulus** | o número gigante comum ao par; serve de "impressão digital" pra comparar |
| **`classpath:`** | "procure este arquivo dentro dos resources do app / do jar" |
| **git-ignored** | arquivo que o Git é instruído a nunca versionar (fica só local) |
