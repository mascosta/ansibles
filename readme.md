# Repositorio público (caso alguem precise) de alguns estudos com ansible :D

- Saudações! Aqui vai alguns guias que podem ser usados ou até mesmo incrementados, de acordo com a necessidade, da galera que está entrando na área de DevOps ou até mesmo que já está mas ainda não teve o prazer de brincar com essa ferramenta show de bola, chamada Ansible.

- Apesar de não ser algo complexo ou avançado, segue abaixo alguns resultados de estudos com a ferramenta, desde a cópia simples da chave ssh pública para um destino "x" até o provisionamento de máquinas virtuais em um ambiente XEN like (XCP-NG)

## Observação1: [+ Alguns arquivos/diretórios de dados sensíveis foram removidos via gitignore, contudo, é só adaptar a realidade do ambiente que for aplicar. +]

**Alguns exemplos:**

- .ssh/
- roles/xen/vars/vars.yaml

## [+ Observação2:  O ambiente foi montado no diretório /opt/ansible +]


## 0 - Pré-configuração padrão do ambiente (adição das chaves, porque ninguem aqui quer usar o ```--ask-pass```)

Antes de mais tudo, como o objetivo é automatizar o máximo possível, foi criada uma role justamente para adição de chaves nos hosts que serão gerenciados:

**Arquivo de inventario:**

```ini

[local]
127.0.0.1 ansible_connection=local

```

**Arquivo de playbook:**

```yaml

---
- name: Copia de chave privada
  hosts: local
  tasks:
  - name: Execução com repetição em loop
    shell:
      cmd: ssh-copy-id -i /opt/ansible/.ssh/chave.pub root@"{{ item.ip }}" -o StrictHostKeyChecking=no
    loop:
      - { ip: '10.255.0.101' }
      - { ip: '10.255.0.102' }
      - { ip: '10.255.0.103' }
...

```
### Execução:

```bash
ansible-playbook -i $(pwd)/roles/addkey/inventario $(pwd)/roles/addkey/playbook.yaml
```

## 1 - Configuração padrão do ambiente (Agora começa a parte legal! :D)

Na role padrão foi feita uma configuração básica, sem o uso do when (pelo fato de as máquinas serem idênticas), e para sua execução, o bom e velho ```ansible-playbook```, claro, ajustando o arquivo de inventário à sua realidade.

**Arquivo de inventario:**

```ini

[teste]
10.255.0.10 ansible_connection=ssh ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_private_key_file=/opt/ansible/.ssh/chave ansible_ssh_user=root

```

[+ **Breve explicação nos parâmetros acima** +]

- ``` [teste] ```: Nome do grupo de hosts (targets) que será(ão) chamado(s) na playbook.
- ``` 10.255.0.10 ```: Muito bem, *#AcertôMizeravi*, é o endereço de rede, IPv4, do(s) target(s) que serão chamados na playbook.
- ``` ansible_connection=ssh ```: Tipo de conexão que será usada para realizar a tarefa, a mais comumente usada é **ssh**.
- ``` ansible_ssh_common_args='-o StrictHostKeyChecking=no' ```: A pessoa escrever um script de automação e ter que ficar apertando **y** toda vez que for conectar num host novo é osso, daí usa essa opção, pra não realizar a checagem de chave de host do destino.
- ``` ansible_ssh_private_key_file=/opt/ansible/.ssh/chave ```: Aponta para a chave privade que será usada pra validar a publica, previamente adicionada no host destino.
- ``` ansible_ssh_user=root ```: Trazendo à tona mais uma vez o espírito *xeroque romes*"*, sim, é o usuário de conexão usado.

**Arquivo de playbook** 

```yaml
---
- name: Atualização dos pacotes padrão
  hosts: all
  tasks:
    - name: Instalação do repositório EPEL
      yum:
        name: 
        - epel-release
        state: latest

    - name: Atualização do sistema
      yum:
        name: '*'
        state: latest
    - name: Adicionando pacotes padrão
      yum:
        name:
        - bash-completion
        - vim
        - nmap
        - tcpdump
        - git
        - wget
        - mtr
        - net-tools
        state: latest
...
```

