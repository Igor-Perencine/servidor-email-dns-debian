# 🛠️ Instalação passo a passo

Guia para reproduzir o ambiente em **Debian 13 (trixie)** no VirtualBox. Todos os comandos são executados como `root` (`su -`).

---

## 0. Máquinas virtuais

| VM | RAM | Placa 1 | Placa 2 |
|---|---|---|---|
| `server01` | 2 GB | NAT | Rede interna `Fatec` |
| `server02` | 1–2 GB | NAT | Rede interna `Fatec` |

No VirtualBox: **Configurações → Rede → Adaptador 2 → Conectado a: Rede Interna → Nome: `Fatec`**.

---

## 1. Rede (as duas VMs)

Arquivo `/etc/network/interfaces` (no `server02`, troque o IP para `192.168.0.2`):

```
auto enp0s8
iface enp0s8 inet static
    address 192.168.0.1
    netmask 255.255.255.0
```

```bash
systemctl restart networking
ip a show enp0s8
```

Em `/etc/hostname`, coloque `server01` ou `server02`.

---

## 2. DNS Master (server01)

```bash
apt install -y bind9 bind9-utils
mkdir -p /etc/bind/zones
```

1. Declare as zonas em `/etc/bind/named.conf.local` → [ver arquivo](../server01-master/etc/bind/named.conf.local)
2. Crie a zona direta → [`zones/aluno-igorperencine.com.br`](../server01-master/etc/bind/zones/aluno-igorperencine.com.br)
3. Crie a zona reversa → [`zones/db.0.168.192`](../server01-master/etc/bind/zones/db.0.168.192)

Valide antes de reiniciar:

```bash
named-checkconf
named-checkzone aluno-igorperencine.com.br /etc/bind/zones/aluno-igorperencine.com.br
named-checkzone 0.168.192.in-addr.arpa /etc/bind/zones/db.0.168.192
systemctl restart named
dig @127.0.0.1 aluno-igorperencine.com.br
```

> 💡 Sempre que alterar uma zona, **incremente o serial** (formato `AAAAMMDDnn`). Sem isso, o slave não percebe a mudança.

---

## 3. DNS Slave (server02)

```bash
apt install -y bind9 bind9-utils
mkdir -p /var/bind
chown bind:bind /var/bind        # ✅ correto (NÃO use chmod 777)
```

Configure `/etc/bind/named.conf.local` → [ver arquivo](../server02-slave/etc/bind/named.conf.local)

```bash
named-checkconf
systemctl restart named
journalctl -u named | grep -i transfer   # deve mostrar "Transfer completed"
dig @192.168.0.2 aluno-igorperencine.com.br
```

---

## 4. Postfix + SASL (server01)

```bash
apt install -y postfix sasl2-bin
```

Na instalação, escolha **"Internet Site"** e informe o domínio `aluno-igorperencine.com.br`.

**4.1 `/etc/postfix/main.cf`.** Parâmetros principais → [ver arquivo](../server01-master/etc/postfix/main.cf):

```
myhostname   = mail.aluno-igorperencine.com.br
mydomain     = aluno-igorperencine.com.br
myorigin     = $mydomain
mydestination = $myhostname, aluno-igorperencine.com.br, localhost
mynetworks   = 127.0.0.0/8 192.168.0.0/24
home_mailbox = Maildir/
smtpd_sasl_path = smtpd
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
```

**4.2 Porta 587 (submission)** em `/etc/postfix/master.cf`. Exige TLS e login → [ver arquivo](../server01-master/etc/postfix/master.cf):

```
submission inet n - n - - smtpd
   -o syslog_name=postfix/submission
   -o smtpd_tls_security_level=encrypt
   -o smtpd_sasl_auth_enable=yes
   -o smtpd_tls_auth_only=yes
   -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

**4.3 SASL.** O Postfix roda em *chroot*, então o socket do `saslauthd` precisa ficar dentro de `/var/spool/postfix`:

```bash
# /etc/postfix/sasl/smtpd.conf
#   pwcheck_method: saslauthd
#   mech_list: PLAIN LOGIN

