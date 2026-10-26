# Prática de Laboratório — Algoritmo RSA

## Tutorial para Windows com PowerShell ou CMD

Criação e uso de chaves RSA no Windows com **OpenSSL**. Os comandos criptográficos são iguais no Windows PowerShell 5.1, PowerShell 7 e Prompt de Comando (CMD); somente a criação e a leitura dos arquivos de texto têm comandos específicos para cada shell.

A prática aborda:

- Geração de par de chaves RSA
- Assinatura digital
- Verificação da assinatura
- Criptografia
- Decriptação

---

## 0. Preparação do ambiente

1. Abra o **PowerShell** ou o **Prompt de Comando (CMD)**.
2. Navegue até a pasta onde deseja trabalhar. Exemplo:
   ```text
   cd C:\Users\SeuUsuario\Desktop
   ```
3. Clone este repositório e entre na pasta:
   ```text
   git clone https://github.com/joseolinda/pratica-rsa.git
   cd pratica-rsa
   ```
4. Todos os comandos abaixo devem ser executados dentro da pasta do repositório clonado.
5. Instale uma distribuição confiável do OpenSSL para Windows, se ainda não estiver instalada, e disponibilize `openssl.exe` no `PATH`. Em computador administrado pela escola, peça a instalação ao suporte.
   ```text
   Install-Module -Name OpenSSL
    ```
    ou baixe em (https://slproweb.com/download/Win64OpenSSL_Light-4_1_0.exe).
6. Confirme a instalação:
   ```text
   openssl version
   ```
   Os comandos desta prática funcionam com OpenSSL 1.1.1 ou 3.x.
7. Arquivos gerados na raiz são ignorados pelo Git, exceto arquivos `.MD` e `.gitignore`. Apenas arquivos nas pastas `public-keys`, `messages` e `responses` serão versionados.
8. Siga as etapas na ordem e use os diretórios indicados.
9.  **Nunca compartilhe nem envie `private.key` ao repositório.** Antes de cada commit, confira os arquivos selecionados.
10. Não edite este arquivo; siga as instruções.

---

## 1. Gerar o par de chaves RSA

No PowerShell ou no CMD, execute:
```text
openssl genrsa -out private.key 2048
openssl rsa -in private.key -pubout -out public.key
```

O arquivo `private.key` contém a chave privada; mantenha-o em local seguro. O arquivo `public.key` é a chave pública.

### Após a criação das chaves

1. Mantenha `private.key` apenas em seu computador. Ela será usada para assinar e decriptar mensagens.
2. Envie apenas `public.key` ao repositório, renomeando-o para:

       public-keys/nome-sobrenome.pub

   No PowerShell ou CMD:
   ```text
   move public.key public-keys\nome-sobrenome.pub
   ```

---

## 2. Criar e assinar uma mensagem

Cada aluno deve criar uma mensagem de texto e assiná-la com sua **chave privada**.

### 2.1 Criar a mensagem

Para manter o arquivo compatível entre versões do Windows e codificações diferentes, use texto sem acentos.

No PowerShell:
```powershell
Set-Content -Path mensagem.txt -Value 'Mensagem destinada a validar autenticidade RSA.' -Encoding Ascii
```

No CMD:
```bat
echo Mensagem destinada a validar autenticidade RSA.> mensagem.txt
```

Se alterar o texto, use exatamente o mesmo arquivo `mensagem.txt` durante a verificação. Não o edite nem o salve novamente depois de assinar.

### 2.2 Assinar a mensagem com a chave privada

No PowerShell ou no CMD:
```text
openssl dgst -sha256 -sign private.key -out assinatura.bin mensagem.txt
```

O comando cria `assinatura.bin` com uma assinatura RSA/SHA-256.

### Enviar a mensagem e a assinatura

Crie uma pasta dentro de `messages` com o nome do destinatário e coloque nela:

- `mensagem.txt`
- `assinatura.bin`

Exemplo:
```text
messages\mensagem-de-jose-para-maria\
    mensagem.txt
    assinatura.bin
```

Faça commit e push somente dos arquivos da mensagem e da assinatura. Não inclua `private.key`.

---

## 3. Verificar a assinatura da mensagem recebida

O aluno B (destinatário) deve verificar a assinatura recebida do colega A, usando a **chave pública** do remetente.

Baixe os arquivos enviados pelo colega A e obtenha a chave pública dele no repositório.

### 3.1 Carregar a chave pública e verificar

No PowerShell ou no CMD, execute na pasta que contém `mensagem.txt` e `assinatura.bin`. Substitua `jose-olinda` pelo nome do remetente:

```text
openssl dgst -sha256 -verify ../../public-keys/jose-olinda.pub -signature assinatura.bin mensagem.txt
```

Se a assinatura for válida, o resultado será:
```text
Verified OK
```

Se a assinatura for inválida, o resultado será:
```text
Verification Failure
```

---

### Interpretação do resultado

Responda: o que significa o resultado?

---

## 4. Criptografar uma resposta para o colega

O aluno B deve enviar uma **resposta confidencial** ao aluno A, criptografada com a **chave pública** do colega A. Não use sua própria chave pública.

### 4.1 Criar a resposta

No PowerShell:
```powershell
Set-Content -Path resposta.txt -Value 'Resposta confidencial para o colega.' -Encoding Ascii
```

No CMD:
```bat
echo Resposta confidencial para o colega.> resposta.txt
```

Você pode alterar o texto; evite acentos para preservar compatibilidade entre shells.

### 4.2 Criptografar com a chave pública do colega

No PowerShell ou no CMD, substitua `nome-sobrenome` pelo nome do destinatário:
```text
openssl pkeyutl -encrypt -in resposta.txt -inkey public-keys/nome-sobrenome.pub -pubin -out responses/resposta-para-nome-sobrenome.enc
```

O OpenSSL usa o padding RSA PKCS#1 v1.5 por padrão neste comando.

### O que o aluno B deve enviar

- Faça commit e push do arquivo `responses/resposta-para-nome-sobrenome.enc`.
- Não envie `resposta.txt` se a mensagem precisar permanecer confidencial.

---

## 5. Decriptar a resposta recebida (obrigatório)

O aluno A, que recebeu `resposta-para-nome-sobrenome.enc`, deve decriptar a resposta usando sua chave privada (`private.key`).

### 5.1 Decriptar com a chave privada

No PowerShell ou no CMD:
```text
openssl pkeyutl -decrypt -in responses/resposta-para-nome-sobrenome.enc -inkey private.key -out resposta-decriptada.txt
```

Para exibir o texto, use o comando do shell aberto.

No PowerShell:
```powershell
Get-Content -Path resposta-decriptada.txt -Raw
```

No CMD:
```bat
type resposta-decriptada.txt
```

A mensagem decriptada deve corresponder ao texto escrito pelo colega. Somente quem possui a chave privada correspondente à chave pública usada na criptografia pode decriptar a mensagem.

---

## Troubleshooting

### Problema: `openssl` não é reconhecido

Instale o OpenSSL para Windows ou peça ajuda ao suporte técnico. Depois de instalar, feche e reabra o terminal e confirme com:
```text
openssl version
```

### Problema: erro ao abrir um arquivo

Confirme que o terminal está na pasta correta e que os nomes e caminhos existem:
```text
dir
dir public-keys
dir responses
```

### Problema: `Verification Failure`

Confirme que:

1. `mensagem.txt` não foi alterado depois de assinado e é o mesmo arquivo usado para a verificação.
2. A chave pública pertence ao remetente.
3. Os comandos de assinatura e verificação usam SHA-256.

---

## Entregáveis Obrigatórios no Classroom

Cada estudante deverá entregar (por exemplo, via Classroom):

1. Um **relatório curto (até 1 página)** contendo:
   - Diferença entre **assinatura digital** e **criptografia**.
   - Diferença de criptografar com chave pública e chave privada.
   - Riscos associados ao vazamento da chave privada.
   - Resumo textual do fluxo realizado.
