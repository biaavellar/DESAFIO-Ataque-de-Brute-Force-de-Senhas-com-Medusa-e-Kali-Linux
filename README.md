# Laboratório: Kali Linux + Medusa (Força Bruta) — Projeto Prático

> Aviso: este laboratório destina-se **exclusivamente** a ambientes controlados e experimentação legal (suas VMs locais). Não execute ataques contra sistemas sem autorização explícita.

## Objetivo
Montar um laboratório com **Kali Linux** (atacante) e **Metasploitable 2 / DVWA** (alvo) em VirtualBox para simular ataques de força bruta (FTP, formulário web, SMB) usando **Medusa**. Documentar os testes, evidências e recomendações de mitigação.

## Estrutura do repositório (conteúdo do ZIP)
- `README.md` — este arquivo.
- `wordlists/` — wordlists de exemplo (curtas).
- `template_relatorio.md` — modelo de relatório técnico para preencher com evidências.
- `checklist.txt` — checklist de mitigação/prioridades.

## Topologia recomendada (VirtualBox)
- Rede: **Host-only** (ex.: `vboxnet0`) ou **Internal Network** chamada `lab-net`.
- VMs:
  - Kali Linux: 2 NICs (NAT para updates opcional + Host-only `lab-net`).
  - Metasploitable 2 / DVWA: 1 NIC (Host-only `lab-net`).
- Exemplo de IPs:
  - Kali: `192.168.56.101`
  - Alvo: `192.168.56.102`

## Preparação rápida
1. Importe/instale as VMs (Kali e Metasploitable/DVWA).
2. Configure adaptadores de rede conforme a topologia acima.
3. Teste conectividade: `ping 192.168.56.102` (a partir da Kali).
4. Faça scan inicial (nmap) e registre o output:
   ```bash
   nmap -sV -oN nmap_initial.txt 192.168.56.102
   ```
5. Assegure-se de ter snapshots antes dos testes para restaurar rapidamente.

## Wordlists
As wordlists neste pacote são curtas e **apenas** para laboratório:
- `wordlists/small-wordlist.txt`
- `wordlists/smb-passlist.txt`
- `wordlists/userlist.txt`

Você pode editar/ampliar conforme necessário.

## Exemplos de comandos (MODELOS — use apenas em seu laboratório)
> Observação: substitua `<ALVO>`, `<USUARIO>`, `<WORDLIST>` pelos valores do seu ambiente.

**FTP (força bruta simples)**
```bash
medusa -h <ALVO> -u <USUARIO> -P wordlists/small-wordlist.txt -M ftp -f
```
Opções:
- `-h` host alvo
- `-u` usuário
- `-U` arquivo com vários usuários
- `-P` arquivo com senhas
- `-M` módulo (ftp, smb, http, etc.)
- `-f` para parar ao primeiro sucesso

**HTTP (formulário web — exemplo genérico)**
Medusa pode trabalhar com HTTP FORM, mas formulários complexos requerem Burp/ZAP.
```bash
medusa -h <ALVO> -u <USUARIO> -P wordlists/small-wordlist.txt -M http -m FORM:/path/login.php:username:password
```
> Para DVWA, prefira usar Burp Intruder para entender requisições CSRF e sessões.

**SMB (password spraying com lista de usuários)**
```bash
medusa -h <ALVO> -U wordlists/userlist.txt -P wordlists/smb-passlist.txt -M smb
```

## Logs e evidências a coletar
- `nmap` scan (salve saída).
- Output do Medusa (redirecione para arquivo): `... | tee medusa_ftp_output.txt`
- Prints de tela (cliente FTP, acesso a DVWA, smbclient).
- Logs do alvo (ex.: `/var/log/vsftpd.log`, `/var/log/apache2/access.log`, `/var/log/samba/`).
- Número de tentativas, tempo total e comportamento de lockout.

## Recomendações rápidas de mitigação
- Usar SFTP/FTPS em vez de FTP simples.
- Implementar bloqueio de conta/lockout após N falhas.
- Rate limiting em formulários web + captchas adaptativos.
- Autenticação multifator (MFA).
- Monitoramento e alertas (SIEM) para spikes de autenticação.

## Template de documentação
Veja `template_relatorio.md` para um relatório pronto a preencher com resultados e evidências.

## Ética e legalidade
Registre consentimentos, mantenha logs e não reutilize credenciais obtidas fora do ambiente de teste. Este material não deve ser usado para ações maliciosas.

