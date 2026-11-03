# Prática de laboratório — RSA com OpenSSL

Ao final da prática, **cada aluno** terá criado suas chaves pública e privada, enviado uma mensagem assinada, verificado uma mensagem recebida, enviado um texto criptografado e descriptografado a resposta do colega.

**Windows:** siga o [roteiro para PowerShell](README-Powershell.MD), que inclui o download do OpenSSL Light pelo navegador e a alternativa com `wget.exe`. Os comandos abaixo são para **Linux e macOS, em Bash ou Zsh**. Alunos em sistemas diferentes podem formar uma dupla.

## Como a dupla vai trabalhar

Escolham identificadores sem espaços ou acentos, como `alice-silva` e `bruno-lima`. Nos exemplos, usamos `alice` e `bruno`. Cada aluno usa seu próprio computador, sua própria chave privada e executa **todas as etapas**, enviando e recebendo arquivos. Usem apenas mensagens fictícias nesta atividade.

| Operação | Chave usada | O que aprendemos |
| --- | --- | --- |
| Assinar uma mensagem | Privada do remetente | Produzir uma assinatura vinculada ao conteúdo |
| Verificar a assinatura | Pública do remetente | Conferir integridade e autenticidade em relação à chave confirmada |
| Criptografar um texto | Pública do destinatário | Proteger a confidencialidade |
| Descriptografar o texto | Privada do destinatário | Recuperar o conteúdo recebido |

Assinatura **não esconde** o texto. Criptografia, sozinha, **não comprova quem enviou** o texto: qualquer pessoa com a chave pública do destinatário pode criptografar para ele. Assinar é uma operação própria; não a descreva como “criptografar com a chave privada”.

## 0. Preparar o ambiente e as pastas

### OpenSSL e Git

Verifique os programas:

```bash
openssl version
git --version
```

Use **OpenSSL 3.x**. Se necessário, instale pelo gerenciador do sistema:

| Sistema | Instalação |
| --- | --- |
| Ubuntu/Debian | `sudo apt update` e depois `sudo apt install openssl git` |
| Fedora | `sudo dnf install openssl git` |
| macOS com Homebrew | `brew install openssl@3 git` |

No macOS, se `openssl version` ainda mostrar LibreSSL ou uma versão antiga, ajuste o terminal atual e confira novamente:

```bash
export PATH="$(brew --prefix openssl@3)/bin:$PATH"
openssl version
```

Clone o repositório indicado pelo professor. O endereço abaixo é o projeto base; use o repositório da turma se houver outro. Para o fluxo com `push`, é necessário ter acesso de escrita ao repositório compartilhado.

```bash
git clone https://github.com/joseolinda/pratica-rsa.git
cd pratica-rsa
```

**Depois deste `cd`, execute todos os comandos na raiz `pratica-rsa`.** Não entre nas pastas de mensagens. Se o terminal for fechado, volte à raiz e defina novamente as variáveis abaixo.

### Identificar remetente e destinatário

Alice usa o bloco a seguir. Bruno troca **somente as duas primeiras linhas** por `eu='bruno'` e `colega='alice'`. Substituam os exemplos pelos identificadores reais da dupla.

```bash
eu='alice'
colega='bruno'
local="local/$eu"
enviada="messages/de-$eu-para-$colega"
recebida="messages/de-$colega-para-$eu"
respostaEnviada="responses/de-$eu-para-$colega"
respostaRecebida="responses/de-$colega-para-$eu"
mkdir -p "$local" public-keys "$enviada" "$respostaEnviada"
chmod 700 "$local"
```

A organização será esta, no exemplo de Alice:

```text
pratica-rsa/
├── local/alice/                       # Somente neste computador; ignorado pelo Git
│   ├── private.key                   # Chave privada de Alice
│   ├── resposta.txt                  # Texto confidencial antes da criptografia
│   └── resposta-recebida.txt          # Texto recebido após a descriptografia
├── public-keys/
│   ├── alice.pub                     # Chave pública de Alice
│   └── bruno.pub                     # Chave pública de Bruno
├── messages/
│   ├── de-alice-para-bruno/           # Mensagem enviada por Alice
│   │   ├── mensagem.txt
│   │   └── assinatura.bin
│   └── de-bruno-para-alice/           # Mensagem recebida de Bruno
│       ├── mensagem.txt
│       └── assinatura.bin
└── responses/
    ├── de-alice-para-bruno/resposta.enc
    └── de-bruno-para-alice/resposta.enc
```

