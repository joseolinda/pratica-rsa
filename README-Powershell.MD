# Prática de Laboratório — Algoritmo RSA

## Tutorial para Windows com PowerShell

Criação e uso de chaves RSA no Windows com **PowerShell**, utilizando as bibliotecas nativas do .NET.

Simula uma ambiente que usa chaves públicas para troca de mensagens que envolvem, de forma **obrigatória**:

- Geração de par de chaves RSA  
- Assinatura digital  
- Verificação da assinatura  
- Criptografia/encriptação  
- Decriptação  

---

## 0. Preparação do ambiente

1. Abra o **PowerShell** (pressione `Win + R`, digite `powershell` e pressione `Enter`).  
2. Navegue até a pasta onde você deseja trabalhar, por exemplo:
   ```bash
   cd C:\Users\SeuUsuario\Desktop
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
10. Não edite este arquivo `README.MD`, apenas siga as instruções.

---

## 1. Gerar o par de chaves RSA

Abra o **PowerShell** e execute:
```bash

$rsa = [System.Security.Cryptography.RSA]::Create(2048)

[IO.File]::WriteAllBytes("private.key", $rsa.ExportRSAPrivateKey())
[IO.File]::WriteAllBytes("public.key",  $rsa.ExportRsaPublicKey())

Write-Host "Chaves geradas: private.key (privada) e public.key (pública)"

```

> Caso `ExportRsaPublicKey()` não funcione em alguma máquina, troque por:
```bash
[IO.File]::WriteAllBytes("public.key", $rsa.ExportSubjectPublicKeyInfo())
```

### Após a criação das chaves

1. Guardar **private.key** apenas em sua própria pasta (não compartilhar com ninguém). O caminho do arquivo será usado posteriormente para assinar e decriptar mensagens.  
2. Enviar **public.key** para o repositório, renomeando para:

       public-keys/nome-sobrenome.pub

---

## 2. Criar e assinar uma mensagem

Cada aluno deve criar uma mensagem de texto e assiná-la com sua **chave privada**.

### 2.1 Criar a mensagem

No PowerShell:
```bash
"Mensagem destinada à verificação de autenticidade." | Out-File -Encoding utf8 mensagem.txt
```

Você deve alterar o texto da mensagem conforme desejar.

### 2.2 Assinar a mensagem com a chave privada

```bash
$rsa = [System.Security.Cryptography.RSA]::Create()
$offset = 0
$privateBytes = [IO.File]::ReadAllBytes("private.key")
$rsa.ImportRSAPrivateKey($privateBytes, [ref]$offset)

$texto  = Get-Content mensagem.txt -Raw
$data   = [System.Text.Encoding]::UTF8.GetBytes($texto)

$signature = $rsa.SignData(
    $data,
    [System.Security.Cryptography.HashAlgorithmName]::SHA256,
    [System.Security.Cryptography.RSASignaturePadding]::Pkcs1
)

[IO.File]::WriteAllBytes("assinatura.bin", $signature)

Write-Host "Assinatura criada e salva em assinatura.bin"
```

### Enviar para o diretório "messages" do repositório

Crie um arquivo ZIP ou uma pasta com o nome do colega destinatário contendo:
- `mensagem.txt`  
- `assinatura.bin`  

Sugestão: salve os arquivos em uma pasta do repositório, por exemplo:

```bash
messages/mensagem-de-jose-para-maria/
    mensagem.txt
    assinatura.bin
```

---

## 3. Verificar a assinatura da mensagem recebida

O aluno B (destinatário) deve verificar a assinatura recebida do colega A, usando a **chave pública** do remetente.

Faça o download dos arquivos enviados pelo colega A e obtenha a chave pública dele no repositório.


### 3.1 Carregar a chave pública e verificar

```bash
$rsa = [System.Security.Cryptography.RSA]::Create()

$offset = 0
$publicBytes = [IO.File]::ReadAllBytes("public-keys/jose-olinda.pub")
$rsa.ImportSubjectPublicKeyInfo($publicBytes, [ref]$offset)

