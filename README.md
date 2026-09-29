![Licença](https://img.shields.io/badge/licença-MIT-green)
![Status](https://img.shields.io/badge/status-ativo-brightgreen)
![Plataforma](https://img.shields.io/badge/plataforma-Android%20%7C%20Linux-blue)

# Guia Completo: Transformando um Celular Antigo em Servidor Pessoal (Homelab)

---

## 📖 Sobre este guia

Este guia documenta o processo completo de transformar um celular Android antigo (no caso, um **Motorola G23** com tela quebrada, 3.67 GB de RAM e Android 14) em um servidor de arquivos pessoal acessível pela rede local.

Aqui irei usar o Copyparty, porque meu celular é um lixo e o NextCloud usa bastante memória RAM, se tiver um celular melhor que o especificado, pode usar o Nextloud, que é uma ferramenta até melhor, expandindo muito o que se pode fazer.

**O que você vai aprender:**
- Instalar e configurar o Termux no Android
- Acessar o celular via SSH do PC
- Criar um "home" limpo no armazenamento do celular
- Instalar e configurar o Copyparty (servidor de arquivos leve)
- Rodar o Copyparty como serviço permanente
- Montar o servidor como pasta local no Linux via rclone
- Criar aliases no Zsh/Bash para agilizar o dia a dia
- Comparar alternativas (Nextcloud, FileBrowser Quantum, ARTH)

**O que você NÃO vai precisar:**
- Root no celular
- Docker
- Apache, PHP ou MariaDB
- Conhecimento avançado em Linux

---

## 📋 Índice

1. [Pré-requisitos](#1-pré-requisitos)
2. [Preparando o Terreno](#2-preparando-o-terreno)
3. [Acesso Remoto via SSH](#3-acesso-remoto-via-ssh)
4. [Criando um "Home" Limpo](#4-criando-um-home-limpo)
5. [Instalando o Copyparty](#5-instalando-o-copyparty)
6. [Rodando como Serviço](#6-rodando-como-serviço)
7. [Montando no PC com rclone](#7-montando-no-pc-com-rclone)
8. [Aliases no Zsh](#8-aliases-no-zsh)
9. [Segurança e Acesso Externo](#9-segurança-e-acesso-externo)
10. [Alternativas e Comparações](#10-alternativas-e-comparações)
11. [Solução de Problemas](#11-solução-de-problemas)
12. [Recursos](#12-recursos)

---

## 1. Pré-requisitos

### No celular
- Android 7 ou superior (testado no Android 14)
- Pelo menos 2 GB de RAM
- Armazenamento livre (recomendado: 2 GB+)
- Conexão Wi-Fi estável

### No PC
- Linux (testado no Arch Linux)
- Terminal com Zsh ou Bash
- `rclone` e `fuse3` instalados

### Conhecimentos básicos
- Comandos básicos de terminal (`cd`, `ls`, `mkdir`, `nano`)
- Noções de rede (IP, porta, SSH)

---

## 2. Preparando o Terreno

### 2.1 Instalar o Termux

**⚠️ Importante:** Instale o Termux pelo **F-Droid**, não pela Play Store. A versão da Play Store está desatualizada e pode causar problemas de compatibilidade.

- **F-Droid:** https://f-droid.org/packages/com.termux/

**Apps opcionais (recomendados):**
- **Termux:Boot** — para iniciar serviços automaticamente com o celular
- **Termux:API** — para notificações e acesso a recursos do Android

### 2.2 Primeiros comandos

Abra o Termux e execute:

```bash
pkg update && pkg upgrade -y
termux-setup-storage
```

O `termux-setup-storage` vai pedir permissão de armazenamento. **Aceite.**

### 2.3 Configurações essenciais do Android

Para evitar que o Android mate o Termux em segundo plano:

1. **Desative a otimização de bateria:**
   - Configurações > Apps > Termux > Bateria > **Não otimizar**

2. **Ative "Desativar restrições de processos filhos":**
   - Configurações > Sistema > Opções do desenvolvedor > **Desativar restrições de processos filhos**

3. **Mantenha a tela ligada durante a configuração:**
   - Opções do desenvolvedor > **Manter tela ligada**

---

## 3. Acesso Remoto via SSH

### 3.1 Por que SSH?

Se a tela do celular está quebrada (como no nosso caso), digitar nele é sofrível. Com SSH, você usa o teclado e a tela do PC para fazer tudo.

### 3.2 No celular (Termux)

```bash
pkg install openssh -y
passwd          # definir uma senha (não aparece nada na tela)
sshd            # iniciar o servidor SSH
ifconfig        # anotar o IP (ex: 192.168.100.52)
whoami          # anotar o usuário (ex: u0_a220)
```

### 3.3 No PC

```bash
ssh -p 8022 u0_a220@192.168.100.52
```

**Notas:**
- A porta padrão do SSH no Termux é **8022**, não 22
- Na primeira conexão, digite `yes` quando perguntar sobre a fingerprint
- Digite a senha que você definiu com o `passwd`

### 3.4 Se o SSH cair sozinho

O Android tende a matar o processo do Termux quando o app vai para segundo plano. Soluções:

- Desative a otimização de bateria (passo 2.3)
- Mantenha a tela ligada
- Use `tmux` para manter a sessão viva:

```bash
pkg install tmux -y
tmux new -s server
# Se o SSH cair, reconecte e rode:
tmux attach -t server
```

---

## 4. Criando um "Home" Limpo

### 4.1 O problema

O Android tem pastas padrão (`Android`, `DCIM`, `Download`, etc.) que poluem tudo. Você não quer isso no seu servidor.

### 4.2 A solução

No Termux (fora do Ubuntu), crie uma pasta dedicada:

```bash
mkdir -p /storage/emulated/0/ServerHome
mkdir -p /storage/emulated/0/ServerHome/Notas
mkdir -p /storage/emulated/0/ServerHome/Documentos
mkdir -p /storage/emulated/0/ServerHome/Backups
```

Agora você tem um "home" limpo, só com o que você quer.

---

## 5. Instalando o Copyparty

### 5.1 O que é o Copyparty?

- Servidor de arquivos em **um único arquivo Python**
- Leve, rápido, com interface web, WebDAV, upload retomável
- Muito mais leve que Nextcloud (que exige Apache + PHP + MariaDB)
- Ativamente mantido

### 5.2 Instalação

```bash
pkg install python -y
pip install copyparty
```

### 5.3 Primeira execução

```bash
copyparty -v /storage/emulated/0/ServerHome:/root:rwadm,admin -a admin:SuasSenha
```

**Explicando os parâmetros:**

| Parâmetro | Significado |
|---|---|
| `-v /storage/emulated/0/ServerHome:/root` | Compartilha a pasta ServerHome como "raiz" do servidor |
| `:rwadm,admin` | Dá permissões de **R**ead, **W**rite e **A**dmin para o usuário admin |
| `-a admin:SuasSenha` | Define usuário e senha |

### 5.4 Acessando do PC

Abra o navegador e acesse:

```
http://192.168.100.52:3923
```

**Notas:**
- A porta padrão é **3923**
- Use `--qr` para gerar um QR Code e conectar de outros celulares facilmente

### 5.5 Dicas

- Ative a busca e o upload-undo com `-e2dsa`
- Ative a deduplicação com `--dedup` (leia as consequências no README)
- Use `--qr` para gerar QR Code

---

## 6. Rodando como Serviço

### 6.1 O problema

Se fechar o Termux, o Copyparty para. Você quer que ele inicie sozinho e fique sempre ativo.

### 6.2 Opção 1: termux-services (recomendado)

```bash
pkg install termux-services -y
source $PREFIX/etc/profile.d/start-services.sh

mkdir -p $PREFIX/var/service/copyparty
cat > $PREFIX/var/service/copyparty/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
exec 2>&1
exec copyparty -v /storage/emulated/0/ServerHome:/root:rwadm,admin -a admin:SuasSenha
EOF

chmod +x $PREFIX/var/service/copyparty/run
sv-enable copyparty
sv up copyparty
```

**Comandos úteis:**

| Comando | Função |
|---|---|
| `sv status copyparty` | Ver status |
| `sv down copyparty` | Parar |
| `sv up copyparty` | Iniciar |

**⚠️ Importante:** O `pkill` normal **não funciona** com o termux-services (o `runsv` reinicia o processo). Use `sv down` para parar de verdade.

### 6.3 Opção 2: Termux:Boot

- Instale o app **Termux:Boot** pelo F-Droid
- Abra o app uma vez para registrá-lo
- Crie um script em `~/.termux/boot/start-copyparty`:

```bash
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
if ! pgrep -f "copyparty -v" > /dev/null; then
    copyparty -v /storage/emulated/0/ServerHome:/root:rwadm,admin -a admin:SuasSenha &
fi
```

- Torne executável:

```bash
chmod +x ~/.termux/boot/start-copyparty
```

---

## 7. Montando no PC com rclone

### 7.1 O que é o rclone?

- "O rsync para armazenamento em nuvem"
- Suporta WebDAV, que é o protocolo que o Copyparty expõe
- Permite montar o servidor como uma pasta local no PC

### 7.2 Instalação no Arch

```bash
sudo pacman -S rclone fuse3
```

### 7.3 Configurando o remote

```bash
rclone config
```

**Passo a passo:**

| Prompt | Resposta |
|---|---|
| `n` para novo remote | `n` |
| Nome | `copyparty` |
| Tipo | `webdav` |
| URL | `http://192.168.100.52:3923` |
| Vendor | `other` |
| User | `admin` |
| Password | `SuasSenha` |
| Confirme | `y` |
| Saia | `q` |

### 7.4 Testando

```bash
rclone lsd copyparty:        # lista pastas
rclone ls copyparty:root     # lista arquivos
```

### 7.5 Montando como pasta local

```bash
mkdir -p ~/Copyparty
rclone mount copyparty:root ~/Copyparty --vfs-cache-mode full &
```

Agora `~/Copyparty` é uma janela para os arquivos do celular.

### 7.6 Desmontando

```bash
fusermount -u ~/Copyparty
```

### 7.7 Dicas

- Use `--allow-non-empty` se a pasta já estiver montada
- O `--vfs-cache-mode full` melhora a performance e permite edição de arquivos
- Para montar automaticamente ao ligar o PC, adicione ao `~/.zshrc` ou crie um serviço systemd de usuário

---

## 8. Aliases no Zsh

### 8.1 Por que aliases?

Digitar o comando completo do rclone toda vez é chato. Aliases deixam tudo mais rápido.

### 8.2 Editando o .zshrc

```bash
nano ~/.zshrc
```

### 8.3 Adicionando no final

```bash
# --- ALIASES COPYPARTY / RCLONE ---
alias cpp-mount='rclone mount copyparty:root ~/Copyparty --vfs-cache-mode full --allow-non-empty &'
alias cpp-umount='fusermount -u ~/Copyparty'
alias cpp-ls='rclone ls copyparty:'
alias cpp-sync='rclone sync ~/Notas copyparty:root/Notas --progress'
```

### 8.4 Recarregando

```bash
source ~/.zshrc
```

### 8.5 Usando

```bash
cpp-mount      # monta
cpp-ls         # lista arquivos
cpp-umount     # desmonta
cpp-sync       # sincroniza ~/Notas com o celular
```

### 8.6 Nota sobre Zsh vs Bash

A sintaxe de aliases no Zsh é **idêntica** à do Bash. A diferença está só no arquivo de configuração:
- **Bash:** `~/.bashrc`
- **Zsh:** `~/.zshrc`

---

## 9. Segurança e Acesso Externo

### 9.1 Acesso na rede local

Funciona automaticamente se PC e celular estiverem no mesmo Wi-Fi. Use o IP do celular (ex: `192.168.100.52`).

### 9.2 Acesso de fora de casa

**Tailscale** é a solução mais simples e segura:

- Cria uma VPN privada entre seus dispositivos
- Não precisa abrir portas no roteador
- Instale no celular (Play Store) e no PC (`sudo pacman -S tailscale`)
- Depois, acesse pelo IP Tailscale (algo como `100.x.y.z`)

### 9.3 Dicas de segurança

- Troque a senha padrão do Copyparty por uma forte
- Use senhas de aplicativo se tiver 2FA
- Não exponha o servidor diretamente à internet sem VPN

---

## 10. Alternativas e Comparações

| Ferramenta | Prós | Contras | Adequação |
|---|---|---|---|
| **Copyparty** | Leve, um único arquivo Python, WebDAV, upload retomável | Interface menos polida | ⭐⭐⭐⭐⭐ |
| **FileBrowser Quantum** | Interface moderna, multiusuário, compartilhamento de links | Um pouco mais pesado | ⭐⭐⭐⭐ |
| **Nextcloud** | Completo (calendário, contatos, edição online) | Pesado demais para celular com 3 GB de RAM | ⭐⭐ |
| **ARTH** | Feito para Termux, leve, dashboard do sistema | Sem suporte a acesso externo | ⭐⭐⭐⭐ |
| **Syncthing** | Sincronização bidirecional automática | Não é um servidor web | ⭐⭐⭐ |

**Conclusão:**
- Para celular simples: **Copyparty** é a melhor escolha
- Para quem quer interface mais bonita: **FileBrowser Quantum**
- Para quem tem hardware melhor: **Nextcloud**

---

## 11. Solução de Problemas

### SSH não conecta

- Verifique se o `sshd` está rodando: `pgrep sshd`
- Confirme o IP do celular: `ifconfig`
- Verifique se PC e celular estão na mesma rede

### Copyparty não inicia

- Verifique se a porta 3923 está livre: `pgrep -f copyparty`
- Mate processos antigos: `pkill -9 -f copyparty`
- Verifique se a pasta compartilhada existe: `ls /storage/emulated/0/ServerHome`

### rclone dá erro 401

- Verifique a senha no `rclone config`
- Recrie o remote se necessário: `rclone config delete copyparty`

### Mount dá "directory already mounted"

- Desmonte primeiro: `fusermount -u ~/Copyparty`
- Ou use `--allow-non-empty`

### Android mata o Termux

- Desative a otimização de bateria
- Ative "Desativar restrições de processos filhos"
- Use `termux-wake-lock`

### Porta 3923 ocupada

- Mate todas as instâncias: `pkill -9 -f copyparty`
- Se estiver usando termux-services: `sv down copyparty`

---

## 12. Recursos

- **Termux (F-Droid):** https://f-droid.org/packages/com.termux/
- **Copyparty:** https://github.com/9001/copyparty
- **rclone:** https://rclone.org/
- **Tailscale:** https://tailscale.com/
- **FileBrowser Quantum:** https://github.com/gtsteffaniak/filebrowser
- **Documentação do Termux:** https://termux.dev/docs
- **Termux:Boot:** https://f-droid.org/packages/com.termux.boot/
- **Termux:API:** https://f-droid.org/packages/com.termux.api/

---



## ⭐ Agradecimentos

- Ao desenvolvedor do Copyparty, por criar uma ferramenta tão leve e poderosa
- À comunidade do Termux, por manter o projeto vivo
- A todos que compartilham conhecimento para que possamos aprender

---

**Se este guia te ajudou, deixe uma estrela no repositório! ⭐**