O `.gitignore` permite as chaves `.pub`, as mensagens assinadas e as respostas `.enc` nos caminhos indicados, além da documentação. A pasta `local/` e as chaves privadas ficam fora do versionamento. O `.gitattributes` preserva os bytes das mensagens assinadas durante a troca entre Windows e Linux/macOS. **Nunca publique sua chave privada nem o texto confidencial.**

Se já fez uma versão anterior da prática, mantenha sua chave antiga guardada e comece em um identificador novo ou em outro clone. Os novos comandos usam `local/<aluno>/private.key`, não a antiga `private.key` da raiz. Os arquivos antigos `.enc` podem ter usado outro padding; refaçam a troca com o novo roteiro.

## 1. Criar e compartilhar as chaves

Gere a chave privada RSA de 2048 bits e extraia a chave pública. O teste inicial evita substituir uma chave existente. Se os dois arquivos já existem e formam seu próprio par, preserve-os e pule a geração. Se apenas um existe ou se você não tem certeza de quem é a chave pública, resolva com o professor antes de continuar; uma nova identidade exige outro identificador livre e a atualização das variáveis.

```bash
if [ -e "$local/private.key" ] || [ -e "public-keys/$eu.pub" ]; then
  printf '%s\n' 'Ja existe uma chave neste caminho. Nao sobrescreva o par existente.'
else
  (umask 077; openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out "$local/private.key") &&
  openssl pkey -in "$local/private.key" -pubout -out "public-keys/$eu.pub"
fi
```

Confira a estrutura da chave privada sem exibir seu conteúdo:

```bash
openssl pkey -in "$local/private.key" -check -noout
```

O resultado deve conter `Key is valid`. Para simplificar o laboratório, a chave é salva sem senha e protegida pela pasta local; mantenha o computador e esse arquivo sob seu controle. A chave privada é usada para **assinar e descriptografar**. Compartilhe somente `public-keys/<seu-identificador>.pub`.

### Publicar a chave pública

```bash
git add "public-keys/$eu.pub"
git diff --cached --name-only
```

Confira a lista antes de continuar: nesta etapa, apenas sua chave pública deve estar selecionada. Se houver algo indevido, retire-o da seleção com `git restore --staged caminho-do-arquivo`.

```bash
git commit -m "Publica chave de $eu"
git pull --rebase
git push
```

**Ponto de encontro:** ambos publicam suas chaves antes de avançar. Atualize seu clone:

```bash
git pull --ff-only
```

### Confirmar a chave do colega

Calcule a impressão digital SHA-256 da sua chave e da chave recebida:

```bash
openssl pkey -pubin -in "public-keys/$eu.pub" -outform DER -out "$local/minha-publica.der"
openssl dgst -sha256 "$local/minha-publica.der"
openssl pkey -pubin -in "public-keys/$colega.pub" -outform DER -out "$local/publica-colega.der"
openssl dgst -sha256 "$local/publica-colega.der"
```

Conversem pessoalmente: o hash que você calculou para a chave do colega deve ser igual ao hash que ele calculou para a própria chave. Comparem a sequência hexadecimal; os nomes dos arquivos serão diferentes. Uma assinatura válida só pode ser atribuída ao colega depois de confirmar que a chave pública realmente é dele.

## 2. Criar, assinar e enviar uma mensagem

Crie uma mensagem pública e assine com **sua chave privada**:

```bash
printf 'Ola, %s! Esta mensagem foi enviada por %s.\n' "$colega" "$eu" > "$enviada/mensagem.txt"
openssl dgst -sha256 -sign "$local/private.key" -out "$enviada/assinatura.bin" "$enviada/mensagem.txt"
```

Você pode personalizar o texto **antes** de assinar. A assinatura usa RSA com SHA-256; `assinatura.bin` é um arquivo binário separado. A mensagem continua legível. Depois de assinar, não edite nem salve novamente `mensagem.txt`, pois a assinatura depende dos bytes exatos do arquivo.

Envie **os dois arquivos**:

```bash
git add "$enviada/mensagem.txt" "$enviada/assinatura.bin"
git diff --cached --name-only
```