$texto      = Get-Content mensagem.txt -Raw
$data       = [System.Text.Encoding]::UTF8.GetBytes($texto)
$signature  = [IO.File]::ReadAllBytes("assinatura.bin")

$ok = $rsa.VerifyData(
    $data,
    $signature,
    [System.Security.Cryptography.HashAlgorithmName]::SHA256,
    [System.Security.Cryptography.RSASignaturePadding]::Pkcs1
)

if ($ok) {
    Write-Host "✔ Assinatura válida — mensagem realmente enviada pelo colega."
}
else {
    Write-Host "✘ Assinatura inválida — mensagem adulterada ou chave incorreta."
}
```
---

### Interpretação do resultado

Responda: o que significa o resultado?  

---

## 4. Criptografar uma resposta para o colega

Agora o aluno B deve enviar uma **resposta confidencial** ao aluno A, criptografada com a **chave pública** do colega A. Não use sua própria chave pública, nem mesmo a sua chave privada.

### 4.1 Criar a resposta

```bash
"Esta é uma resposta confidencial para o colega." |
        Out-File -Encoding utf8 resposta.txt
```

Fique à vontade para alterar o texto da resposta. Mensagem ofensiva? Nota zero na prática!

### 4.2 Criptografar com a chave pública do colega

```bash
$rsa = [System.Security.Cryptography.RSA]::Create()

$offset = 0
$publicBytes = [IO.File]::ReadAllBytes("public-keys/nome-sobrenome.pub")
$rsa.ImportSubjectPublicKeyInfo($publicBytes, [ref]$offset)

$textoResposta = Get-Content resposta.txt -Raw
$dataResposta  = [System.Text.Encoding]::UTF8.GetBytes($textoResposta)

$encrypted = $rsa.Encrypt(
    $dataResposta,
    [System.Security.Cryptography.RSAEncryptionPadding]::Pkcs1
)

[IO.File]::WriteAllBytes("resposta-para-<nome-sobrenome>.enc", $encrypted)

Write-Host "Resposta criptografada salva em resposta.enc"
```

### O que o aluno B deve enviar

- Arquivo `resposta-para-<nome-sobrenome>.enc` para o diretório `responses` no repositório.
- Faça o commit e push.

---

## 5. Decriptar a resposta recebida (obrigatório)

O aluno A (que recebeu `resposta-para-<nome-sobrenome>.enc`) deve decriptar a resposta usando **sua chave privada** (`private.key`).

### 5.1 Decriptar com a chave privada
```bash
$rsa = [System.Security.Cryptography.RSA]::Create()

$offset = 0
$privateBytes = [IO.File]::ReadAllBytes("private.key")
$rsa.ImportRSAPrivateKey($privateBytes, [ref]$offset)

$encrypted = [IO.File]::ReadAllBytes("responses/resposta-para-<nome-sobrenome>.enc")

$decryptedBytes = $rsa.Decrypt(
    $encrypted,
    [System.Security.Cryptography.RSAEncryptionPadding]::Pkcs1
)

$mensagemDecriptada = [System.Text.Encoding]::UTF8.GetString($decryptedBytes)

Write-Host "Mensagem decriptada:"
Write-Host $mensagemDecriptada
```

A mensagem decriptada deve ser exatamente o texto que o colega escreveu em `resposta.txt`. Somente o dono da chave privada correspondente à chave pública usada na criptografia pode decriptar a mensagem.

Recomenda-se que outros colegas tentem decriptar mensagens que não são destinadas a eles, para entenderem o conceito de confidencialidade.

---

## Entregáveis Obrigatórios no Classroom

Cada estudante deverá entregar (por exemplo, via Classroom):

1. Um **relatório curto (até 1 página)** contendo:
   - Diferença entre **assinatura digital** e **criptografia**.  
   - Diferença de criptografar com chave pública e chave privada.
   - Riscos associados ao vazamento da chave privada.  
   - Resumo textual do fluxo realizado.