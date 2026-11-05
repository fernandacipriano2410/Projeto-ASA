# Projeto 01 — DevOps com Vagrant e Ansible

**IFPB — Administração de Sistemas Abertos (2026.2)**  
**Professor:** Leonidas Lima  
**Aluna 1:** Fernanda Emily de Medeiros Cipriano — 20251380038  
**Aluna 2:** Júlia Evelim Virginio da Silva — 20251380004

## Objetivo
Criar quatro VMs Debian 12 (Bookworm) via Vagrant/VirtualBox e configurar os serviços usando Ansible, conforme o enunciado.

| VM | RAM | IP host-only | Serviços |
|---|---:|---|---|
| arq | 512 MB | 192.168.56.138 | DHCP, DNS, LVM, NFS |
| db | 512 MB | 192.168.56.104 (reserva DHCP) | MariaDB, autofs |
| app | 512 MB | 192.168.56.184 (reserva DHCP) | Apache, autofs |
| cli | 1024 MB | 192.168.56.50–100 (DHCP) | Firefox, xauth, X11, autofs |

**Observação:** `XY = 84` foi interpretado como os últimos algarismos das matrículas (8 e 4). Confirmar com o professor, pois o PDF não explica formalmente como combinar X e Y.

## Conteúdo
- `Vagrantfile` — infraestrutura.
- `ansible/inventory.ini` — hosts e conexões SSH via portas NAT.
- `ansible/vars.yml` — nomes, matrículas, IPs e MACs.
- `ansible/site.yml` — playbook principal, com arq antes das demais VMs.
- `ansible/tasks/` — playbooks pequenos por responsabilidade.
- `ansible/templates/` — arquivos de DHCP, DNS e Apache.
- `GUIA_PASSO_A_PASSO.md` — execução explicada e testes.

## Pré-requisitos
Máquina Linux com VirtualBox, Vagrant, Ansible, Python 3 e acesso à internet. Instale também as coleções Ansible necessárias:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Sequência resumida
1. Conferir/ajustar interface host-only `vboxnet0` no `Vagrantfile`, e garantir IP `192.168.56.1/24` no host.
2. Desligar o DHCP do VirtualBox **apenas na rede host-only**; não desativar NAT.
3. Criar duas chaves SSH locais (`chaves/fernanda` e `chaves/julia`).
4. `vagrant up arq --no-provision` e `vagrant up db app cli --no-provision`.
5. Conferir `vagrant ssh-config arq`, especialmente a chave privada e o forwarded port. Ajustar `ansible/inventory.ini` se os valores forem diferentes.
6. `cd ansible && ansible todos -m ping && ansible-playbook site.yml`.
7. Validar cada serviço com os comandos documentados no guia.

> **Importante:** o projeto foi preparado e verificado estaticamente, mas **não foi executado em VirtualBox** aqui. Versões do Vagrant/VirtualBox e configuração de host-only podem requerer pequenos ajustes no computador da equipe. Nunca execute as tarefas de LVM em discos com dados relevantes.

## Segurança
O acesso SSH usa chaves; o root e as senhas por SSH ficam bloqueados. As chaves privadas permanecem apenas no computador das alunas e estão ignoradas no Git. O acesso administrativo depende do grupo `ifpb` e de `sudo`.

## Funcionamento
O `arq` fornece DHCP com reservas por MAC, DNS interno e compartilhamento NFS em volume LVM ext4. As máquinas `db`, `app` e `cli` ativam DHCP na segunda interface, acessam `/dados/nfs` via `autofs` em `/var/nfs`, e recebem os serviços de sua função. O `app` hospeda uma página HTML com os dados do trabalho.

Veja o guia detalhado e as verificações em `GUIA_PASSO_A_PASSO.md`.
