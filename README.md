# How to build an Ubuntu Server VM from scratch with Ansible

* [x] Vmware ESXi
* [x] vSphere
* [x] vCenter

Equivalente Linux do [ansible-vmware-windows](https://github.com/helviojunior/ansible-vmware-windows):
lá a instalação desatendida vai num floppy com `autounattend.xml`; aqui ela usa o
**autoinstall** do Ubuntu Server (subiquity + cloud-init NoCloud).

A ISO oficial é remasterizada a cada deploy: o seed (`user-data`/`meta-data`)
entra em `/nocloud` e o `grub.cfg` ganha `autoinstall ds=nocloud;s=/cdrom/nocloud/`
na linha do kernel — sem esse parâmetro o instalador para num prompt pedindo
confirmação antes de mexer no disco.

## Installing

```bash
ansible-galaxy collection install community.vmware vmware.vmware
```

A `vmware.vmware` já vem como dependência da `community.vmware`, mas instalar as
duas explicitamente evita ficar sem ela quando a `community.vmware` já estava
instalada de antes (o `ansible-galaxy` pula o que já existe).

O `xorriso` é obrigatório (extrai e regrava a ISO):

```bash
apt-get install -y xorriso      # Debian/Ubuntu
brew install xorriso            # macOS
```

Install pyvmomi lib from my repo, because I made a fix aboult Free ESXi licence limitation
```bash
python3 -m pip install git+https://github.com/helviojunior/pyvmomi
```

## Executing

```bash
cp vars_sample.yml vars.yml && $EDITOR vars.yml

ip="10.10.10.10"; # Vmware server IP
ansible-playbook -i $ip, deploy_ubuntu.yml -e @vars.yml
```

Numa estação limpa, acrescente `-e install_controller_deps=true` para instalar as
dependências python do controller (`jmespath`, `netaddr`, `requests`).

O IP da VM fica em `output/<vm_name>_ip.txt`.

## Variáveis

| Var | Descrição |
|-----|-----------|
| `vcenter_login` / `vcenter_password` | credenciais do ESXi/vCenter |
| `vcenter_datacenter` / `vcenter_datastore` | destino da VM (`ha-datacenter`/`datastore1` no ESXi standalone) |
| `vm_iso` / `vm_iso_url` | ISO do Ubuntu Server (live server). A ISO baixada fica em `iso_cache/` |
| `vm_iso_path` | ISO já resolvida por um cache externo — com ela o playbook não baixa nada |
| `vm_iso_sha256` | sha256 esperado da ISO; conferido no download e no reaproveitamento |
| `vm_name` / `vm_hostname` | nome da VM e hostname do sistema instalado |
| `vm_network` | portgroup |
| `vm_disk_gb` / `vm_memory_mb` / `vm_num_cpus` | dimensionamento |
| `vm_username` / `vm_password` | identidade criada pelo autoinstall |
| `vm_password_hash` | hash crypt(3)/SHA-512 da senha. Vazio = gerado de `vm_password` |
| `vm_ssh_public_key` | chave pública autorizada para `vm_username` **e root** |
| `vm_ssh_private_key_file` | opcional: liga o smoke test de SSH ao final |
| `install_controller_deps` | `true` instala as deps python do controller |

## O que a VM entrega

- usuário `vm_username` com a senha `vm_password` e **sudo sem senha**
  (`/etc/sudoers.d/90-vmmanager`);
- `vm_ssh_public_key` no `authorized_keys` do usuário e do root;
- `open-vm-tools` instalado — é por ele que o vCenter reporta o IP do guest;
- `/var/lib/cloud/instance/boot-finished` (criado pelo cloud-init do sistema
  instalado no primeiro boot; o `late-commands` cobre o caso de não haver
  cloud-init). É o marcador que automações costumam usar para saber que a
  máquina está pronta.

## Como o deploy funciona

| Play | O que faz |
|------|-----------|
| 1 (`all`) | `tasks/base.yml` (checagens + deps do controller) e `tasks/seed.yml` (baixa a ISO, extrai, injeta o seed, regrava a ISO de autoinstall) |
| 2 (`vSphere`) | `tasks/deploy.yml` — checa datacenter/datastore, sobe a ISO, cria a VM, liga, espera a instalação terminar, pega o IP e solta a mídia |
| 3 (`new_ubuntu`) | smoke test por SSH (só quando `vm_ssh_private_key_file` é informado) |

Detalhes que valem a pena saber:

- **Boot order `disk, cdrom`**: no primeiro boot o disco está vazio (sem MBR) e a
  BIOS cai no CD; depois da instalação ela passa a bootar do disco sem depender
  de o instalador ejetar a mídia.
- **Fim da instalação**: o instalador live também roda o `open-vm-tools`, então
  "tools disponível" não significa instalação concluída. O playbook espera o
  guest reportar o `vm_hostname` (no live o hostname é `ubuntu-server`).
- **Regravação da ISO**: as opções vêm do próprio xorriso
  (`-report_el_torito as_mkisofs` sobre a ISO original), o que preserva volume
  id, El Torito BIOS, partição EFI e system area. Referenciar o
  `boot_hybrid.img` pelo caminho extraído não funciona: a partir do 24.04 ele
  não existe como arquivo dentro da ISO, mora na system area.
- **Volume id**: a ISO regravada preserva o rótulo da original — o `casper`
  procura o sistema live pelo rótulo do CD.
- **Nomes de arquivo**: derivam do slug do `vm_hostname` (`Teste 001` →
  `teste-001`), então espaço e acento no nome da VM não vazam para os comandos
  do xorriso nem para o caminho no datastore.
- **Espaço em disco** no controller: durante o deploy convivem a ISO original em
  cache, a árvore extraída e a ISO gerada (~3 GB cada, nas ISOs recentes). Ao
  final o playbook apaga `build/<slug>/` e a ISO de autoinstall — só a ISO
  original permanece em cache, para os próximos deploys não a baixarem de novo.
  Rode com `-e keep_build_files=true` para manter tudo e depurar o seed.
- **Cache da ISO original**: `iso_cache/` por padrão, dentro do repo. Quando quem
  chama já tem um cache próprio (o vm_manager tem um global, compartilhado por
  todos os deploys), basta passar `vm_iso_path` apontando o arquivo — o playbook
  usa e não baixa nada.
- **Datastore**: a ISO customizada é apagada do datastore assim que a mídia é
  solta da VM; as ISOs stock que estiverem lá não são tocadas.

## Common error

```
objc[7453]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called.
objc[7453]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called. We cannot safely call it or ignore it in the fork() child process. Crashing instead. Set a breakpoint on objc_initializeAfterForkError to debug.
```

To solve this error run
```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

## Inspiration
- [ansible-vmware-windows](https://github.com/helviojunior/ansible-vmware-windows)
- [Automated Server Installer (autoinstall)](https://canonical-subiquity.readthedocs-hosted.com/en/latest/)