Confira que apenas esses dois arquivos foram selecionados e publique:

```bash
git commit -m "Envia mensagem assinada de $eu para $colega"
git pull --rebase
git push
```

**Ponto de encontro:** ambos enviam suas mensagens antes de verificar a mensagem recebida.

## 3. Conferir se a mensagem recebida é original

Atualize os arquivos e verifique com a **chave pública do remetente**:

```bash
git pull --ff-only
openssl dgst -sha256 -verify "public-keys/$colega.pub" -signature "$recebida/assinatura.bin" "$recebida/mensagem.txt"
echo "Codigo de saida: $?"
```

Resultado esperado: `Verified OK` e código `0`. Isso confirma que o conteúdo corresponde à assinatura para a chave pública usada. Com a chave confirmada na etapa 1, você também relaciona a assinatura ao colega. Não significa que o conteúdo da mensagem seja verdadeiro.

### Experimento 1: alterar uma cópia

Mantenha a mensagem recebida intacta. Altere apenas uma cópia local e tente verificar com a assinatura original:

```bash
cp "$recebida/mensagem.txt" "$local/mensagem-alterada.txt"
printf '\nTexto acrescentado depois da assinatura.\n' >> "$local/mensagem-alterada.txt"
openssl dgst -sha256 -verify "public-keys/$colega.pub" -signature "$recebida/assinatura.bin" "$local/mensagem-alterada.txt"
echo "Codigo de saida: $?"
```

Resultado esperado: `Verification failure` ou `Verification Failure`, com código diferente de `0`. Esse é um **erro esperado**: a assinatura detectou a alteração.

### Experimento 2: usar a chave pública errada

Use sua chave pública para tentar verificar a mensagem assinada pelo colega:

```bash
openssl dgst -sha256 -verify "public-keys/$eu.pub" -signature "$recebida/assinatura.bin" "$recebida/mensagem.txt"
echo "Codigo de saida: $?"
```

Também deve falhar. Registre os três resultados: original com chave correta, cópia alterada e original com chave errada.

## 4. Criar e enviar um texto criptografado

Agora cada aluno responde ao colega com um texto confidencial. Salve o texto somente em `local/`:

```bash
printf 'Resposta confidencial de %s para %s: encontro na biblioteca.\n' "$eu" "$colega" > "$local/resposta.txt"
wc -c < "$local/resposta.txt"
```

O número exibido é o tamanho em **bytes**, incluindo acentos e quebras de linha. Nesta prática, RSA de 2048 bits com **OAEP/SHA-256** aceita no máximo **190 bytes** (`256 - 2 × 32 - 2`). Se passar disso, encurte o texto antes de continuar. Para arquivos maiores, usa-se criptografia híbrida, assunto para outra atividade.

Criptografe com a **chave pública do destinatário**, usando os mesmos parâmetros OAEP nos dois sistemas:

```bash
openssl pkeyutl -encrypt -pubin -inkey "public-keys/$colega.pub" -in "$local/resposta.txt" -out "$respostaEnviada/resposta.enc" -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
```

O comando deve terminar sem erro. O arquivo `resposta.enc` tem 256 bytes e não é texto legível. Ele é criptografado, mas não foi assinado nesta etapa; a criptografia sozinha não autentica seu remetente. Publique somente o arquivo criptografado:

```bash
git add "$respostaEnviada/resposta.enc"
git diff --cached --name-only
```

Confira que apenas `resposta.enc` foi selecionado:

```bash
git commit -m "Envia resposta criptografada de $eu para $colega"
git pull --rebase
git push
```

**Ponto de encontro:** ambos enviam suas respostas criptografadas. Não envie `local/resposta.txt` nem sua chave privada.

## 5. Descriptografar o texto recebido

Atualize os arquivos e use **sua chave privada**, correspondente à chave pública usada pelo colega:

```bash
git pull --ff-only
openssl pkeyutl -decrypt -inkey "$local/private.key" -in "$respostaRecebida/resposta.enc" -out "$local/resposta-recebida.txt" -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
echo "Codigo de saida: $?"
cat "$local/resposta-recebida.txt"
```

Resultado esperado: código `0` e o texto que seu colega escreveu. **Se houver erro, não considere o arquivo de saída uma mensagem válida.** Compare o texto recuperado com o original na tela do colega, sem publicar os textos confidenciais.

