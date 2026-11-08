# Projeto 01 — DevOps com Vagrant e Ansible

**Instituto Federal da Paraíba (IFPB) — Campus João Pessoa**  
**Disciplina:** Administração de Sistemas Abertos — 2026.2  
**Professor:** Leonidas Lima  
**Integrantes:**

| Integrante | Matrícula |
|---|---|
| Fernanda Emily de Medeiros Cipriano | 20251380038 |
| Júlia Evelim Virginio da Silva | 20251380004 |

## 1. Visão geral

Este projeto implementa uma infraestrutura virtual com **quatro máquinas Debian 12 (Bookworm)**, provisionadas por **Vagrant/VirtualBox** e configuradas por **Ansible**, de acordo com os requisitos da disciplina. A proposta aplica **Infraestrutura como Código (IaC)** para automatizar a criação de servidores e a configuração de rede, usuários e serviços.

A solução reúne DHCP e DNS centralizados, armazenamento LVM, compartilhamento NFS, banco MariaDB, servidor Apache e uma máquina cliente com encaminhamento X11.

## 2. Arquitetura

```text
                         Hospedeiro Linux
                      Vagrant + VirtualBox
                       Ansible (via SSH)
                              |
                 Rede host-only 192.168.56.0/24
                              |
            +-----------------+-------------------+
            |                 |                   |
  arq.fernanda.julia.devops   |                   |
       192.168.56.138        |                   |
      DHCP, DNS, LVM, NFS    |                   |
            |                |                   |
            +--------+-------+------------+------+
                     |                    |
       db.fernanda.julia.devops    app.fernanda.julia.devops
            192.168.56.104              192.168.56.184
           MariaDB + autofs            Apache + autofs
                     |
              cli.fernanda.julia.devops
                  DHCP (50–100)
               Firefox + X11 + autofs
```

> O diagrama representa a rede lógica; as quatro VMs compartilham a rede privada. Cada VM também possui um adaptador NAT utilizado para acesso inicial e instalação de pacotes.

| VM | Endereço privado | RAM | Função |
|---|---|---:|---|
| `arq` | `192.168.56.138` (fixo) | 512 MB | DHCP, DNS, LVM e NFS |
| `db` | `192.168.56.104` (reserva por MAC) | 512 MB | MariaDB e autofs |
| `app` | `192.168.56.184` (reserva por MAC) | 512 MB | Apache2 e autofs |
| `cli` | DHCP, faixa `192.168.56.50–100` | 1024 MB | Firefox ESR, X11 e autofs |

**Domínio:** `fernanda.julia.devops`  
**Rede privada:** `192.168.56.0/24`  
**Interface host-only do hospedeiro:** `vboxnet0` (`192.168.56.1`)  
**Servidor DHCP e DNS:** `192.168.56.138`  
**Gateway informado pelo DHCP:** `192.168.56.1`

O IP do APP adota `XY = 84` (último algarismo da matrícula da primeira integrante seguido pelo da segunda); essa interpretação do enunciado deve ser confirmada com o professor.

### Recursos definidos no Vagrant

- Box `debian/bookworm64` e provider VirtualBox.
- *Linked clones* habilitados e verificação de Guest Additions desabilitada.
- Geração automática de novas chaves SSH do Vagrant desativada.
- Três discos adicionais de 10 GB no ARQ.
- Gatilho para desativar o DHCP do VirtualBox na rede host-only.
- Portas encaminhadas para SSH no hospedeiro: `2221` (ARQ), `2232` (DB), `2223` (APP) e `2224` (CLI). **A porta do DB foi ajustada para evitar um conflito no ambiente de testes.**

## 3. Serviços implementados

### Configuração comum às quatro VMs

- Atualização de pacotes Debian; Chrony utilizando `pool.ntp.br` e fuso `America/Recife`.
- Grupo `ifpb` e contas `fernanda` e `julia`.
- Instalação de cliente NFS e utilitários de administração.
- SSH por chave pública, sem autenticação por senha e sem login remoto de `root`.
- Restrição de acesso SSH aos grupos `vagrant` e `ifpb`.
- Banner: `Acesso apenas para pessoas com autorização expressa!!!`.
- Privilégios administrativos via `sudo` para membros de `ifpb`. Neste laboratório, foi configurado `NOPASSWD`; **não é uma política recomendada para produção**.

### ARQ — Infraestrutura e arquivos

| Serviço | Configuração |
|---|---|
| DHCP (`isc-dhcp-server`) | Autoritativo; faixa `.50–.100`; *lease* padrão 180 s e máximo 3600 s; reservas para DB e APP |
| DNS (BIND9) | Zona direta `fernanda.julia.devops` e reversa `56.168.192.in-addr.arpa`; encaminhadores `1.1.1.1` e `8.8.8.8` |
| LVM | Três discos de 10 GB → VG `dados` → LV `ifpb` de 15 GiB, `ext4`, montado em `/dados` |
| NFS | Exportação `/dados/nfs` para `192.168.56.0/24`, opções `rw,sync,all_squash` e conta `nfs-ifpb` sem shell |