### Execução:

```bash

ansible-playbook -i $(pwd)/roles/padrao/inventario $(pwd)/roles/padrao/playbook.yaml

```

## 2 - Provisionando máquinas virtuais no XCP-NG (Agora é onde o filho chora e a mãe não vê... D:)

Nessa parte, apos algumas horas de busca na internet e lida em fóruns, foram levantados os seguintes requisitos:

- É recomendável que se configure uma máquina toda ok, para ser criado um template a partir desta *pode ser rodada a playbook padrão acima que já é um grande adianto hehehe :grin:*.
- Depois desta criada, cria-se um snapshot.  *como eu tinha configurado ela antes com endereço de rede, antes de gerar o snapshot eu "desconfigurei", pra configurar depois das cópias ¯\_(ツ)_/¯*
- Não foi possível adição da configuração de rede através da playbook, necessitando melhoria. *Deve dar certo mesmo... Só não rolou no lab. :disappointed_relieved:*
- Para o provisionamento horizontal de nós a um cluster, acaba se tornando uma opção válida, apesar da questão do endereçamento. *Ou você pode ir na mão e subir de um por um também... :grimacing:* 
- Necessária a versão do ``` ansible >= 2.10 ```, por padrão algumas distribuições ainda trazem a 2.9 como padrão e a *collection* ``` community.general ```, para o módulo ``` community.general.xenserver_guest ``` usado na playbook. *E vai ficar quebrando a cabeça achando que quem escreveu esse guia ta viajando...:sleepy:*

```bash

ansible --version

```

```bash

ansible [core 2.12.7]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/user/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/user/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Jun 22 2022, 20:18:18) [GCC 9.4.0]
  jinja version = 2.10.1
  libyaml = True

```

```bash

ansible-galaxy collection install community.general

```


**Arquivo de inventario:**

```ini

[xenserver]
10.0.0.10 ansible_connection=ssh ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_private_key_file=/opt/ansible/.ssh/chave ansible_ssh_user=root

```

**Arquivo de playbook** 

```yaml
---
- name: Provisionando ambiente
  hosts: xenserver
  tasks:
  - name: Incluindo variáveis
    include_vars: vars/vars.yaml
  - name: Criando vm a partir do template
    community.general.xenserver_guest:
      hostname: 10.0.0.10
      username: "{{ user }}"
      password: "{{ pass }}"
      validate_certs: no
      name: "{{ item.vmname }}"
      state: poweredon
      template: ol8_default
      networks: 
      - name: vlanvms
      wait_for_ip_address: no
    loop:
      - { vmname: 'k8sm1' }
      - { vmname: 'k8sm2' }
      - { vmname: 'k8sm3' }
      - { vmname: 'k8sw1' }
      - { vmname: 'k8sw2' }
    register: deploy
...

```
[+ **Breve explicação nos parâmetros acima** +]

- ``` hostname: 10.0.0.10 ```: Endereço de IP ou FQDN do servidor onde será provisionado o ambiente.
- ``` username: "{{ user }}" ``` e ``` password: "{{ pass }}" ```: Valores existentes no arquivo de variaveis, apontado em ```include_vars: vars/vars.yaml```.
- ``` state: poweredon ```: Após a máquina criada, esse parâmetro define se esta estará ligada ``` poweredon ``` ou desligada ``` poweredoff ```.
- ``` name: "{{ item.vmname }}" ```: Sim, muito bem, é o nome da VM. Pode ser digitado a string desejada diretamente, mas como foi usado um laço de repetição para várias VMs, ficou no formato de variável.
- ``` template: ol8_default ```: Template usado para criação da máquina.
- ``` - name: vlanvms ```: Nome da rede, na configuração do XenServer/XCP-NG, que a VM será conectada.


### Execução:

```bash

ansible-playbook -i $(pwd)/roles/padrao/inventario $(pwd)/roles/padrao/playbook.yaml

```
