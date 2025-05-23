---
title: Creazione Automatica di Template - Packer - Ansible - VMware vSphere
author: Antoine Le Morvan
contributors: Steven Spencer, Ryan Johnson, Pedro Garcia, Ganna Zhyrnova
---

# Creazione automatica di template con Packer e distribuzione con Ansible in un ambiente VMware vSphere

**Conoscenza**: :star: :star: :star  
**Complessità**: :star: :star: :star: :star:

**Tempo di lettura**: 30 minuti

## Prerequisiti, presupposti e note generali

- È disponibile un ambiente vSphere e un utente con accesso garantito
- Un server web interno per archiviare i file
- Accesso web ai repository Rocky Linux
- Una ISO di Rocky Linux
- Un ambiente Ansible disponibile.
- Si presuppone che abbiate una certa conoscenza di ogni prodotto menzionato. In caso contrario, prima di iniziare, è necessario consultare la documentazione.
- Vagrant **non è** in uso qui. È stato sottolineato che con Vagrant sarebbe stata fornita una chiave SSH non autofirmata. Se volete approfondire l'argomento, potete farlo, ma non è oggetto di questo documento.

## Introduzione

Questo documento copre la creazione di modelli di macchine virtuali vSphere con Packer e come distribuire l'artefatto come nuove macchine virtuali con Ansible.

## Possibili aggiustamenti

Naturalmente, potete adattare questo how-to per altri hypervisor.

Sebbene qui si utilizzi l'immagine ISO minima, si può scegliere di utilizzare l'immagine del DVD (molto più grande e forse troppo grande) o l'immagine di avvio (molto più piccola e forse troppo piccola). La scelta spetta a voi. La scelta spetta a voi. In particolare, influisce sulla larghezza di banda necessaria per l'installazione e quindi sul tempo di provisioning. Discuteremo in seguito l'impatto di questa scelta predefinita e come porvi rimedio.

Si può anche scegliere di non convertire la macchina virtuale in un modello. In questo caso si utilizzerà Packer per distribuire ogni nuova macchina virtuale, il che è ancora abbastanza fattibile. Un'installazione a partire da 0 richiede meno di 10 minuti senza interazione umana.

## Packer

### Introduzione a Packer

Packer è uno strumento di imaging di macchine virtuali open-source, rilasciato sotto licenza MPL 2.0 e creato da HashiCorp. Vi aiuterà ad automatizzare il processo di creazione di immagini di macchine virtuali con sistemi operativi preconfigurati e software installato da un'unica fonte di configurazione in ambienti virtualizzati sia cloud che on-prem.

Con Packer è possibile creare immagini da utilizzare sulle seguenti piattaforme:

- Il [sito web di Packer](https://www.packer.io/).
- [Azure](https://azure.microsoft.com).
- [GCP](https://cloud.google.com).
- [DigitalOcean](https://www.digitalocean.com).
- [OpenStack](https://www.openstack.org).
- [VirtualBox](https://www.virtualbox.org/).
- [VMware](https://www.vmware.com).

Per ulteriori informazioni, potete consultare queste risorse:

- Il [sito web di Packer](https://www.packer.io/)
- [Documentazione Packer](https://www.packer.io/docs)
- La [documentazione](https://www.packer.io/docs/builders/vsphere/vsphere-iso) del costruttore `vsphere-iso`

### Installazione di Packer

Esistono due modi per installare Packer nel sistema Rocky Linux.

#### Installazione di Packer dal repo di HashiCorp

HashiCorp mantiene e firma pacchetti per diverse distribuzioni Linux. Per installare Packer nel nostro sistema Rocky Linux, seguite i passi successivi:

1. Installare dnf-config-manager:

    ```bash
    sudo dnf install -y dnf-plugins-core
    ```

2. Aggiungere il repository HashiCorp ai repository disponibili nel nostro sistema Rocky Linux:

    ```bash
    sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
    ```

3. Installare Packer:

    ```bash
    sudo dnf -y install packer
    ```

#### Scaricare e installare dal sito web di Packer

Si può iniziare scaricando i binari per la propria piattaforma con [Packer downloads](https://www.packer.io/downloads).

1. Nella pagina di download, copiate il link di download nella sezione Linux Binary Download che corrisponde all'architettura del vostro sistema.

2. Da una shell o da un terminale, scaricatelo utilizzando lo strumento `wget`:

    ```bash
    wget https://releases.hashicorp.com/packer/1.8.3/packer_1.8.3_linux_amd64.zip
    ```

    Verrà scaricato un file .zip.

3. Per decomprimere l'archivio scaricato, eseguire il seguente comando nella shell:

    ```bash
    unzip packer_1.8.3_linux_amd64.zip
    ```

    !!! tip "Suggerimento"

     Se si ottiene un errore e non si ha l'applicazione unzip installata sul sistema, è possibile installarla eseguendo questo comando ``sudo dnf install unzip``.

4. Spostare l'applicazione Packer nella cartella bin:

    ```bash
    sudo mv packer /usr/local/bin/
    ```

#### Verifica della corretta installazione di Packer

Se tutti i passaggi delle procedure precedenti sono stati completati correttamente, possiamo procedere alla verifica dell'installazione di Packer sul nostro sistema.

Per verificare che Packer sia stato installato correttamente, eseguite il comando `packer` e otterrete il risultato mostrato di seguito:

```bash
$ packer 
Usage: packer [--version] [--help] <command> [<args>]

I comandi disponibili sono:
    build           creare le immagini dal modello
    console         crea una console per testare l'interpolazione delle variabili
    fix             corregge i modelli delle vecchie versioni di packer
    fmt             riscrive i file di configurazione HCL2 nel formato canonico
    hcl2_upgrade    trasformare un modello JSON in una configurazione HCL2
    init            installare i plugin mancanti o aggiornarli
    inspect         vedere i componenti di un modello
    plugins         interagire con i plugin e il catalogo di Packer
    validate        controllare che un modello sia valido
    version         stampa la versione di Packer
```

### Creazione di modelli con Packer

!!! note "Nota"

    Negli esempi che seguono, si presuppone che l'utente si trovi su un sistema Linux.

Poiché ci collegheremo a un server VMware vCenter per inviare i nostri comandi tramite Packer, dobbiamo memorizzare le nostre credenziali al di fuori dei file di configurazione che creeremo successivamente.

Creiamo un file nascosto con le nostre credenziali nella nostra home directory. Si tratta di un file json:

```bash
$ vim .vsphere-secrets.json {
    "vcenter_username": "rockstar",
    "vcenter_password": "mysecurepassword"
  }
```

Queste credenziali devono garantire l'accesso all'ambiente vSphere.

Creiamo un file json (in futuro, il formato di questo file cambierà in HCL):

```json
{
  "variables": {
    "version": "0.0.X",
    "HTTP_IP": "fileserver.rockylinux.lan",
    "HTTP_PATH": "/packer/rockylinux/8/ks.cfg"
  },
  "sensitive-variables": ["vcenter_password"],
  "provisioners": [
    {
      "type": "shell",
      "expect_disconnect": true,
      "execute_command": "bash '{{.Path}}'",
      "script": "{{template_dir}}/scripts/requirements.sh"
    }
  ],
  "builders": [
    {
      "type": "vsphere-iso",
      "CPUs": 2,
      "CPU_hot_plug": true,
      "RAM": 2048,
      "RAM_hot_plug": true,
      "disk_controller_type": "pvscsi",
      "guest_os_type": "centos8_64Guest",
      "iso_paths": [
        "[datasyno-contentlibrary-mylib] contentlib-a86ad29a-a43b-4717-97e6-593b8358801b/3a381c78-b9df-45a6-82e1-3c07c8187dbe/Rocky-8.4-x86_64-minimal_72cc0cc6-9d0f-4c68-9bcd-06385a506a5d.iso"
      ],
      "network_adapters": [
        {
          "network_card": "vmxnet3",
          "network": "net_infra"
        }
      ],
      "storage": [
        {
          "disk_size": 40000,
          "disk_thin_provisioned": true
        }
      ],
      "boot_command": [
      "<up><tab> text ip=192.168.1.11::192.168.1.254:255.255.255.0:template:ens192:none nameserver=192.168.1.254 inst.ks=http://{{ user `HTTP_IP` }}/{{ user `HTTP_PATH` }}<enter><wait><enter>"
      ],
      "ssh_password": "mysecurepassword",
      "ssh_username": "root",
      "shutdown_command": "/sbin/halt -h -p",
      "insecure_connection": "true",
      "username": "{{ user `vcenter_username` }}",
      "password": "{{ user `vcenter_password` }}",
      "vcenter_server": "vsphere.rockylinux.lan",
      "datacenter": "DC_NAME",
      "datastore": "DS_NAME",
      "vm_name": "template-rockylinux8-{{ user `version` }}",
      "folder": "Templates/RockyLinux",
      "cluster": "CLUSTER_NAME",
      "host": "esx1.rockylinux.lan",
      "notes": "Template RockyLinux version {{ user `version` }}",
      "convert_to_template": true,
      "create_snapshot": false
    }
  ]
}
```

In seguito, descriveremo ogni sezione di questo file.

### Sezione Variabili

In una prima fase, dichiariamo le variabili, soprattutto per motivi di leggibilità:

```json
"variables": {
  "version": "0.0.X",
  "HTTP_IP": "fileserver.rockylinux.lan",
  "HTTP_PATH": "/packer/rockylinux/8/ks.cfg"
},
```

Useremo la variabile `version` più tardi nel nome del modello che creeremo. È possibile incrementare facilmente questo valore in base alle proprie esigenze.

È inoltre necessario che la macchina virtuale in fase di avvio acceda a un file `ks.cfg` (Kickstart).

Un file Kickstart contiene le risposte alle domande poste durante il processo di installazione. Questo file passa tutto il suo contenuto ad Anaconda (il processo di installazione), il che permette di automatizzare completamente la creazione del modello.

L'autore preferisce memorizzare il file `ks.cfg` in un server web interno accessibile dal suo modello, ma esistono altre possibilità che si può scegliere di utilizzare al suo posto.

Ad esempio, nel nostro laboratorio il file `ks.cfg` è accessibile dalla macchina virtuale a questo URL: <http://fileserver.rockylinux.lan/packer/rockylinux/8/ks.cfg>. Per utilizzare questo metodo è necessario impostare qualcosa di simile.

Poiché vogliamo mantenere la password privata, essa viene dichiarata come variabile sensibile. Esempio:

```json
"sensitive-variables": ["vcenter_password"],
```

### Sezione Provisioners

La parte successiva è interessante e sarà trattata più avanti fornendo lo script `requirements.sh`:

```json
"provisioners": [
  {
    "type": "shell",
    "expect_disconnect": true,
    "execute_command": "bash '{{.Path}}'",
    "script": "{{template_dir}}/scripts/requirements.sh"
  }
],
```

Al termine dell'installazione, la macchina virtuale si riavvia. Non appena Packer rileva un indirizzo IP (grazie a VMware Tools), copia il file `requirements.sh` e lo esegue. È una bella funzione per pulire la macchina virtuale dopo il processo di installazione (rimuovere le chiavi SSH, pulire la cronologia, ecc.) e installare qualche pacchetto extra.

### La sezione dei builders

È possibile dichiarare uno o più costruttori per puntare a qualcosa di diverso dal proprio ambiente vSphere (magari un modello Vagrant).

Ma qui stiamo usando `vsphere-iso` builder:

```json
"builders": [
  {
    "type": "vsphere-iso",
```

Questo costruttore ci permette di configurare l'hardware necessario:

```json
"CPUs": 2,
"CPU_hot_plug": true,
"RAM": 2048,
"RAM_hot_plug": true,
"disk_controller_type": "pvscsi",
"guest_os_type": "centos8_64Guest",
"network_adapters": [
  {
    "network_card": "vmxnet3",
    "network": "net_infra"
  }
],
"storage": [
  {
    "disk_size": 40000,
    "disk_thin_provisioned": true
  }
],
```

!!! note "Nota"

    Non dimenticherete mai più di includere CPU_hot_plug, perché ora è automatico!

Si possono fare cose più interessanti con il disco, la CPU e così via. Se si desidera effettuare altre regolazioni, consultare la documentazione.

Per avviare l'installazione, è necessaria un'immagine ISO di Rocky Linux. Ecco un esempio di come utilizzare un'immagine situata in una libreria di contenuti vSphere. Naturalmente è possibile memorizzare l'ISO altrove. Nel caso di una libreria di contenuti vSphere, è necessario ottenere il percorso completo del file ISO sul server che ospita la libreria di contenuti. In questo caso si tratta di Synology, quindi direttamente su DSM explorer.

```json
"iso_paths": [
  "[datasyno-contentlibrary-mylib] contentlib-a86ad29a-a43b-4717-97e6-593b8358801b/3a381c78-b9df-45a6-82e1-3c07c8187dbe/Rocky-8.4-x86_64-minimal_72cc0cc6-9d0f-4c68-9bcd-06385a506a5d.iso"
],
```

Quindi è necessario fornire il comando completo da inserire durante il processo di installazione: configurazione dell'IP e trasmissione del percorso del file di risposta di Kickstart.

!!! note "Nota"

    Questo esempio prende in considerazione il caso più complesso: l'utilizzo di un IP statico. Se si dispone di un server DHCP, il processo sarà molto più semplice.

Questa è la parte più divertente della procedura: Sono sicuro che andrete ad ammirare la console VMware durante la generazione, solo per vedere l'inserimento automatico dei comandi durante l'avvio.

```json
"boot_command": [
"<up><tab> text ip=192.168.1.11::192.168.1.254:255.255.255.0:template:ens192:none nameserver=192.168.1.254 inst.ks=http://{{ user `HTTP_IP` }}/{{ user `HTTP_PATH` }}<enter><wait><enter>"
],
```

Dopo il primo riavvio, Packer si connetterà al server tramite SSH. Si può usare l'utente root o un altro utente con diritti sudo, ma in ogni caso questo utente deve corrispondere all'utente definito nel file ks.cfg.

```json
"ssh_password": "mysecurepassword",
"ssh_username": "root",
```

Al termine del processo, la macchina virtuale deve essere arrestata. È un po' più complicato con un utente non root, ma è ben documentato:

```json
"shutdown_command": "/sbin/halt -h -p",
```

Successivamente, ci occupiamo della configurazione di vSphere. Le uniche cose degne di nota sono l'uso delle variabili definite all'inizio del documento nella nostra home directory e l'opzione `insecure_connection`, perché vSphere utilizza un certificato autofirmato (si veda la nota in Assumptions all'inizio di questo documento):

```json
"insecure_connection": "true",
"username": "{{ user `vcenter_username` }}",
"password": "{{ user `vcenter_password` }}",
"vcenter_server": "vsphere.rockylinux.lan",
"datacenter": "DC_NAME",
"datastore": "DS_NAME",
"vm_name": "template-rockylinux8-{{ user `version` }}",
"folder": "Templates/RockyLinux",
"cluster": "CLUSTER_NAME",
"host": "esx1.rockylinux.lan",
"notes": "Template RockyLinux version {{ user `version` }}"
```

E infine, chiederemo a vSphere di convertire la nostra VM arrestata in un template.

In questa fase, si può anche decidere di utilizzare la macchina virtuale così com'è (senza convertirla in un modello). In questo caso, si può decidere di fare un'istantanea:

```json
"convert_to_template": true,
"create_snapshot": false,
```

## Il file ks.cfg

Come già detto, è necessario fornire un file di risposta di kickstart che verrà utilizzato da Anaconda.

Ecco un esempio di questo file:

```bash
# Use CD-ROM installation media
repo --name="AppStream" --baseurl="http://download.rockylinux.org/pub/rocky/8.4/AppStream/x86_64/os/"
cdrom
# Use text install
text
# Don't run the Setup Agent on first boot
firstboot --disabled
eula --agreed
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network --bootproto=static --device=ens192 --gateway=192.168.1.254 --ip=192.168.1.11 --nameserver=192.168.1.254,4.4.4.4 --netmask=255.255.255.0 --onboot=on --ipv6=auto --activate

# Root password
rootpw mysecurepassword

# System services
selinux --permissive
firewall --enabled
services --enabled="NetworkManager,sshd,chronyd"
# System timezone
timezone Europe/Paris --isUtc
# System booloader configuration
bootloader --location=mbr --boot-drive=sda
# Partition clearing information
clearpart --all --initlabel --drives=sda
# Disk partitionning information
part /boot --fstype="xfs" --ondisk=sda --size=512
part pv.01 --fstype="lvmpv" --ondisk=sda --grow
volgroup vg_root --pesize=4096 pv.01
logvol /home --fstype="xfs" --size=5120 --name=lv_home --vgname=vg_root
logvol /var --fstype="xfs" --size=10240 --name=lv_var --vgname=vg_root
logvol / --fstype="xfs" --size=10240 --name=lv_root --vgname=vg_root
logvol swap --fstype="swap" --size=4092 --name=lv_swap --vgname=vg_root

skipx

reboot

%packages --ignoremissing --excludedocs
openssh-clients
curl
dnf-utils
drpm
net-tools
open-vm-tools
perl
perl-File-Temp
sudo
vim
wget
python3

# unnecessary firmware
-aic94xx-firmware
-atmel-firmware
-b43-openfwwf
-bfa-firmware
-ipw2100-firmware
-ipw2200-firmware
-ivtv-firmware
-iwl*-firmware
-libertas-usb8388-firmware
-ql*-firmware
-rt61pci-firmware
-rt73usb-firmware
-xorg-x11-drv-ati-firmware
-zd1211-firmware
-cockpit
-quota
-alsa-*
-fprintd-pam
-intltool
-microcode_ctl
%end

%addon com_redhat_kdump --disable
%end

%post

# Manage Ansible access
groupadd -g 1001 ansible
useradd -m -g 1001 -u 1001 ansible
mkdir /home/ansible/.ssh
echo -e "<---- PAST YOUR PUBKEY HERE ---->" >  /home/ansible/.ssh/authorized_keys
chown -R ansible:ansible /home/ansible/.ssh
chmod 700 /home/ansible/.ssh
chmod 600 /home/ansible/.ssh/authorized_keys
echo "ansible ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible

systemctl enable vmtoolsd
systemctl start vmtoolsd

%end
```

Poiché si è scelto di usare l'iso minimale, invece del Boot o del DVD, non tutti i pacchetti di installazione richiesti saranno disponibili.

Poiché Packer si affida a VMware Tools per rilevare la fine dell'installazione e il pacchetto `open-vm-tools` è disponibile solo nei repo di AppStream, dobbiamo specificare al processo di installazione che vogliamo usare come sorgente sia il CD-ROM che questo repo remoto:

!!! note "Nota"

    Se non si ha accesso ai repo esterni, si può usare un mirror del repo, un proxy squid o il DVD.

```bash
# Use CD-ROM installation media
repo --name="AppStream" --baseurl="http://download.rockylinux.org/pub/rocky/8.4/AppStream/x86_64/os/"
cdrom
```

Passiamo alla configurazione di rete, poiché ancora una volta, in questo esempio, non stiamo usando un server DHCP:

```bash
# Network information
network --bootproto=static --device=ens192 --gateway=192.168.1.254 --ip=192.168.1.11 --nameserver=192.168.1.254,4.4.4.4 --netmask=255.255.255.0 --onboot=on --ipv6=auto --activate
```

Ricordate che alla fine dell'installazione abbiamo specificato l'utente con cui connettersi a Packer tramite SSH. L'utente e la password devono corrispondere:

```bash
# Root password
rootpw mysecurepassword
```

!!! warning "Attenzione"

    È possibile utilizzare una password non sicura, purché ci si assicuri che questa password venga cambiata immediatamente dopo la distribuzione della macchina virtuale, ad esempio con Ansible.

Ecco lo schema di partizionamento selezionato. Si possono fare cose molto più complesse. È possibile definire uno schema di partizione adatto alle proprie esigenze, adattandolo allo spazio su disco definito in Packer e rispettando le regole di sicurezza definite per il proprio ambiente (partizione dedicata per `/tmp`, ecc.):

```bash
# System booloader configuration
bootloader --location=mbr --boot-drive=sda
# Partition clearing information
clearpart --all --initlabel --drives=sda
# Disk partitionning information
part /boot --fstype="xfs" --ondisk=sda --size=512
part pv.01 --fstype="lvmpv" --ondisk=sda --grow
volgroup vg_root --pesize=4096 pv.01
logvol /home --fstype="xfs" --size=5120 --name=lv_home --vgname=vg_root
logvol /var --fstype="xfs" --size=10240 --name=lv_var --vgname=vg_root
logvol / --fstype="xfs" --size=10240 --name=lv_root --vgname=vg_root
logvol swap --fstype="swap" --size=4092 --name=lv_swap --vgname=vg_root
```

La sezione successiva riguarda i pacchetti che verranno installati. Una "best practice" è quella di limitare la quantità di pacchetti installati solo a quelli necessari, limitando così la superficie di attacco, soprattutto in un ambiente server.

!!! note "Nota"

    All'autore piace limitare le azioni da compiere nel processo di installazione e rimandare l'installazione di ciò che è necessario nello script di post-installazione di Packer. Quindi, in questo caso, installiamo solo i pacchetti minimi richiesti.

Il pacchetto `openssh-clients` sembra essere necessario per consentire a Packer di copiare i suoi script nella macchina virtuale.

`Open-vm-tools` è anche necessario a Packer per rilevare la fine dell'installazione; questo spiega l'aggiunta del repository AppStream. The packages `perl` and `perl-File-Temp` will also be required by VMware Tools during the deployment part. This is a shame because it requires a lot of other dependent packages. Anche `python3` (3.6) sarà richiesto in futuro per il funzionamento di Ansible (se non si usa Ansible o python, rimuoverli!).

```bash
%packages --ignoremissing --excludedocs
openssh-clients
open-vm-tools
python3
perl
perl-File-Temp
curl
dnf-utils
drpm
net-tools
sudo
vim
wget
```

È possibile non solo aggiungere pacchetti, ma anche rimuoverli. Dal momento che controlliamo l'ambiente in cui funzionerà il nostro hardware, possiamo rimuovere qualsiasi firmware che sia inutile per noi:

```bash
# unnecessary firmware
-aic94xx-firmware
-atmel-firmware
...
```

La parte successiva aggiunge alcuni utenti. Nel nostro caso è interessante creare un utente `ansible`, senza password ma con una chiave pubblica. In questo modo tutte le nuove macchine virtuali saranno accessibili dal nostro server Ansible per eseguire le azioni post-installazione:

```bash
# Manage Ansible access
groupadd -g 1001 ansible
useradd -m -g 1001 -u 1001 ansible
mkdir /home/ansible/.ssh
echo -e "<---- PAST YOUR PUBKEY HERE ---->" >  /home/ansible/.ssh/authorized_keys
chown -R ansible:ansible /home/ansible/.ssh
chmod 700 /home/ansible/.ssh
chmod 600 /home/ansible/.ssh/authorized_keys
echo "ansible ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible
```

Ora dobbiamo abilitare e avviare `vmtoolsd` (il processo che gestisce open-vm-tools). vSphere rileverà l'indirizzo IP dopo il riavvio della macchina virtuale.

```bash
systemctl enable vmtoolsd
systemctl start vmtoolsd
```

Il processo di installazione è terminato e la macchina virtuale si riavvia.

## I provisioners

Ricordiamo che in Packer abbiamo dichiarato un provisioner, che nel nostro caso corrisponde a uno script `.sh`, da memorizzare in una sottodirectory accanto al nostro file json.

Esistono diversi tipi di provisioner, avremmo potuto usare anche Ansible. Siete liberi di esplorare queste possibilità.

Questo file può essere interamente modificato, ma fornisce un esempio di ciò che si può fare con uno script, in questo caso `requirements.sh:` Notare sotto “URL dell'articolo della knowledge base” sostituisce un URL interrotto, ma non ne modifica il significato:

```bash
#!/bin/sh -eux

echo "Updating the system..."
dnf -y update

echo "Installing cloud-init..."
dnf -y install cloud-init

# see https://bugs.launchpad.net/cloud-init/+bug/1712680
# and "URL to knowlege base article"
# Virtual Machine customized with cloud-init is set to DHCP after reboot
echo "manual_cache_clean: True " > /etc/cloud/cloud.cfg.d/99-manual.cfg

echo "Disable NetworkManager-wait-online.service"
systemctl disable NetworkManager-wait-online.service

# cleanup current SSH keys so templated VMs get fresh key
rm -f /etc/ssh/ssh_host_*

# Avoid ~200 meg firmware package we don't need
# this cannot be done in the KS file so we do it here
echo "Removing extra firmware packages"
dnf -y remove linux-firmware
dnf -y autoremove

echo "Remove previous kernels that preserved for rollbacks"
dnf -y remove -y $(dnf repoquery --installonly --latest-limit=-1 -q)
dnf -y clean all  --enablerepo=\*;

echo "truncate any logs that have built up during the install"
find /var/log -type f -exec truncate --size=0 {} \;

echo "remove the install log"
rm -f /root/anaconda-ks.cfg /root/original-ks.cfg

echo "remove the contents of /tmp and /var/tmp"
rm -rf /tmp/* /var/tmp/*

echo "Force a new random seed to be generated"
rm -f /var/lib/systemd/random-seed

echo "Wipe netplan machine-id (DUID) so machines get unique ID generated on boot"
truncate -s 0 /etc/machine-id

echo "Clear the history so our install commands aren't there"
rm -f /root/.wget-hsts
export HISTSIZE=0
```

Sono necessarie alcune spiegazioni:

```bash
echo "Installing cloud-init..." dnf -y install cloud-init

# see https://bugs.launchpad.net/cloud-init/+bug/1712680
# and https://kb.vmware.com/s/article/71264
# Virtual Machine customized with cloud-init is set to DHCP after reboot
echo "manual_cache_clean: True" > /etc/cloud/cloud.cfg.d/99-manual.cfg
```

Poiché vSphere ora utilizza cloud-init tramite i VMware Tools per configurare la rete di una macchina guest centos8, è necessario installarlo. Tuttavia, se non si fa nulla, la configurazione verrà applicata al primo riavvio e tutto andrà bene. Ma al successivo riavvio, cloud-init non riceverà alcuna nuova informazione da vSphere. In questi casi, senza informazioni su cosa fare, cloud-init riconfigurerà l'interfaccia di rete della macchina virtuale in modo da utilizzare DHCP, perdendo così la configurazione statica.

Poiché questo non è il comportamento che vogliamo, dobbiamo specificare a cloud-init di non cancellare automaticamente la sua cache e quindi di riutilizzare le informazioni di configurazione che ha ricevuto durante il suo primo riavvio e ogni riavvio successivo.

Per questo, si crea un file `/etc/cloud/cloud.cfg.d/99-manual.cfg` con la direttiva `manual_cache_clean: True`.

!!! note "Nota"

    Ciò implica che se è necessario riapplicare una configurazione di rete tramite le personalizzazioni del guest di vSphere (cosa che, nell'uso normale, dovrebbe essere piuttosto rara), è necessario cancellare la cache di cloud-init.

Il resto dello script è commentato e non richiede ulteriori dettagli.

È possibile consultare il [progetto Bento](https://github.com/chef/bento/tree/master/packer_templates) per avere maggiori idee su cosa si può fare in questa parte del processo di automazione.

## Creazione di Template

Ora è il momento di lanciare Packer e verificare che il processo di creazione, completamente automatico, funzioni bene.

È sufficiente inserire questo comando alla riga di comando:

```bash
./packer build -var-file=~/.vsphere-secrets.json rockylinux8/template.json
```

È possibile passare rapidamente a vSphere e ammirare il lavoro svolto.

Si vedrà la creazione e l'avvio della macchina e, se si lancia una console, si vedrà l'inserimento automatico dei comandi e il processo di installazione.

Al termine della creazione, il modello è pronto per essere utilizzato in vSphere.

## Parte della distribuzione

Questa documentazione non sarebbe completa senza la parte di distribuzione automatica del modello.

A tale scopo, utilizzeremo un semplice playbook di Ansible, che utilizza il modulo `vmware_guest`.

Il manuale che vi forniamo deve essere adattato alle vostre esigenze e al vostro modo di fare.

```ansible
---
- name: Deploy VM from template
  hosts: localhost
  gather_facts: no
  vars_files:
    - ./vars/credentials.yml

  tasks:

  - name: Clone the template
    vmware_guest:
      hostname: "{{ vmware_vcenter_hostname }}"
      username: "{{ vmware_username }}"
      password: "{{ vmware_password }}"
      validate_certs: False
      name: "{{ vm_name }}"
      template: "{{ template_name }}"
      datacenter: "{{ datacenter_name }}"
      folder: "{{ storage_folder }}"
      state: "{{ state }}"
      cluster: "{{ cluster_name | default(omit,true) }}"
      esxi_hostname: "{{ esxi_hostname | default(omit,true) }}"
      wait_for_ip_address: no
      annotation: "{{ comments | default('Deployed by Ansible') }}"
      datastore: "{{ datastore_name | default(omit,true) }}"
      networks:
      - name: "{{ network_name }}"
        ip: "{{ network_ip }}"
        netmask: "{{ network_mask }}"
        gateway: "{{ network_gateway }}"
        device_type: "vmxnet3"
        type: static
      hardware:
        memory_mb: "{{ memory_mb|int * 1024 }}"
        num_cpu: "{{ num_cpu }}"
        hotadd_cpu: True
        hotadd_memory: True
      customization:
        domain: "{{ domain }}"
        dns_servers: "{{ dns_servers.split(',') }}"
      guest_id: "{{ guest_id }}"
    register: deploy_vm
```

È possibile memorizzare i dati sensibili nel file `./vars/credentials.yml`, che ovviamente sarà stato criptato in precedenza con `ansible-vault` (soprattutto se si usa git per il proprio lavoro). Poiché tutto utilizza una variabile, è possibile adattarla facilmente alle proprie esigenze.

Se non utilizzate qualcosa come Rundeck o Awx, potete lanciare l'installazione client con una riga di comando simile a questa:

```bash
ansible-playbook -i ./inventory/hosts  -e '{"comments":"my comments","cluster_name":"CS_NAME","esxi_hostname":"ESX_NAME","state":"started","storage_folder":"PROD","datacenter_name":"DC_NAME}","datastore_name":"DS_NAME","template_name":"template-rockylinux8-0.0.1","vm_name":"test_vm","network_name":"net_prod","network_ip":"192.168.1.20","network_gateway":"192.168.1.254","network_mask":"255.255.255.0","memory_mb":"4","num_cpu":"2","domain":"rockylinux.lan","dns_servers":"192.168.1.254","guest_id":"centos8_64Guest"}' ./vmware/create_vm.yml --vault-password-file /etc/ansible/vault_pass.py
```

A questo punto è possibile avviare la configurazione finale della macchina virtuale utilizzando Ansible. Non dimenticate di cambiare la password di root, proteggere SSH, registrare la nuova VM nel vostro strumento di monitoraggio e nell'inventario IT, ecc.

## In sintesi

Come abbiamo visto, oggi esistono soluzioni DevOps completamente automatizzate per creare e distribuire le macchine virtuali.

Allo stesso tempo, questo rappresenta un innegabile risparmio di tempo, soprattutto in ambienti cloud o data center. Inoltre, facilita una conformità standard per tutti i computer dell'azienda (server e postazioni di lavoro) e una facile manutenzione / evoluzione dei modelli.

## Altri riferimenti

Per un progetto dettagliato che copre anche la distribuzione di Rocky Linux e di altri sistemi operativi utilizzando le ultime novità di vSphere, Packer e il Packer Plugin per vSphere, visitate [questo progetto](https://github.com/vmware-samples/packer-examples-for-vsphere).