O ARQ utiliza `127.0.0.1` como DNS local, com configuração no `dhclient` para manter a escolha em renovações DHCP. A resolução foi testada novamente após reinício da VM.

### DB — Banco de dados

Instalação do **MariaDB Server** e montagem automática do compartilhamento NFS do ARQ em `/var/nfs`, utilizando **autofs**.

### APP — Aplicação web

Instalação do **Apache2**, substituição da página inicial por um `index.html` com informações do projeto e das integrantes, além de montagem NFS em `/var/nfs` por **autofs**.

**Endereço de acesso no laboratório:** `http://192.168.56.184`.

### CLI — Estação cliente

Instalação de **Firefox ESR** e **xauth**, configuração do SSH para encaminhamento gráfico **X11** e montagem automática NFS via **autofs**.

## 4. Estrutura do repositório

```text
Projeto-ASA/
├── Vagrantfile
├── README.md
├── GUIA_PASSO_A_PASSO.md
├── requirements.yml
├── .gitignore
├── chaves/
│   ├── fernanda.pub
│   └── julia.pub
└── ansible/
    ├── ansible.cfg
    ├── inventory.ini
    ├── vars.yml
    ├── site.yml
    ├── handlers.yml
    ├── tasks/
    │   ├── comum.yml
    │   ├── rede.yml
    │   ├── arq.yml
    │   └── servicos.yml
    └── templates/
        ├── dhcpd.conf.j2
        ├── named.conf.options.j2
        ├── named.conf.local.j2
        ├── db.forward.j2
        ├── db.reverse.j2
        └── index.html.j2
```

As **chaves privadas** e a pasta `.vagrant/` são locais e ficam fora do Git. Os playbooks temporários utilizados no diagnóstico também não são necessários para a entrega.

## 5. Requisitos e preparação

Ambiente de testes utilizado:

| Ferramenta | Versão |
|---|---|
| Vagrant | 2.4.9 |
| VirtualBox | 7.2.6 (Ubuntu) |
| Ansible Core | 2.20.1 |
| Python no hospedeiro | 3.14.4 |
| Ansible `community.general` | 12.1.0 |
| Ansible `ansible.posix` | 2.1.0 |

É necessário um hospedeiro Linux com virtualização habilitada, acesso à internet e recursos para quatro VMs. A interface host-only deve estar disponível como `vboxnet0` (ou ter seu nome adaptado no Vagrantfile).

### 5.1. Preparar a rede host-only

```bash
VBoxManage list hostonlyifs
VBoxManage list dhcpservers
VBoxManage dhcpserver modify --interface=vboxnet0 --disable
```

A interface `vboxnet0` deve possuir `192.168.56.1/24`. O comando desativa o DHCP **somente da rede host-only**; o DHCP da rede NAT do VirtualBox deve permanecer disponível. Atenção a outras VMs que utilizem a mesma rede.

### 5.2. Chaves SSH

O projeto utiliza as chaves públicas em `chaves/fernanda.pub` e `chaves/julia.pub`. As respectivas chaves privadas devem permanecer com suas proprietárias. Para gerar um novo par em outro ambiente:

```bash
mkdir -p chaves
ssh-keygen -t ed25519 -f chaves/fernanda -C fernanda@ifpb
ssh-keygen -t ed25519 -f chaves/julia -C julia@ifpb
```

**Atenção:** gerar novas chaves substitui a identidade correspondente; as chaves públicas utilizadas pelo Ansible devem corresponder às privadas utilizadas no acesso. Não sobrescreva pares existentes sem necessidade.

### 5.3. Instalar coleções Ansible

```bash
ansible-galaxy collection install -r requirements.yml
```

### 5.4. Criar as VMs

```bash
vagrant validate
vagrant up arq --no-provision
vagrant up db app cli --no-provision
vagrant status
```

Em caso de portas SSH ocupadas, ajustar de forma correspondente o `Vagrantfile` **e** `ansible/inventory.ini`. O inventário utiliza as portas encaminhadas no `127.0.0.1`; a chave inicial do usuário `vagrant` é a chave utilizada pelo Vagrant com `config.ssh.insert_key = false`.

### 5.5. Executar o provisionamento

```bash
cd ansible
ansible todos -m ping
ansible-playbook --syntax-check site.yml
ansible-playbook -i inventory.ini site.yml --limit arq
ansible-playbook -i inventory.ini site.yml --limit demais
```

O ARQ deve ser configurado antes das demais máquinas, pois fornece DHCP, DNS e NFS. **As tarefas de LVM criam e formatam volumes; executá-las somente nas VMs de laboratório e com os discos adicionais identificados corretamente.**

## 6. Testes executados no laboratório

Os testes a seguir foram realizados durante a implantação em **7 de novembro de 2026**. Os resultados descrevem o ambiente testado; não substituem uma validação de instalação limpa em outro computador.

