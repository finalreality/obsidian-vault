安装
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl build-essential -y
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```
 安装 nvm
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

激活 nvm (如果当前终端未生效，关闭终端重开，或运行下面这行)
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

安装 Node.js 24.x
```bash
nvm install 24.14.0
nvm use 24.14.0
nvm alias default 24
```

验证
```
node -v
npm -v
```
安装openclaw
```
npm install -g openclaw

openclaw onboard --install-daemon
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
openclaw gateway restart
```
clawhub安装skills问题：
```bash
$ clawhub install weather
✖ Rate limit exceeded
Error: Rate limit exceeded
```

```bash
clawhub login --token clh_0tH9uB7DXvSK008hMbQ6TUgDTa0UBimnBoyH9yv6nbQ
```