# **Arch Linux - Instalação Completa**


### **Bootando com o LiveUSB do Arch**

Com o LiveUSB montado, carregue a configuração correta para o seu teclado. Geralmente:

	loadkeys br-abnt2

**Para conectar-se à internet**, pode ser usado um cabo de rede comum (ethernet). Não é necessário configurar. Para conexão via Wi-Fi usamos `iwctl`.

Para iniciar:

	iwctl

Procure os dispositivos disponíveis para conexão:

	iwctl device list

O dispositivo encontrado, geralmente `wlan0` em modo `station`, será usado para encontrar redes disponíveis:

	iwctl station wlan0 scan

	iwctl station wlan0 get-networks

Caso o dispositivo tenha outro nome ou esteja em outro modo, o código deverá ser ajustado.

Para conectar-se à sua rede:

	iwctl station wlan0 connect "network"

Substitua "network" pelo nome da sua rede. Ele pedirá sua senha.

Confira se a conexão foi bem sucedida com:

	iwctl station wlan0 show

Volte ao modo root com `exit`.

### **Editar e formatar partições com Gparted**

Todas as partições que serão usadas no novo sistema **para armazenar arquivos** deverão estar no formato `ext4`, a partição de boot deve estar em `fat32`, e a partição de swap deve estar em `linux-swap`. Em geral é necessário ter apenas uma partição de armazenamento, mas é possível configurar mais de uma. Inclusive, é uma boa prática isolar a pasta `/home` numa partição separada da sua partição `/`, a raiz do sistema. Aqui, faremos dessa forma, considerando um SSD/ HDD de 240GB:

- Boot: `fat32`, `100MB`
- Root: `ext4`, `100GB`
- Home: `ext4`, `120GB`
- Swap: `linux-swap`, `12GB`

Caso tenha outras partições no mesmo disco ou em outro, podemos adicioná-las depois.

Para ver quais partições podem ser usadas:

	lsblk -f

Para modificar, apagar, redimensionar ou mover partições, usaremos o `gparted`:

	sudo pacman -S gparted

Sua interface é bem intuitiva e fácil de usar.

Caso precise apenas formatar as partições para o formato correto, sem que seja necessário alterar mais nada, podemos usar simplesmente:

Para `fat32`:

	mkfs.fat -F32 /dev/nvme0n1pX

Para `ext4`:

	mkfs.ext4 /dev/nvme0n1pY

Para `linux-swap`:

	/dev/nvme0n1pZ

Substituindo `X`, `Y` e `Z` pelo número correspondente de cada partição.

### **Montando as partições**

Com todas as partições formatadas, devemos então montá-las para que possam ser usadas. Tomando este esquema como exemplo:

- Boot: `/dev/nvme0n1p1`

- Root: `/dev/nvme0n1p2`

- Home: `/dev/nvme0n1p3`

- Swap: `/dev/nvme0n1p4`

Montaremos primeiro nossa partição `root`:

	mount /dev/nvme0n1p2 /mnt

Em seguida devemos criar os diretórios corretos para as próximas partições:

	mkdir -p /mnt/boot/efi

	mkdir -p /mnt/home

Agora podemos montar as demais partições, respectivamente:

	mount /dev/nvme0n1p1 /mnt/boot/efi

	mount /dev/nvme0n1p3 /mnt/home

Algumas fontes ensinam que a partição `boot` deve ser montada em `/mnt/boot`, mas se fizermos isso teremos um problema logo adiante. Quando iniciarmos a instalação básica do sistema, veremos que alguns arquivos de kernel serão criados em `/mnt/boot`. Se a nossa partição de `boot` estiver montada ali, esses arquivos serão criados nela. O primeiro ponto é que não haverá espaço para todos os arquivos. E mesmo que haja, não é nada prático. Caso você queria montar em `/mnt/boot` mesmo assim, apenas se certifique de que a partição tenha pelo menos 1GB livre para os arquivos de kernel.

Por último, nossa partição `swap`. É a única que não iremos montar. Ela deve ser ativada:

	swapon /dev/nvme0n1p4

Agora podemos prosseguir para a instalação do sistema.

### **Instalação básica do Arch Linux**

Com todas as partições montadas corretamente, devemos instalar os pacotes necessários e mais alguns essenciais:

	pacstrap -K /mnt base base-devel linux linux-firmware git grub efibootmgr sudo nano vim fastfetch htop intel-ucode bluez bluez-utils networkmanager

Com isso, já temos uma instalação básica do sistema.

### **Definindo labels e configurando o GRUB**

Aqui precisamos entender que o GRUB, nosso bootloader, usará como base para parâmetros para iniciar nosso sistema as informações contidas num arquivo chamado `fstab`, em `/etc/fstab`. No entanto, há dois métodos de configurá-lo: via `Labels` e via `UUIDs`. Particularmente, eu prefiro usar `Labels`. Geralmente dá menos problema depois. Caso prefira usar `UUIDs`, pode gerar o `fstab` imediatamente:

	genfstab -U /mnt >> /mnt/etc/fstab

Agora, para definir nossas `Labels`, precisamos de um passo adicional. Uma Label é basicamente um nome que atribuímos a cada partição para uma fácil localização. Cada formato receberá um tratamento específico.

Para `ext4`:

	sudo e2label /dev/nvme0n1p2 ArchRoot

	sudo e2label /dev/nvme0n1p3 Home

Para `fat32`:

	sudo fatlabel /dev/nvme0n1p2 ArchBOOT

Para `swap`, primeiro desative a partição:

	sudo swapoff /dev/nvme0n1p4

	sudo mkswap -L SwapArch /dev/nvme0n1p4

Agora, gere um novo arquivo fstab.

	genfstab -L /mnt >> /mnt/etc/fstab

Confira se está tudo correto com `nano` ou `cat`.

	nano /mnt/etc/fstab
	
	cat /mnt/etc/fstab

### **Entrando no novo sistema**

Para prosseguir com as configurações, devemos entrar no sistema com `arch-chroot`:

	arch-chroot /mnt

### **Instalação e configuração do GRUB**

Baixe todos os pacotes necessários, caso ainda não tenha feito:

	pacman -S grub (if not yet installed) efibootmgr dosfstools mtools os-prober

Instale os arquivos necessários para boot via GRUB

	grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB

Com o GRUB instalado, precisamos configurá-lo. Para isso devemos editar o arquivo `/etc/default/grub` com `nano`.

	nano /etc/default/grub

Navegue até `GRUB_DISABLE_OS_PROBER=false` e descomente essa linha. Isso fará com que o pacote `os prober` seja ativado e executado na próxima etapa. isso nem sempre é necessário, mas evita dor de cabeça. Rode:

	grub-mkconfig -o /boot/grub/grub.cfg

Atente-se para os sistemas encontrados. Ele deve informar que encontrou os arquivos de kernel do nosso linux. Se não informar, há algum erro, e, nesse caso, o sistema não será iniciado.

### **Configuração inicial do sistema**

Senha do root:

	passwd

Criação de usuário:

	useradd -m -g users -G wheel,storage,power,video,audio -s /bin/bash seu_usuario
 
	passwd seu_usuario

	EDITOR=nano visudo

Descomente para permitir aos membros do grupo `wheel` executar qualquer comando:

	%wheel ALL=(ALL:ALL) ALL

Atualize o sistema:

	su - seu_usuario

	sudo pacman -Syu

	exit
