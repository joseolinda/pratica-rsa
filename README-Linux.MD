# Prática de Laboratório — Algoritmo RSA

## Tutorial para Linux/Unix/Mac com OpenSSL

Criação e uso de chaves RSA em Linux, Unix e macOS com **OpenSSL** e utilitários shell padrão.

Simula uma ambiente que usa chaves públicas para troca de mensagens que envolvem, de forma **obrigatória**:

- Geração de par de chaves RSA  
- Assinatura digital  
- Verificação da assinatura  
- Criptografia/encriptação  
- Decriptação  

---

## 0. Preparação do ambiente

1. Abra o **Terminal** (macOS: `Cmd + Space` e digite `Terminal`; Linux: geralmente já está disponível).
2. Navegue até a pasta onde você deseja trabalhar, por exemplo:
   ```bash
   cd ~/Desktop
   ```
3. Para ter certeza onde está trabalhando, execute:
   ```bash
   pwd
   ```
4. Clone este repositório:
   ```bash
   git clone https://github.com/joseolinda/pratica-rsa.git
   cd pratica-rsa
   ```
5. Todos os comandos abaixo devem ser executados dentro da pasta do repositório clonado.
6. Tudo que for gerado dentro da pasta raiz do projeto será ignorado pelo Git, exceto os arquivos `.MD` e `.gitignore`.  
   Apenas os arquivos dentro das pastas `public-keys`, `messages` e `responses` serão versionados.
7. É importante seguir os passos na ordem apresentada, renomeando e movendo os arquivos conforme solicitado, para os diretórios corretos.
8. **Pode manter a chave privada (`private.key`) nesta pasta, mas não envie para o repositório**. (!!!!)
9. Ao gerar e mover os arquivos conforme as instruções, faça o commit e push para o repositório remoto. Mas preste atenção para não enviar sua chave privada ou qualquer arquivo na raiz do repositório.
10. Não edite este arquivo `README-Linux.MD`, apenas siga as instruções.

### Verificar se o OpenSSL está instalado

Execute no terminal:
```bash
openssl version
```

Se não estiver instalado:
- **macOS**: `brew install openssl`
- **Ubuntu/Debian**: `sudo apt-get install openssl`
- **Fedora/RHEL**: `sudo dnf install openssl`
- **Arch**: `sudo pacman -S openssl`

---

## 1. Gerar o par de chaves RSA

Execute no terminal:
```bash
# Gerar a chave privada (2048 bits)
openssl genrsa -out private.key 2048

# Extrair a chave pública da chave privada
openssl rsa -in private.key -pubout -out public.key

echo "Chaves geradas: private.key (privada) e public.key (pública)"
```

> **Nota**: `openssl genrsa` gera a chave privada no formato PKCS#1. O comando `openssl rsa -pubout` exporta a chave pública em formato PKIX (SubjectPublicKeyInfo), compatível com a prática do Windows.

### Após a criação das chaves

1. Guardar **private.key** apenas em sua própria pasta (não compartilhar com ninguém). O caminho do arquivo será usado posteriormente para assinar e decriptar mensagens.  
2. Enviar **public.key** para o repositório, renomeando para:

       public-keys/nome-sobrenome.pub

---

## 2. Criar e assinar uma mensagem

Cada aluno deve criar uma mensagem de texto e assiná-la com sua **chave privada**.

### 2.1 Criar a mensagem

No terminal:
```bash
echo "Mensagem destinada à verificação de autenticidade." > mensagem.txt
```

Você deve alterar o texto da mensagem conforme desejar.

### 2.2 Assinar a mensagem com a chave privada

```bash
# Assinar a mensagem com SHA-256 e PKCS#1 padding
openssl dgst -sha256 -sign private.key -out assinatura.bin mensagem.txt

echo "Assinatura criada e salva em assinatura.bin"
```

### Enviar para o diretório "messages" do repositório

Crie uma pasta com o nome do colega destinatário contendo:
- `mensagem.txt`  
- `assinatura.bin`  

Sugestão: salve os arquivos em uma pasta do repositório, por exemplo:

```bash
mkdir -p messages/mensagem-de-jose-para-maria
cp mensagem.txt assinatura.bin messages/mensagem-de-jose-para-maria/
```