mkdir -p /var/spool/postfix/var/run/saslauthd
dpkg-statoverride --add root sasl 710 /var/spool/postfix/var/run/saslauthd
adduser postfix sasl
```

Em `/etc/default/saslauthd`:

```
START=yes
MECHANISMS="pam"
OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd"
```

```bash
systemctl restart saslauthd postfix
postfix check
```

**4.4 Aliases.** Em `/etc/aliases`, encaminhe os e-mails do `root` para um usuário real:

```bash
newaliases
```

---

## 5. Dovecot (server01)

```bash
apt install -y dovecot-imapd dovecot-pop3d
```

`/etc/dovecot/conf.d/10-mail.conf`. Atenção à **sintaxe nova do Dovecot 2.4** → [ver arquivo](../server01-master/etc/dovecot/conf.d/10-mail.conf):

```
mail_driver = maildir
mail_path   = ~/Maildir
```

`/etc/dovecot/conf.d/10-auth.conf`:

```
auth_mechanisms = plain login
```

Crie o modelo de caixa de e-mail para novos usuários:

```bash
maildirmake.dovecot /etc/skel/Maildir
for p in Drafts Sent Trash Templates; do maildirmake.dovecot /etc/skel/Maildir/.$p; done
systemctl restart dovecot
doveconf -n | grep mail_
```

---

## 6. Usuários de e-mail

Cada usuário do sistema Linux é uma conta de e-mail:

```bash
useradd -m -s /bin/bash teste
passwd teste
testsaslauthd -f /var/spool/postfix/var/run/saslauthd/mux -u teste -p '<senha>'
```

---

## 7. Webmail RainLoop (server01)

> ⚠️ O RainLoop foi **descontinuado** e tem vulnerabilidades conhecidas. Para um ambiente novo, use o **[SnappyMail](https://snappymail.eu/)** (sucessor compatível) ou o **Roundcube**. Veja a [análise de segurança](analise-seguranca.md).

```bash
apt install -y apache2 php libapache2-mod-php php-xml php-zip php-curl php-gd unzip curl
mkdir -p /var/www/html/mail && cd /var/www/html/mail
# baixe o instalador oficial do webmail escolhido e execute-o aqui
chown -R www-data:www-data /var/www/html/mail
find /var/www/html/mail -type d -exec chmod 755 {} \;
find /var/www/html/mail -type f -exec chmod 644 {} \;
systemctl restart apache2
```

1. Acesse `http://192.168.0.1/mail/?admin`
2. **Troque imediatamente a senha padrão do administrador**
3. Em *Domains*, adicione `aluno-igorperencine.com.br` com IMAP e SMTP apontando para o próprio servidor → [exemplo](../server01-master/webmail/dominio-aluno-igorperencine.com.br.ini)
4. Faça login em `http://192.168.0.1/mail/` com `teste@aluno-igorperencine.com.br`

---

## 8. Testes finais

```bash
ss -tlnp | grep -E ':(25|587|143|993|110|995|80)\b'
openssl s_client -connect aluno-igorperencine.com.br:993 -quiet
echo "teste" | mail -s "Olá" teste@aluno-igorperencine.com.br
ls /home/teste/Maildir/new/
journalctl -u postfix -n 20 --no-pager
```

---

## 🧯 Problemas comuns

| Sintoma | Causa | Solução |
|---|---|---|
| `SASL authentication failure: cannot connect to saslauthd` | Socket fora do chroot do Postfix | Passo 4.3 (`-m /var/spool/postfix/var/run/saslauthd`) |
| Dovecot ignora `mail_location` | Sintaxe antiga (Dovecot ≤ 2.3) | Usar `mail_driver` + `mail_path` |
| Slave não recebe a zona | Pasta sem permissão para o usuário `bind`, ou serial não incrementado | `chown bind:bind` e incrementar o serial |
| Webmail mostra erro de permissão em `data/` | Dono errado | `chown -R www-data:www-data` |
