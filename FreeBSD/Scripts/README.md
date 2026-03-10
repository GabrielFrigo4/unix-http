# ⚙️ **Automação e Setup**

## 🛠️ **Scripts Disponíveis**
 Scripts desenvolvidos em **sh** (Shell padrão do FreeBSD) para automatizar a preparação do ambiente de desenvolvimento.

| Script | Local de Execução | Descrição |
| --- | --- | --- |
| **[`download.sh`](./download.sh)** | Host | Baixa a ISO mais recente do FreeBSD 15, verifica o Checksum (SHA256) e extrai o arquivo `.xz`. |
| **[`install.sh`](./install.sh)** | Host | Cria a VM no KVM via `virt-install` (4 vCPUs, 8GB RAM, 32GB Disk, UEFI). |
| **[`connect.sh`](./connect.sh)** | Host | Inicia a VM, abre o console gráfico e tenta conexão automática via SSH. |
| **[`setup.sh`](./setup.sh)** | Guest (VM) | **O coração do setup.** Configura drivers, Desktop (GNOME), Editores, Shell e Ferramentas. |
| **[`uninstall.sh`](./uninstall.sh)** | Host | Remove completamente a VM e seus discos do sistema. |


## 🚀 **Como Utilizar**
 1. Dê permissão de execução:
 ```sh
 chmod +x *.sh
 ```

 2. Execute conforme a necessidade:
 ```sh
 ./downlaod.sh
 ./install.sh
 ./connect.sh
 ./setup.sh
 ./uninstall.sh
 ```
