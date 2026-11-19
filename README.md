# 🛡️ Ataques de Força Bruta com Medusa, Hydra e Kali Linux — Laboratório de Segurança Ofensiva e Defensiva

Projeto prático do desafio da **DIO** sobre auditoria de senhas em ambiente controlado. Aqui eu simulo ataques de força bruta contra serviços **FTP**, **formulário web (DVWA)** e **SMB (password spraying)**, documento cada etapa e — o mais importante — proponho **medidas de mitigação** para cada cenário.

> ⚠️ **Aviso legal e ético**
> Todo o conteúdo deste repositório foi executado em um **laboratório isolado**, com máquinas virtuais **intencionalmente vulneráveis** (Metasploitable 2 e DVWA), em uma **rede interna/host-only sem acesso à internet**.
> Executar ataques de força bruta contra sistemas de terceiros **sem autorização explícita por escrito é crime** (no Brasil, Lei nº 12.737/2012 — "Lei Carolina Dieckmann" — e Marco Civil da Internet). O objetivo deste projeto é **educacional e defensivo**.

---

## 📑 Sumário

- [Objetivos de aprendizagem](#-objetivos-de-aprendizagem)
- [O que é um ataque de força bruta](#-o-que-é-um-ataque-de-força-bruta)
- [Arquitetura do laboratório](#-arquitetura-do-laboratório)
- [Configuração do ambiente](#-configuração-do-ambiente)
- [Reconhecimento com Nmap](#-reconhecimento-com-nmap)
- [Wordlists](#-wordlists)
- [Cenário 1 — Força bruta em FTP](#-cenário-1--força-bruta-em-ftp)
- [Cenário 2 — Formulário web (DVWA)](#-cenário-2--formulário-web-dvwa)
- [Cenário 3 — Password spraying em SMB](#-cenário-3--password-spraying-em-smb)
- [Medidas de mitigação](#-medidas-de-mitigação)
- [Reflexões e aprendizados](#-reflexões-e-aprendizados)
- [Referências](#-referências)

---

## 🎯 Objetivos de aprendizagem

- Compreender ataques de força bruta em diferentes serviços (FTP, Web, SMB);
- Utilizar o **Kali Linux** e o **Medusa** para auditoria de senhas em ambiente controlado;
- Documentar processos técnicos de forma clara e reproduzível;
- Reconhecer vulnerabilidades comuns e propor **medidas de mitigação**;
- Utilizar o GitHub como portfólio técnico.

---

## 🧠 O que é um ataque de força bruta

Um ataque de **força bruta** tenta adivinhar credenciais testando sistematicamente combinações de usuário/senha até encontrar uma válida. Variações importantes:

| Técnica | Como funciona | Quando é usada |
|---|---|---|
| **Força bruta pura** | Testa todas as combinações possíveis de caracteres | Senhas curtas/simples |
| **Ataque de dicionário** | Testa senhas de uma lista pré-definida (*wordlist*) | Senhas comuns e vazadas |
| **Password spraying** | Testa **uma** senha comum contra **muitos** usuários | Evita o bloqueio de conta por tentativas |
| **Credential stuffing** | Reutiliza pares usuário/senha vazados de outros sites | Usuários que repetem senhas |


---

## 🏗️ Arquitetura do laboratório

```
┌─────────────────────────┐        Rede Interna / Bridge           ┌──────────────────────────────┐
│       Kali Linux        │                                        │       Metasploitable 2       │
│   (Atacante / Auditor)  │ <───────────────────────────────────>  │   (Alvo vulnerável)          │
│   192.168.1.101         │                                        │   192.168.56.1.114           │
│   - Medusa              |                                        |   - FTP (vsftpd)  :21        |
|   - Hydra               |                                        │   - SMB (Samba)   :139/445   │
│   - Nmap                │                                        │   - DVWA (Apache) :80        │
│   - enum4linux          │                                        │                              │
└─────────────────────────┘                                        └──────────────────────────────┘
```


## ⚙️ Configuração do ambiente

### 1. Softwares utilizados
- **VirtualBox** (hipervisor)
- **Kali Linux** (VM atacante)
- **Metasploitable 2** (VM alvo) — já traz FTP, SMB e DVWA
- *(Opcional)* DVWA instalado separadamente, caso queira a versão mais recente

### 2. Rede isolada no VirtualBox
Para garantir que nada "vaze" para fora do laboratório:

1. Em cada VM: **Configurações → Rede**
2. Escolha **Rede Interna** (ou **Modo Bridge **)


### 3. Descobrindo os IPs
```bash
# No Kali
ip a

# No Metasploitable 2 (login: msfadmin / msfadmin)
ifconfig
```

### 4. Teste de conectividade
```bash
ping -c 4 192.168.1.114
```
<img width="729" height="618" alt="01-configuracao-rede" src="https://github.com/user-attachments/assets/20028553-4d91-4a8f-a7d1-1b674817da38" />


---

## 🔍 Reconhecimento com Nmap

Antes de atacar, identifique os serviços e portas abertas do alvo:

```bash
# Varredura de serviços e versões
nmap -sV -p 21,22,80,445,139, 192.168.1.114


```

Isso confirma que FTP (21), SMB (139/445) e HTTP (80) estão ativos.

<img width="724" height="399" alt="02 - nmap - scan" src="https://github.com/user-attachments/assets/96ca123c-1c77-4466-8b70-17648a79481a" />


---

## 📝 Wordlists

Para um laboratório, listas pequenas já demonstram o conceito.

**`usuarios.txt`**
```
admin
root
msfadmin
user
service
guest
```

**`senhas.txt`**
```
123456
password
admin
msfadmin
root
qwerty
123456789
letmein
```

O Kali também traz a clássica **rockyou.txt**:
```bash
# Descompactar (vem compactada por padrão)
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
ls -lh /usr/share/wordlists/rockyou.txt
```

> Boa prática: comece com wordlists pequenas para validar o comando e só depois use listas grandes.

---

## 🎯 Cenário 1 — Força bruta em FTP - Medusa

O Metasploitable 2 roda o **vsftpd** na porta 21.

```bash
medusa -h 192.168.1.114 -U usuarios.txt -P senhas.txt -M ftp -t 6 -f
```

- `-U` lista de usuários
- `-P` lista de senhas
- `-M ftp` módulo FTP
- `-t 6` seis tentativas simultâneas
- `-f` para ao encontrar a primeira credencial válida

**Saída esperada (exemplo):**

<img width="791" height="571" alt="medusa-saida-ftp" src="https://github.com/user-attachments/assets/18908769-5c77-481f-ba22-1a67ef8c179c" />


### Validação do acesso
```bash
ftp 192.168.1.114
# informe o usuário/senha encontrados
```

<img width="724" height="280" alt="04 - frp - login - successful" src="https://github.com/user-attachments/assets/7abe6816-8c7d-498e-b5e8-82e70232f044" />


---

## 🌐 Cenário 2 — Formulário web (DVWA) - Hydra

A DVWA (Damn Vulnerable Web Application) possui uma página de login ideal para testar ataques a formulários.

### Preparação
1. Acesse a DVWA pelo navegador: `http://192.168.1.114/dvwa/login.php`
2. Faça login inicial (`admin` / `password`)
3. Em **DVWA Security**, defina o nível como **Low** para o exercício
4. Vá até o módulo **Brute Force**

### Por que trocar o Medusa pelo Hydra neste cenário?

Nos cenários de **FTP** e **SMB**, o Medusa funciona muito bem: são protocolos "diretos", em que a autenticação acontece em uma única troca de mensagens. Já um **formulário web** é diferente — e foi aí que o Medusa mostrou suas limitações para este caso:

| Desafio do formulário DVWA | Por que o Medusa complica | Como o Hydra resolve |
|---|---|---|
| **Cookie de sessão (`PHPSESSID`)** | O módulo `web-form` do Medusa tem suporte limitado e pouco documentado para injetar cookies | O Hydra permite passar cabeçalhos HTTP arbitrários com `H=Cookie: ...` de forma simples |
| **Nível de segurança por cookie (`security=low`)** | Difícil de enviar junto da requisição no Medusa | Enviado no mesmo cabeçalho `H=Cookie:` |
| **Detecção de falha/sucesso** | A sintaxe de `DENY-SIGNAL` do Medusa é mais engessada | O Hydra usa a *string* de falha (`:incorrect:`) de forma clara e flexível |
| **Documentação e exemplos** | Poucos exemplos práticos do `web-form` do Medusa | Grande volume de material sobre `http-get-form`/`http-post-form` do Hydra |

**Resumindo:** o Hydra tem um módulo HTTP mais maduro e flexível para formulários, lidando melhor com **cookies, cabeçalhos e sessões** — exatamente o que um login web moderno exige. Por isso ele é a ferramenta certa aqui, enquanto o Medusa segue sendo a escolha nos cenários de FTP e SMB.

> 💡 Lição de aprendizado: não existe "ferramenta melhor", existe a **ferramenta certa para cada protocolo**. Saber escolher é parte do trabalho de auditoria.

### Captura dos parâmetros
Use o **DevTools (F12) → aba Network** para inspecionar a requisição de login e identificar:
- O método (GET/POST) e a URL do formulário
- Os campos enviados (`username`, `password`, `Login`)
- A **mensagem de erro** exibida em falha (ex.: *"Username and/or password incorrect."*)
- O **cookie de sessão** (`PHPSESSID`) e o nível de segurança (`security=low`)

### Ataque com Hydra (módulo `http-post-form`)
```bash
hydra 192.168.1.114 http-post-form \
    -L usuarios.txt -P senhas.txt \
    "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed"
```

Entendendo a sintaxe do módulo (separada por `:`):

| Parte | Significado |
|---|---|
| `/dvwa/login.php` | Caminho da página de login |
| `username=^USER^&password=^PASS^&Login=Login` | Parâmetros do formulário; `^USER^` e `^PASS^` são substituídos pelas listas |
| `Login failed` | Texto que indica **falha** (se aparecer, a senha está errada) |


Demais parâmetros:
- `-L usuarios.txt` lista de usuarios)
- `-P senhas.txt` lista de senhas


**Saída esperada (exemplo):**
```
[80][http-post-form] host: 192.168.1.114   login: admin   password: password

```

<img width="1606" height="400" alt="05 -dvwa-hydra" src="https://github.com/user-attachments/assets/047737b0-2e3a-4066-9578-7effae3cc112" />


---

---

## 🔑 Cenário 3 — Password spraying em SMB

No *password spraying*, testamos **uma senha comum** contra **vários usuários**, reduzindo o risco de bloqueio de contas.

### Etapa 1 — Enumeração de usuários
```bash
# Com enum4linux
enum4linux -U 192.168.56.20

# Ou com Nmap
nmap -p 445 --script smb-enum-users 192.168.56.20
```
Salve os usuários encontrados em `wordlists/usuarios_smb.txt`.

### Etapa 2 — O spraying com Medusa
```bash
medusa -h 192.168.56.20 \
       -U wordlists/usuarios_smb.txt \
       -p 'msfadmin' \
       -M smbnt \
       -t 1
```

- `-U` lista de usuários enumerados
- `-p 'msfadmin'` **uma única** senha (característica do spraying)
- `-M smbnt` módulo SMB
- `-t 1` baixa concorrência para simular um ataque "silencioso"

### Validação do acesso
```bash
smbclient -L //192.168.56.20 -U msfadmin
```

📸 *`/images/05-smb-spraying.png`*

---

## 🛡️ Medidas de mitigação

O objetivo real de um exercício como este é aprender a **defender**. Abaixo, as contramedidas por camada:

### Políticas de senha e autenticação
- **Senhas fortes e longas** (mínimo 12–14 caracteres, com complexidade) e proibição de senhas comuns/vazadas;
- **Autenticação multifator (MFA/2FA)** — mitiga força bruta mesmo com a senha correta;
- **Rotação e verificação** contra listas de senhas comprometidas (ex.: *Have I Been Pwned*).

### Controles de tentativa
- **Bloqueio de conta** (*account lockout*) após X tentativas falhas;
- **Rate limiting / throttling** nas requisições de autenticação;
- **Atrasos progressivos** (*exponential backoff*) entre tentativas;
- **CAPTCHA** em formulários web após algumas falhas;
- **Fail2ban** para banir IPs com muitas falhas:
  ```bash
  sudo apt install fail2ban
  # configurar jails para sshd, vsftpd, etc. em /etc/fail2ban/jail.local
  ```

### Hardening por serviço
- **FTP:** desabilitar login anônimo, preferir **FTPS/SFTP** (criptografado), restringir usuários;
- **SMB:** **desativar o SMBv1**, exigir **assinatura SMB** (*SMB signing*), restringir sessões nulas/anônimas, aplicar princípio do menor privilégio;
- **Web:** tokens anti-CSRF, sessões seguras (`HttpOnly`, `Secure`), não revelar se o usuário existe nas mensagens de erro.

### Monitoramento e resposta
- **Logs centralizados** e alertas para picos de falhas de autenticação (SIEM);
- **Detecção de password spraying** (muitas contas, uma senha, de um mesmo IP);
- **Segmentação de rede** para limitar o alcance lateral de um invasor.

---

## 💭 Reflexões e aprendizados

> *Preencha com suas impressões pessoais. Algumas perguntas para guiar:*

- O que mais te surpreendeu sobre a velocidade/facilidade de um ataque de dicionário?
- Qual cenário foi mais difícil de automatizar e por quê (ex.: o CSRF da DVWA)?
- Como a diferença entre **força bruta** e **password spraying** muda a estratégia de defesa?
- Depois deste laboratório, quais medidas você aplicaria primeiro em um sistema real?

---


## 📚 Referências

- [Kali Linux — Site Oficial](https://www.kali.org/)
- [Medusa — Documentação](https://github.com/jmk-foofus/medusa/wiki)
- [DVWA — Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Nmap — Manual Oficial](https://nmap.org/book/man.html)
- [Metasploitable 2 — Documentação](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [GitHub Markdown — Guia de sintaxe](https://docs.github.com/pt/get-started/writing-on-github)

---

> Projeto desenvolvido como parte do desafio da **[Digital Innovation One (DIO)](https://www.dio.me/)**.
> Uso estritamente educacional, em ambiente controlado e autorizado.
