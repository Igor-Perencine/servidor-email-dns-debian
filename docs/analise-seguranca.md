# 🛡️ Análise de Segurança

Auditoria do ambiente **depois** de ele estar funcionando, tratando-o como se fosse um servidor de produção exposto a uma rede corporativa.

**Método:** revisão das configurações (BIND, Postfix, Dovecot, Apache, RainLoop), das portas abertas (`ss -tulpn`), dos serviços ativos e do histórico de administração.

## Resumo

| # | Achado | Severidade | MITRE ATT&CK |
|---|---|---|---|
| 1 | Painel admin do webmail com **senha padrão** | 🔴 Alta | T1078.001 – Default Accounts |
| 2 | Webmail servido apenas em **HTTP** (senhas trafegam em texto puro) | 🔴 Alta | T1557 – Adversary-in-the-Middle |
| 3 | **Firewall sem nenhuma regra** (`nftables` vazio) | 🔴 Alta | T1046 – Network Service Discovery |
| 4 | **RainLoop descontinuado**, com vulnerabilidades conhecidas | 🔴 Alta | T1190 – Exploit Public-Facing Application |
| 5 | Sem **SPF, DKIM e DMARC** (e sem registro MX explícito) | 🟠 Média | T1566 – Phishing (spoofing de remetente) |
| 6 | BIND sem `allow-query` / `allow-recursion` | 🟠 Média | T1498 – Network Denial of Service (amplificação) |
| 7 | **AppArmor desativado** | 🟠 Média | T1562.001 – Impair Defenses |
| 8 | `chmod -R 777` no diretório de zonas do DNS slave | 🟠 Média | T1222 – File Permissions Modification |
| 9 | Webmail envia via porta 25 **sem autenticação** (confia em `mynetworks`) | 🟠 Média | T1078 – Valid Accounts |
| 10 | IMAP/POP3 sem TLS (portas 110 e 143) abertos | 🟠 Média | T1040 – Network Sniffing |
| 11 | Certificados autoassinados (`snakeoil`) | 🟡 Baixa | — |
| 12 | Erros de sintaxe no `main.cf` do Postfix | 🟡 Baixa | — |

---

## 🔴 1. Senha padrão no painel administrativo do webmail

**Evidência:** o painel `/?admin` do RainLoop mantinha o usuário e a senha de fábrica.

**Risco:** qualquer pessoa na rede poderia entrar no painel, redirecionar o IMAP/SMTP para um servidor malicioso e capturar as senhas de todos os usuários no próximo login.

**Correção:**
- Trocar a senha logo após a instalação
- Restringir o acesso ao painel admin por IP no Apache:
  ```apache
  <If "%{QUERY_STRING} =~ /admin/">
      Require ip 192.168.0.0/24
  </If>
  ```

## 🔴 2. Webmail apenas em HTTP

**Evidência:** só a porta **80** está ouvindo. O site `default-ssl.conf` existe, mas não está habilitado.

**Risco:** usuário e senha atravessam a rede em texto puro e podem ser capturados com Wireshark ou `tcpdump`.

**Correção:**
```bash
a2enmod ssl rewrite && a2ensite default-ssl
# redirecionar HTTP -> HTTPS no 000-default.conf:
#   RewriteEngine On
#   RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
systemctl reload apache2
```

## 🔴 3. Firewall sem regras

**Evidência:** [`nftables.conf`](../server01-master/etc/nftables.conf) tem as chains `input`, `forward` e `output` **sem nenhuma regra e com política `accept`**. O SSH (22) e todos os serviços ficam expostos em todas as interfaces, inclusive na NAT.

**Correção** (política *deny by default*):
```
table inet filter {
  chain input {
    type filter hook input priority filter; policy drop;
    ct state established,related accept
    iif lo accept
    icmp type echo-request accept
    ip saddr 192.168.0.0/24 tcp dport { 22, 53, 25, 587, 993, 995, 443 } accept
    ip saddr 192.168.0.0/24 udp dport 53 accept
  }
  chain forward { type filter hook forward priority filter; policy drop; }
  chain output  { type filter hook output  priority filter; policy accept; }
}
```
```bash
systemctl enable --now nftables
```

## 🔴 4. RainLoop descontinuado

**Evidência:** RainLoop **1.17.0**. O projeto não recebe mais atualizações e tem vulnerabilidades públicas, como a **CVE-2022-29360** (XSS armazenado, que permite roubar e-mails da vítima só por ela abrir uma mensagem maliciosa).

**Correção:** migrar para o **SnappyMail** (fork mantido e compatível) ou para o **Roundcube**.