| Componente | Evidência observada | Resultado |
|---|---|---|
| Vagrant | `vagrant status`: quatro máquinas `running` | Aprovado |
| Ansible | `ansible ... -m ping`: resposta `pong` nos hosts testados | Aprovado |
| DHCP | DB recebeu `192.168.56.104/24` em `eth1`, com endereço dinâmico e MAC reservado | Aprovado |
| DNS direto | `dig` resolveu ARQ (`.138`), DB (`.104`) e APP (`.184`) | Aprovado |
| DNS reverso | `dig -x 192.168.56.184` retornou `app.fernanda.julia.devops.` | Aprovado |
| LVM | `vgs`: VG `dados` com três PVs; `lvs`: LV `ifpb` de 15 GiB | Aprovado |
| Montagem ext4 | `findmnt /dados`: `/dev/mapper/dados-ifpb`, `ext4` | Aprovado |
| NFS | `exportfs -v`: exportação de `/dados/nfs` para `192.168.56.0/24` | Aprovado |
| MariaDB | Serviço retornou `active` no DB | Aprovado |
| autofs no DB | `findmnt /var/nfs`: montagem NFSv4.2 de `192.168.56.138:/dados/nfs` | Aprovado |
| Apache | Página acessada pelo navegador em `http://192.168.56.184` | Aprovado |
| SSH | Login por chave de Fernanda e Júlia no ARQ; banner e grupo `ifpb` confirmados | Aprovado |
| Políticas SSH | `sshd -T`: senha desativada, `PermitRootLogin no`, grupos permitidos | Aprovado |
| X11 | Firefox iniciado na VM CLI e exibido na sessão gráfica do hospedeiro | Aprovado |
| DNS do hospedeiro | `getent ahostsv4` retornou os três endereços internos | Aprovado |
| DNS após reinício do ARQ | `resolv.conf` manteve `127.0.0.1`; BIND9 e DHCP ativos | Aprovado |
| Sudo | Regra `NOPASSWD` incluída e teste de uso informado como bem-sucedido | Aprovado no ambiente testado |

### Comandos úteis de verificação

Executar na pasta `ansible/`:

```bash
ansible todos -m ping
ansible arq -b -m command -a "systemctl is-active bind9"
ansible arq -b -m command -a "systemctl is-active isc-dhcp-server"
ansible arq -b -m command -a "lvs"
ansible arq -b -m command -a "vgs"
ansible arq -b -m command -a "exportfs -v"
ansible db -b -m command -a "systemctl is-active mariadb"
ansible db -b -m shell -a "ls /var/nfs && findmnt /var/nfs"
ansible app -b -m command -a "systemctl is-active apache2"
```

**Nota:** a classificação `CHANGED` exibida pelo módulo ad hoc `command` não significa necessariamente que o comando de consulta modificou o sistema.

## 7. Resolução DNS no hospedeiro

O Ubuntu hospedeiro usa `systemd-resolved`. Para consultas do domínio interno utilizarem o DNS da VM ARQ, sem substituir os servidores DNS habituais da internet:

```bash
sudo resolvectl dns vboxnet0 192.168.56.138
sudo resolvectl domain vboxnet0 '~fernanda.julia.devops'
sudo resolvectl flush-caches
getent ahostsv4 arq.fernanda.julia.devops
getent ahostsv4 db.fernanda.julia.devops
getent ahostsv4 app.fernanda.julia.devops
```

No ambiente testado, os três nomes foram resolvidos corretamente. **A configuração via `resolvectl` é temporária** e pode precisar ser reaplicada após reinicialização ou recriação da interface. Para persistência, é necessário integrar as configurações ao gerenciador de rede utilizado pelo hospedeiro; isso não é feito pelos playbooks das VMs.

## 8. Segurança e observações

- Não publicar chaves privadas, credenciais, dados pessoais extras nem o diretório `.vagrant/`.
- As chaves **públicas** `.pub` integram o repositório para criação das contas SSH.
- O sudo sem senha foi adotado apenas para o laboratório; ambientes reais devem adotar uma política mais restritiva.
- O encaminhamento X11 confiável (`ssh -Y`) foi utilizado nos testes da VM CLI; exige cautela fora de um laboratório isolado.
- O projeto foi testado em VMs existentes e corrigido durante a implantação. **Uma reprodução completa a partir de VMs zeradas ainda não foi demonstrada**, portanto podem ser necessários ajustes relativos ao ambiente do avaliador.
- O endereço IP `192.168.56.184` corresponde à interpretação utilizada para `XY` no enunciado e deve ser confirmado com o professor.

## 9. Conclusão

A equipe implantou quatro máquinas virtuais Debian utilizando Vagrant e realizou o provisionamento de seus serviços com Ansible. Os testes de laboratório demonstraram integração entre DHCP, DNS, LVM, NFS, MariaDB, Apache, autenticação SSH por chaves e X11, além de resolução de nomes no hospedeiro.

O projeto reúne os arquivos de infraestrutura e automação necessários à avaliação e documenta o processo de implantação e os resultados obtidos no ambiente testado.
