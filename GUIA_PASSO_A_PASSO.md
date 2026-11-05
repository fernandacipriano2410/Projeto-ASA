# Guia didático — Projeto 01, Vagrant e Ansible

## Antes de começar
O enunciado exige as quatro VMs, mais configurações específicas. Nossa versão evita Python, inventários gerados e ferramentas fora do pedido. **Não execute todos os comandos sem conferir cada etapa.**

### 1 — Confira as ferramentas
```bash
vagrant --version
VBoxManage --version
ansible --version
```
**Por quê?** O VirtualBox virtualiza os computadores; o Vagrant descreve a infraestrutura; o Ansible instala e configura os serviços.

### 2 — Veja as configurações da dupla
Abra `Vagrantfile` e `ansible/vars.yml`. As matrículas terminam em `38` e `04`. Por isso `arq=192.168.56.138` e `db=192.168.56.104`. Interpretamos `XY` como 84, então `app=192.168.56.184`. Essa interpretação deve ser confirmada pelo professor.

### 3 — Prepare a rede host-only
```bash
VBoxManage list hostonlyifs
VBoxManage list dhcpservers
```
Use no `Vagrantfile` o nome correto da interface, normalmente `vboxnet0`. A rede do hospedeiro deve ser `192.168.56.1/24`. Pelo VirtualBox Manager, crie/ajuste a rede host-only se ela não existir. Desabilite o servidor DHCP do VirtualBox **somente nessa rede**; o Vagrant também possui um gatilho para desabilitá-lo, mas confira o resultado, pois a CLI varia de versão para versão.

**Por quê?** O `arq` é quem deve atribuir IPs às VMs. O VirtualBox não pode concorrer pelo mesmo pool. **Não desative o DHCP da interface NAT**, necessária para o acesso remoto inicial.

### 4 — Crie as chaves dos estudantes
Na raiz do projeto, execute:
```bash
mkdir -p chaves
ssh-keygen -t ed25519 -f chaves/fernanda -C fernanda-ifpb
ssh-keygen -t ed25519 -f chaves/julia -C julia-ifpb
```
Use uma senha nas chaves privadas, se desejar. Os arquivos `.pub` são instalados pelo Ansible nas contas `fernanda` e `julia`. **Não envie os arquivos privados ao GitHub**.

### 5 — Instale coleções necessárias
```bash
ansible-galaxy collection install -r requirements.yml
```
**Por quê?** Algumas funções (volume lógico, fuso horário, chave SSH, montagem) estão em coleções Ansible.

### 6 — Crie as VMs
```bash
vagrant up arq --no-provision
vagrant up db app cli --no-provision
vagrant status
```
**Por quê?** O `Vagrantfile` configura uma VM de arquivos com três discos extras, uma de banco, uma web e uma cliente. Todas usam Debian Bookworm, linked clones, 512 MB (menos cli com 1024 MB) e SSH sem inserção automática de chaves.

**Atenção:** se as portas 2221–2224 já estiverem ocupadas, feche outros Vagrants ou altere Vagrantfile e inventário juntos.

### 7 — Teste SSH pelo Vagrant
```bash
vagrant ssh arq
hostname
exit
vagrant ssh-config arq
```
Confira `IdentityFile` e `Port`. O inventário usa `~/.vagrant.d/insecure_private_key`; se a box devolver outro arquivo, atualize `ansible/inventory.ini` para cada máquina. A verificação aqui evita erro de permissão e autenticação depois.

### 8 — Verifique o inventário Ansible
```bash
cd ansible
ansible todos -m ping
```
O resultado esperado é `pong` em todos os hosts. O Ansible se conecta pela primeira placa (NAT/SSH) enquanto a segunda placa ainda aguarda DHCP.

### 9 — Configure o ARQ primeiro
```bash
ansible-playbook site.yml --limit arq
```
Ele atualiza o Debian, cria grupo/usuárias, instala SSH, Chrony, DHCP, DNS, LVM e NFS. **LVM modifica os três discos extras; use exclusivamente as VMs desse projeto.**

Verifique:
```bash
vagrant ssh arq -c 'sudo systemctl status isc-dhcp-server --no-pager'
vagrant ssh arq -c 'sudo systemctl status bind9 --no-pager'
vagrant ssh arq -c 'sudo vgs && sudo lvs'
vagrant ssh arq -c 'findmnt /dados'
vagrant ssh arq -c 'sudo exportfs -v'
```
Execute os comandos `vagrant` na raiz do projeto (não dentro de `ansible`).

### 10 — Configure DB, APP e CLI
No diretório `ansible`:
```bash
ansible-playbook site.yml --limit demais
```
O Ansible primeiro configura a interface privada para receber DHCP do ARQ, depois instala as configurações comuns e os serviços específicos. Ele faz a montagem automática via autofs em `/var/nfs`.

### 11 — Faça os testes do enunciado
Na raiz:
```bash
vagrant ssh db -c 'ip -4 addr; systemctl is-active mariadb; ls /var/nfs'
vagrant ssh app -c 'ip -4 addr; systemctl is-active apache2; ls /var/nfs'
vagrant ssh cli -c 'ip -4 addr; grep X11Forwarding /etc/ssh/sshd_config.d/70-ifpb-x11.conf'
vagrant ssh arq -c 'dig @192.168.56.138 app.fernanda.julia.devops +short'
vagrant ssh arq -c 'dig @192.168.56.138 -x 192.168.56.104 +short'
```
No navegador do computador Linux, acesse `http://192.168.56.184` para verificar o site do Apache. O `ls /var/nfs` dispara a montagem autofs. Para confirmar:
```bash
vagrant ssh app -c 'findmnt /var/nfs'
```
Teste a conexão por nomes nas VMs (resolução DNS):
```bash
vagrant ssh cli -c 'getent hosts arq.fernanda.julia.devops'
```
Se o sistema estiver usando DNS NAT em vez do DNS do ARQ, ajuste o resolvedor da interface privada para `192.168.56.138` antes de considerar o requisito validado. O host Linux também precisará consultar o DNS do ARQ para acessar os nomes diretamente.

### 12 — Entregue ao professor
```bash
git init
git add README.md GUIA_PASSO_A_PASSO.md Vagrantfile requirements.yml ansible/ .gitignore
git commit -m 'Projeto DevOps IFPB'
```
Crie o repositório no GitHub e configure o remoto de sua conta. Envie os arquivos; não suba `.vagrant/` nem chaves privadas.

## O que cada playbook faz
- **comum.yml:** atualizações, relógio, grupo `ifpb`, usuárias, chaves, SSH e sudo.
- **arq.yml:** DHCP (reservas dos servidores), BIND9, três discos via LVM, ext4, NFS.
- **rede.yml:** cliente DHCP nas interfaces privadas das outras três máquinas.
- **servicos.yml:** MariaDB no `db`, Apache com página no `app`, Firefox/X11 no `cli`, autofs nas três.

## Ressalvas de validação
- O gatilho DHCP foi incluído no Vagrantfile conforme pedido, mas o sucesso deve ser conferido no VirtualBox.
- O host-only e o DNS do hospedeiro podem exigir configuração própria do Linux da equipe.
- O projeto foi montado para o laboratório; não foi executado em VMs reais aqui.
