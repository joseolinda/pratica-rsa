# Prática de laboratório — RSA no Windows

Roteiro para **Windows PowerShell 5.1 ou PowerShell 7**, com OpenSSL 3.x e Git. Abra o **PowerShell** para executar os blocos abaixo; as variáveis e a criação de arquivos deste roteiro não são comandos do CMD.

Em dupla, **cada estudante realiza todas as etapas**: gerar suas chaves, enviar uma mensagem assinada, verificar a mensagem recebida, detectar uma alteração, enviar uma resposta criptografada e descriptografar a resposta do colega. O [roteiro principal](README.MD) reúne os objetivos e a versão para Linux/macOS.

## 0. Preparar o laboratório

### 0.1 Instalar o OpenSSL Light

**A opção mais simples é baixar pelo navegador:** [Win64 OpenSSL 3.5.9 Light — instalador EXE](https://slproweb.com/download/Win64OpenSSL_Light-3_5_9.exe). A edição **Light é suficiente** para todos os comandos desta prática. O link foi conferido em **03/11/2026**; se a versão mudar, abra o [catálogo oficial da Shining Light Productions](https://slproweb.com/products/Win32OpenSSL.html) e escolha **Win64 OpenSSL Light da série 3.5 LTS**. Esse pacote é para Windows x64; em máquinas x86 ou ARM, escolha a arquitetura correspondente disponível no catálogo. A [política oficial do OpenSSL](https://openssl-library.org/policies/releasestrat/) informa os prazos de suporte das séries.

Se o **GNU Wget já estiver instalado**, esta é uma alternativa no PowerShell:

```powershell
wget.exe -O "$env:TEMP\OpenSSL-Light.exe" "https://slproweb.com/download/Win64OpenSSL_Light-3_5_9.exe"
```

Depois, abra `OpenSSL-Light.exe` na pasta temporária indicada por `$env:TEMP`, ou execute:

```powershell
Start-Process -FilePath "$env:TEMP\OpenSSL-Light.exe" -Wait
```

Use **`wget.exe`**, incluindo a extensão. No Windows PowerShell 5.1, `wget` é um alias de `Invoke-WebRequest`, que possui parâmetros diferentes dos do GNU Wget ([documentação da Microsoft](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-5.1)). Se aparecer “`wget.exe` não é reconhecido”, use o link no navegador; não é necessário instalar outro programa só para baixar o OpenSSL.

Execute o instalador Light. Em computadores administrados pela escola, a instalação pode exigir a conta do responsável pelo laboratório. **`Install-Module -Name OpenSSL` não substitui a instalação do executável `openssl.exe` usado nesta prática.**

Feche e reabra o PowerShell e teste:

```powershell
openssl version
```

Se o comando não for encontrado, acrescente a pasta do executável ao `PATH` **desta sessão** e teste novamente:

```powershell
Test-Path 'C:\Program Files\OpenSSL-Win64\bin\openssl.exe'
$env:Path = 'C:\Program Files\OpenSSL-Win64\bin;' + $env:Path
openssl version
```

O primeiro comando deve retornar `True`. Se retornar `False`, localize a pasta escolhida durante a instalação e ajuste o caminho antes de continuar. A alteração acima vale apenas para a janela atual; se abrir outra janela, execute-a novamente. O responsável pelo laboratório também pode cadastrar essa pasta no `PATH` do Windows.

### 0.2 Obter o projeto

Confira também se o Git está disponível:

```powershell
git --version
```

Se não for reconhecido, instale o [Git for Windows](https://gitforwindows.org/) ou peça ao responsável pelo laboratório para instalá-lo. Depois, reabra o PowerShell e confira novamente.

Use uma pasta na qual você possa salvar arquivos. Cada estudante deve trabalhar em **seu próprio clone** e ter acesso ao repositório usado pela turma:

```powershell
git clone https://github.com/joseolinda/pratica-rsa.git
Set-Location pratica-rsa
```

Se já clonou, entre na pasta existente. **Daqui até o final, execute todos os comandos na raiz `pratica-rsa`; não entre nas subpastas.** Confira:

```powershell
Get-Location
Get-ChildItem
```

### 0.3 Definir os nomes e os diretórios

Combine com o colega dois identificadores únicos na turma, sem espaços nem acentos, como `ana-silva` e `bruno-lima`. Os exemplos usam `alice` e `bruno`. No computador de Bruno, inverta os valores de `$eu` e `$colega`.

```powershell
$eu = 'alice'
$colega = 'bruno'
$raiz = (Get-Location).Path
$local = "local/$eu"
$enviada = "messages/de-$eu-para-$colega"
$recebida = "messages/de-$colega-para-$eu"
$respostaEnviada = "responses/de-$eu-para-$colega"
$respostaRecebida = "responses/de-$colega-para-$eu"
$utf8 = New-Object System.Text.UTF8Encoding -ArgumentList $false
New-Item -ItemType Directory -Force -Path $local, 'public-keys', $enviada, $respostaEnviada | Out-Null
```

As variáveis duram somente enquanto essa janela estiver aberta. Se precisar reabrir o PowerShell, volte à raiz do projeto e execute novamente este bloco com os mesmos nomes.

| Pasta | Conteúdo | Enviar ao repositório? |
| --- | --- | --- |
| `local/alice/` | Chave privada, texto secreto e texto descriptografado de Alice | **Não**; ignorada pelo Git |
| `public-keys/` | Chaves públicas de todos | Sim |
| `messages/de-alice-para-bruno/` | Mensagem pública e assinatura de Alice | Sim |
| `responses/de-alice-para-bruno/` | Resposta criptografada para Bruno | Sim |

Não reutilize a chave privada de outra pessoa, inclusive em computadores compartilhados. Preserve sua chave privada durante toda a prática. **Não envie `local/` nem arquivos de chave privada.**

**Se você já começou a versão anterior da prática:** preserve a antiga `private.key` da raiz. Para continuar com aquela identidade, copie sua própria chave para `local/<seu-identificador>/private.key`, sem sobrescrever outro arquivo, e confirme com o professor que ela corresponde à chave pública já publicada. Não gere outra chave por cima. Respostas antigas feitas com o padding padrão PKCS#1 v1.5 não usam os mesmos parâmetros deste roteiro; para o novo fluxo, a dupla deve gerar novas respostas com OAEP/SHA-256.

## 1. Gerar e compartilhar a chave pública

Execute esta geração **uma única vez**. O bloco só gera chaves quando os dois caminhos estão livres, protegendo tanto sua chave privada quanto uma chave pública já publicada com esse identificador:

```powershell
if ((Test-Path "$local/private.key") -or (Test-Path "public-keys/$eu.pub")) {
    Write-Host 'Ja existe uma chave em um dos caminhos. Nenhuma chave foi gerada; confira as instrucoes abaixo.'
} else {
    openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out "$local/private.key"
    if ($LASTEXITCODE -ne 0) { throw 'Falha ao gerar a chave privada. Resolva antes de continuar.' }
    openssl pkey -in "$local/private.key" -pubout -out "public-keys/$eu.pub"
    if ($LASTEXITCODE -ne 0) { throw 'Falha ao exportar a chave publica. Resolva antes de continuar.' }
    openssl pkey -in "$local/private.key" -check -noout
}
```

Se os dois arquivos já existem e formam **seu próprio par de chaves**, pule a geração e preserve o par. Se apenas um existe, se a chave pública pertence a outra pessoa ou se você não tem certeza, resolva com o professor antes de continuar; para uma nova identidade, escolha outro identificador livre e ajuste as variáveis. Substituir a chave privada impede descriptografar mensagens destinadas à chave anterior.

A chave privada permite **assinar e descriptografar**. A chave pública permite **verificar assinaturas e criptografar para o dono da chave privada**. A chave pública começa com `-----BEGIN PUBLIC KEY-----`; a privada permanece em `local/`.

Publique apenas sua chave pública:

```powershell
git add -- "public-keys/$eu.pub"
git diff --cached --name-only
```

Confira a lista antes de prosseguir: ela deve conter apenas sua chave pública. Se houver algo de outra atividade ou uma chave privada, pare e retire esse arquivo da seleção com `git restore --staged -- caminho-do-arquivo`. Não use `git add -f` para contornar os arquivos ignorados. Depois de conferir:

```powershell
git commit -m "Publica chave de $eu"
git pull --rebase
git push
```

**Aguarde os dois publicarem suas chaves.** Atualize seu clone:

```powershell
git pull --ff-only
Test-Path "public-keys/$colega.pub"
```

O teste deve retornar `True`. Confirme presencialmente com o colega que o arquivo é dele. Para comparar também a impressão digital da chave, cada um calcula o SHA-256 da própria chave pública e informa o resultado ao outro por um canal confiável:

```powershell
openssl pkey -pubin -in "public-keys/$eu.pub" -outform DER -out "$local/publica.der"
openssl dgst -sha256 "$local/publica.der"
openssl pkey -pubin -in "public-keys/$colega.pub" -outform DER -out "$local/colega.der"
openssl dgst -sha256 "$local/colega.der"
```

Compare os valores hexadecimais, não os nomes dos arquivos. Uma assinatura válida só identifica o colega quando a chave pública usada na verificação foi confirmada como sendo dele.

## 2. Criar, assinar e enviar uma mensagem

Escreva uma mensagem destinada ao colega. Ela será **pública**, pois assinatura digital não oculta o conteúdo. O comando abaixo grava UTF-8 sem BOM, incluindo acentos, de forma consistente entre PowerShell 5.1 e 7:

```powershell
[System.IO.File]::WriteAllText((Join-Path $raiz "$enviada/mensagem.txt"), "Olá, $colega! Esta mensagem foi enviada por $eu.", $utf8)
openssl dgst -sha256 -sign "$local/private.key" -out "$enviada/assinatura.bin" "$enviada/mensagem.txt"
```

São dois arquivos: `mensagem.txt` contém o texto; `assinatura.bin` contém a assinatura RSA/SHA-256 calculada com sua chave privada. Depois de assinar, **não edite nem salve novamente a mensagem**. Para mudar o texto, assine novamente antes de enviar.

```powershell
git add -- "$enviada/mensagem.txt" "$enviada/assinatura.bin"
git diff --cached --name-only
```

Confira se estão selecionados somente a mensagem e a assinatura enviadas. Depois:

```powershell
git commit -m "Envia mensagem assinada de $eu para $colega"
git pull --rebase
git push
```

## 3. Conferir a mensagem recebida e detectar alterações

### 3.1 Verificar o arquivo original

**Aguarde o colega concluir a etapa 2** e atualize seu clone:

```powershell
git pull --ff-only
Get-Content -LiteralPath "$recebida/mensagem.txt" -Encoding UTF8 -Raw
openssl dgst -sha256 -verify "public-keys/$colega.pub" -signature "$recebida/assinatura.bin" "$recebida/mensagem.txt"
$LASTEXITCODE
```

O resultado esperado é **`Verified OK`** e código **`0`**. Isso indica que os bytes da mensagem correspondem à assinatura e à chave pública informada. Com a chave confirmada na etapa 1, você tem evidência de autenticidade e integridade. A assinatura não torna o texto confidencial nem garante que o conteúdo escrito seja verdadeiro.

### 3.2 Alterar uma cópia e verificar novamente

Preserve o arquivo recebido. Faça a experiência apenas com uma cópia local:

```powershell
Copy-Item -LiteralPath "$recebida/mensagem.txt" -Destination "$local/mensagem-alterada.txt"
[System.IO.File]::AppendAllText((Join-Path $raiz "$local/mensagem-alterada.txt"), ' ALTERACAO DE TESTE', $utf8)
openssl dgst -sha256 -verify "public-keys/$colega.pub" -signature "$recebida/assinatura.bin" "$local/mensagem-alterada.txt"
$LASTEXITCODE
```

O esperado é **falha de verificação** e código **diferente de `0`**, geralmente com `Verification failure` ou `Verification Failure`. Uma alteração no texto invalida a assinatura original, mesmo que a assinatura não tenha mudado.

### 3.3 Usar a chave pública errada

Verifique a mensagem original do colega usando **sua própria chave pública**:

```powershell
openssl dgst -sha256 -verify "public-keys/$eu.pub" -signature "$recebida/assinatura.bin" "$recebida/mensagem.txt"
$LASTEXITCODE
```

Também deve falhar. Registre os três resultados: mensagem original com chave correta, cópia alterada com chave correta e mensagem original com chave errada. Não publique as cópias de teste.

## 4. Criar e enviar um texto criptografado

Cada estudante escreve uma resposta confidencial ao colega. Use um texto curto, pois **RSA de 2048 bits com OAEP/SHA-256 aceita no máximo 190 bytes** por operação. Acentos podem ocupar mais de um byte em UTF-8. Esta limitação pertence ao RSA direto; arquivos maiores normalmente usam criptografia híbrida.

```powershell
[System.IO.File]::WriteAllText((Join-Path $raiz "$local/resposta.txt"), "Resposta secreta de $eu para $colega.", $utf8)
$tamanho = (Get-Item -LiteralPath "$local/resposta.txt").Length
$tamanho
if ($tamanho -gt 190) { throw 'Reduza a resposta para no maximo 190 bytes antes de criptografar.' }
```

Com o texto dentro do limite, criptografe usando a **chave pública do destinatário**, mantendo todos os parâmetros explícitos:

```powershell
openssl pkeyutl -encrypt -pubin -inkey "public-keys/$colega.pub" -in "$local/resposta.txt" -out "$respostaEnviada/resposta.enc" -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
$LASTEXITCODE
```

Só prossiga se o código for `0`. O arquivo `resposta.enc` é binário e não deve ser aberto e salvo como texto. O colega precisará da chave privada correspondente à chave pública que você usou.

Envie **somente o arquivo criptografado**. O texto `resposta.txt` permanece na pasta local:

```powershell
git add -- "$respostaEnviada/resposta.enc"
git diff --cached --name-only
```

Confira se somente o arquivo `resposta.enc` está selecionado. Depois:

```powershell
git commit -m "Envia resposta criptografada de $eu para $colega"
git pull --rebase
git push
```

Criptografar com a chave pública do colega protege a confidencialidade. Esse comando, sozinho, não comprova quem enviou a resposta: qualquer pessoa que possua a chave pública pode criptografar para ele.

## 5. Descriptografar a resposta recebida

**Aguarde o colega concluir a etapa 4** e atualize seu clone. Descriptografe usando **sua própria chave privada**, com os mesmos parâmetros OAEP usados na criptografia:

```powershell
git pull --ff-only
openssl pkeyutl -decrypt -inkey "$local/private.key" -in "$respostaRecebida/resposta.enc" -out "$local/resposta-recebida.txt" -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
$LASTEXITCODE
```

O código deve ser `0`. Leia o texto recuperado:

```powershell
Get-Content -LiteralPath "$local/resposta-recebida.txt" -Encoding UTF8 -Raw
```

Confirme com o colega, presencialmente, que o texto recuperado corresponde ao que ele escreveu. Não publique a resposta em texto claro nem sua chave privada para fazer essa comparação.

### Experiência: tentar abrir uma resposta destinada ao colega

Use a resposta que **você enviou** na etapa 4: ela foi criptografada para a chave pública do colega. Tente abri-la com **sua** chave privada e uma saída de teste separada:

```powershell
openssl pkeyutl -decrypt -inkey "$local/private.key" -in "$respostaEnviada/resposta.enc" -out "$local/tentativa-chave-errada.txt" -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
$LASTEXITCODE
```

O resultado esperado é erro, como `oaep decoding error`, e código diferente de `0`. Um arquivo de saída vazio não significa que houve descriptografia: confira o código de saída. Cada estudante deve ter sua própria chave para que esta experiência funcione.

## 6. Registrar o aprendizado

Cada estudante entrega **no Classroom um relatório de até uma página**, identificando seu nome, o colega e os identificadores usados. Inclua evidências dos comandos e resultados, contendo:

1. Nome dos arquivos de chave pública e chave privada e a função de cada uma. **Não mostre o conteúdo da chave privada.**
2. Caminho da mensagem e da assinatura enviadas.
3. Resultado da verificação da mensagem original, da cópia alterada e do teste com a chave pública errada.
4. Caminho do texto criptografado enviado, resultado da descriptografia da resposta recebida e resultado do teste com a chave privada errada.
5. Explicação: por que a assinatura detecta alterações? Por que criptografar para alguém exige a chave pública dessa pessoa? Por que assinar com chave privada não é “criptografar um texto com a chave privada”? O que acontece se a chave privada vazar ou for perdida?

Os dois integrantes devem concluir o fluxo completo. Uma assinatura válida e uma descriptografia bem-sucedida demonstram propriedades diferentes; explique quais são elas.

## Solução de problemas

| Sintoma | O que conferir |
| --- | --- |
| `wget.exe` não é reconhecido | Use o link de download no navegador. `wget` sem extensão pode chamar um alias do PowerShell. |
| `openssl` não é reconhecido | Confira a instalação Light, o caminho de `openssl.exe` e o `PATH` da sessão na etapa 0.1. |
| Falta de DLL ou dependência do Windows | Siga as dependências indicadas no catálogo do instalador e peça ao responsável pelo laboratório a instalação necessária. |
| `Cannot open file`, arquivo inexistente ou caminho vazio | Execute `Get-Location`; volte à raiz, confira as variáveis e use `Get-ChildItem public-keys`, `Get-ChildItem messages` e `Get-ChildItem responses`. Aguarde o envio do colega e faça `git pull --ff-only`. |
| Assinatura do original falha | Confirme a chave do remetente e os mesmos bytes da mensagem. Salvar em outra codificação ou trocar quebras de linha altera os bytes. Baixe novamente a versão enviada, sem editar. |
| `data too large for key size` | Reduza `local/<seu-nome>/resposta.txt` para até 190 bytes, conferindo `.Length` como na etapa 4. |
| Descriptografia falha | Confirme o destinatário, sua chave privada original e os três parâmetros OAEP/SHA-256/MGF1. Arquivos de atividades antigas que usaram outro padding não seguem este roteiro. |
| `git push` rejeitado porque o remoto mudou | Depois de conferir e fazer commit dos seus arquivos, execute `git pull --rebase` e tente `git push` novamente. Não use `push --force`. |
| Sem permissão de envio ao repositório | Use o repositório indicado pelo professor ou transfira apenas a chave pública, o par mensagem/assinatura e o arquivo `.enc` pelo canal combinado. Preserve os mesmos caminhos ao recebê-los. |

Para inspecionar arquivos binários, use os comandos do OpenSSL; não passe assinaturas ou textos criptografados por `Get-Content`, `Set-Content` ou redirecionamentos de texto do PowerShell. Preserve os arquivos exatamente como foram gerados.