Faça o commit e push dessas alterações para o repositório remoto.
Os arquivos a seguir podem ser removidos da raiz do repositório após o envio:
- `mensagem.txt`  
- `assinatura.bin`

**Não enviar NEM apagar `private.key`!**

Aguarde os demais colegas conluirem a etapa 2 antes de prosseguir para a etapa 3.

---

## 3. Verificar a assinatura da mensagem recebida

O aluno B (destinatário) deve verificar a assinatura recebida do colega A, usando a **chave pública** do remetente.

Faça o download dos arquivos enviados pelo colega A e obtenha a chave pública dele no repositório.

### 3.1 Carregar a chave pública e verificar

```bash
# Verificar a assinatura usando a chave pública do colega
openssl dgst -sha256 -verify public-keys/jose-olinda.pub -signature assinatura.bin mensagem.txt
```

Se a assinatura for válida, o resultado será:
```
Verified OK
```

Se a assinatura for inválida, o resultado será:
```
Verification Failure
```

---

### Interpretação do resultado

Responda: o que significa o resultado?  

---

## 4. Criptografar uma resposta para o colega

Agora o aluno B deve enviar uma **resposta confidencial** ao aluno A, criptografada com a **chave pública** do colega A. Não use sua própria chave pública, nem mesmo a sua chave privada.

### 4.1 Criar a resposta

```bash
echo "Esta é uma resposta confidencial para o colega." > resposta.txt
```

Fique à vontade para alterar o texto da resposta. Mensagem ofensiva? Nota zero na prática!

### 4.2 Criptografar com a chave pública do colega

```bash
# Criptografar a resposta com a chave pública do colega
openssl pkeyutl -encrypt -in resposta.txt \
  -inkey public-keys/nome-sobrenome.pub -pubin \
  -out responses/resposta-para-nome-sobrenome.enc

echo "Resposta criptografada salva em resposta-para-nome-sobrenome.enc"
```

> **Nota**: O comando `openssl pkeyutl -encrypt` usa PKCS#1 padding por padrão, compatível com o Windows.

### O que o aluno B deve enviar

- Arquivo `resposta-para-nome-sobrenome.enc` já está na pasta `responses/` do repositório. Faça o commit e push.

---

## 5. Decriptar a resposta recebida (obrigatório)

O aluno A (que recebeu `resposta-para-nome-sobrenome.enc`) deve decriptar a resposta usando **sua chave privada** (`private.key`).

### 5.1 Decriptar com a chave privada

```bash
# Decriptar a resposta com a chave privada
openssl pkeyutl -decrypt -in responses/resposta-para-nome-sobrenome.enc \
  -inkey private.key \
  -out resposta-decriptada.txt

echo "Mensagem decriptada:"
cat resposta-decriptada.txt
```

A mensagem decriptada deve ser exatamente o texto que o colega escreveu em `resposta.txt`. Somente o dono da chave privada correspondente à chave pública usada na criptografia pode decriptar a mensagem.

Recomenda-se que outros colegas tentem decriptar mensagens que não são destinadas a eles (usando suas próprias chaves privadas), para entenderem o conceito de confidencialidade.

---

## Troubleshooting

### Problema: "Cannot open file" ao assinar ou criptografar

Verifique se os caminhos dos arquivos estão corretos:
```bash
ls -la private.key public.key mensagem.txt
```

### Problema: Erro ao carregar chave pública em "Verificar a assinatura"

Verifique se a chave pública está no formato correto:
```bash
head -n 1 public-keys/jose-olinda.pub
```

Deve começar com `-----BEGIN PUBLIC KEY-----`

### Problema: "Verification Failure" mesmo com chave correta

Certifique-se de que:
1. A mensagem não foi modificada após assinatura
2. A chave pública é realmente do remetente
3. Ambos os lados usam SHA-256 (compatível com Windows)

---

## Entregáveis Obrigatórios no Classroom

Cada estudante deverá entregar (por exemplo, via Classroom):

1. Um **relatório curto (até 1 página)** contendo:
   - Diferença entre **assinatura digital** e **criptografia**.  
   - Diferença de criptografar com chave pública e chave privada.
   - Riscos associados ao vazamento da chave privada.  
   - Resumo textual do fluxo realizado.
