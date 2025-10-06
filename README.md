#Objetivo: Montar um laboratório em VirtualBox com Kali (atacante) e Metasploitable2/DVWA (alvo) para simular ataques de força bruta/password spraying (FTP, formulário web, SMB) usando Medusa, documentar os testes e propor mitigação.

#Topologia recomendada: Rede: Host-only ou Internal Network (lab-net).
IPs exemplo: Kali 192.168.56.101, Alvo 192.168.56.102.

#Conteúdo do pacote:
README.md 
relatorio.md 
checklist.txt 
wordlists/ 

#Evidências a coletar: Saída do nmap, logs do Medusa e logs do alvo (vsftpd, apache2, samba).
Arquivos wordlists usados, número de tentativas e comportamento de lockout.

#Mitigações principais:
Alta: MFA, lockout após N falhas, desativar FTP sem TLS.
Média: rate limiting, centralizar logs/SIEM.
Baixa: captchas adaptativos, treinamento de usuários.

#Comandos-modelo:

FTP (força bruta):
medusa -h <ALVO> -u <USUARIO> -P wordlists/small-wordlist.txt -M ftp -f


HTTP FORM (exemplo genérico):
medusa -h <ALVO> -u <USUARIO> -P wordlists/small-wordlist.txt -M http -m FORM:/path/login.php:username:password

SMB (password spraying):
medusa -h <ALVO> -U wordlists/userlist.txt -P wordlists/smb-passlist.txt -M smb