---

## 🟠 5. Sem SPF, DKIM e DMARC

**Evidência:** a [zona DNS](../server01-master/etc/bind/zones/aluno-igorperencine.com.br) não tem registros `MX`, `TXT` (SPF/DMARC) nem chave DKIM.

**Risco:** qualquer servidor pode enviar e-mail se passando por `@aluno-igorperencine.com.br`, o que abre caminho para phishing com remetente falso.

**Correção:**
```dns
@        IN MX  10 mail.aluno-igorperencine.com.br.
mail     IN A   192.168.0.1
@        IN TXT "v=spf1 mx -all"
_dmarc   IN TXT "v=DMARC1; p=reject; rua=mailto:postmaster@aluno-igorperencine.com.br"
```
DKIM: instalar `opendkim`, gerar a chave e publicar o registro `default._domainkey`.

## 🟠 6. BIND sem restrição de consultas

**Evidência:** [`named.conf.options`](../server01-master/etc/bind/named.conf.options) contém apenas `directory`. O servidor responde e faz recursão para **qualquer** cliente que alcance a porta 53.

**Risco:** uso como *open resolver* em ataques de amplificação DNS, e exposição de informações.

**Correção:**
```
options {
    directory "/var/cache/bind";
    allow-query     { 127.0.0.1; 192.168.0.0/24; };
    allow-recursion { 127.0.0.1; 192.168.0.0/24; };
    allow-transfer  { none; };      // liberado só por zona, para o slave
    version "não informado";
    dnssec-validation auto;
};
```

## 🟠 7. AppArmor desativado

**Evidência:** `systemctl stop apparmor` no histórico de administração, usado para contornar um erro de permissão.

**Risco:** o AppArmor confina serviços como `named` e `dovecot`. Sem ele, um serviço comprometido tem acesso a muito mais coisas do sistema.

**Correção:** reativar (`systemctl enable --now apparmor`) e ajustar o perfil específico em `/etc/apparmor.d/local/`, em vez de desligar a proteção inteira.

## 🟠 8. `chmod 777` nas zonas do DNS slave

**Evidência:** `chmod -R 777 *` em `/var/bind` no `server02`.

**Risco:** qualquer usuário local pode alterar as zonas e redirecionar o domínio (*DNS poisoning* local).

**Correção:**
```bash
chown -R bind:bind /var/bind
chmod -R 750 /var/bind
```

## 🟠 9. Webmail envia sem autenticação

**Evidência:** a [config do domínio](../server01-master/webmail/dominio-aluno-igorperencine.com.br.ini) usa `smtp_port = 25`, `smtp_auth = Off` e `smtp_secure = "None"`. Funciona porque o Postfix confia em toda a rede `192.168.0.0/24` (`mynetworks`).

**Risco:** qualquer máquina da rede pode enviar e-mail como qualquer usuário, sem senha.

**Correção:** no webmail, usar a porta **587 com STARTTLS e autenticação**. No Postfix, reduzir `mynetworks` para `127.0.0.0/8`.

## 🟠 10. IMAP/POP3 sem criptografia

**Evidência:** portas **110** e **143** abertas ao lado de 993 e 995.

**Correção:** desabilitar os listeners sem TLS em `10-master.conf` (`port = 0`) ou forçar STARTTLS com `auth_allow_cleartext = no` (sintaxe do Dovecot 2.4).

---

## 🟡 11. Certificados autoassinados

Postfix, Dovecot e Apache usam o certificado `snakeoil` gerado automaticamente. É aceitável em laboratório. Em produção, use **Let's Encrypt** (`certbot`) ou uma CA interna.

## 🟡 12. Erros no `main.cf`

Encontrados na [configuração do Postfix](../server01-master/etc/postfix/main.cf):
- `myorigin = %mydomain` → o correto é **`$mydomain`**
- `smtpd_banner = %myhostname ESMTP $mail_name (Ubuntu) biff = no`: duas diretivas ficaram na mesma linha, e o `%` deveria ser `$`. O banner ainda anuncia "Ubuntu" num servidor Debian.

Recomendação: banner neutro, que não revele software nem versão:
```
smtpd_banner = $myhostname ESMTP
biff = no
```

---

## ✅ Próximos passos

- [ ] Aplicar as correções 1 a 4 (severidade alta)
- [ ] Instalar **Fail2ban** para bloquear força bruta em SSH, SMTP e IMAP
- [ ] Enviar os logs para um **SIEM (Wazuh)** e criar alertas para falhas de login em série
- [ ] Repetir a varredura com `nmap -sV` e comparar o antes e o depois