### Experimento 3: tentar descriptografar com a chave errada

Tente abrir a resposta que **você enviou** ao colega. Ela foi criptografada com a chave pública dele, então sua chave privada não deve funcionar:

```bash
openssl pkeyutl -decrypt -inkey "$local/private.key" -in "$respostaEnviada/resposta.enc" -out "$local/teste-chave-errada.txt" -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
echo "Codigo de saida: $?"
```

Resultado esperado: erro de descriptografia OAEP e código diferente de `0`. O arquivo de saída dessa tentativa não deve ser usado. Apenas quem possui a chave privada correspondente à chave pública do destinatário consegue recuperar corretamente esse texto.

## 6. Entrega individual

Cada estudante deve entregar um relatório de até uma página no Classroom, com seu identificador, o identificador do colega e:

- Caminhos da sua chave **pública**, da mensagem assinada enviada e da resposta criptografada enviada no repositório.
- Evidências dos três testes de assinatura: original válido, cópia alterada inválida e chave pública errada inválida.
- Evidência de descriptografia bem-sucedida e da falha com chave privada errada. Use o texto fictício da prática.
- Explicação de qual chave usou em cada operação e da diferença entre autenticidade, integridade e confidencialidade.
- Respostas: por que confirmar a chave pública? O que acontece se a chave privada vazar? Criptografar para alguém prova quem enviou?

**Critério de conclusão:** cada aluno precisa realizar geração de chaves, assinatura, envio, verificação, criptografia e descriptografia. Dividir as operações entre os integrantes não completa a prática individual. A chave privada não faz parte de nenhuma entrega.

## Problemas frequentes

| Sintoma | Como resolver |
| --- | --- |
| `openssl` não encontrado ou LibreSSL no macOS | Instale OpenSSL 3 e confira o `PATH` da etapa 0. No Windows, siga o [roteiro PowerShell](README-Powershell.MD). |
| Arquivo não encontrado | Execute `pwd` e `ls`; volte à raiz do repositório e confira identificadores e variáveis. Aguarde o colega publicar e execute `git pull --ff-only`. |
| Variáveis vazias após reabrir o terminal | Repita o bloco de identificação da etapa 0; não gere outro par de chaves. |
| `Verified OK` não aparece no teste original | Confira a chave do remetente, a assinatura correspondente e se o arquivo foi editado ou convertido após a assinatura. Peça ao remetente para assinar e enviar novamente o arquivo definitivo. |
| `data too large for key size` | A resposta excedeu 190 bytes. Encurte o texto; conte bytes com `wc -c`, não caracteres. |
| Erro OAEP ao descriptografar | Confira o destinatário, sua chave privada original e os três parâmetros `rsa_padding_mode`, `rsa_oaep_md` e `rsa_mgf1_md`. Não misture respostas antigas com o novo fluxo. |
| `push` rejeitado porque o colega enviou antes | Depois de seu commit, execute `git pull --rebase` e tente `git push` novamente. Em caso de conflito, peça ajuda ao professor antes de sobrescrever arquivos. |
| Git pede nome e e-mail | Configure sua identidade com `git config user.name "Seu Nome"` e `git config user.email "seu-email"` neste clone. |
| Sem acesso de escrita ao repositório | Use o repositório da turma indicado pelo professor ou troque os mesmos arquivos por uma pasta compartilhada, preservando nomes e bytes. Não compartilhe a pasta `local/`. |
| Chave privada perdida | Ela não pode ser recuperada da chave pública. Gere um novo par em outra pasta, confirme a nova chave com o colega e refaçam o envio destinado ao novo par. |

## Referências técnicas

- Geração e exportação das chaves: [OpenSSL `genpkey`](https://docs.openssl.org/3.5/man1/openssl-genpkey/) e [`pkey`](https://docs.openssl.org/3.5/man1/openssl-pkey/).
- Assinatura e verificação: [OpenSSL `dgst`](https://docs.openssl.org/3.5/man1/openssl-dgst/).
- Criptografia e descriptografia com OAEP: [OpenSSL `pkeyutl`](https://docs.openssl.org/3.5/man1/openssl-pkeyutl/).
- Preservação dos bytes durante a troca de arquivos: [Git `gitattributes`](https://git-scm.com/docs/gitattributes).
