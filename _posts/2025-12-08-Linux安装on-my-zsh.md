
---
title: debian安装on-my-zsh
date: 2025-12-08 20:00:00 +0800
categories: [linux,zsh]
tags: [on-my-zsh]
---

系统:debian12
其他Linux系统安装操作类似

debian默认不带zsh
```
sudo apt update
sudo apt install zsh -y
```

安装好以后检查是否安装成功
```
zsh --version
```

安装on-my-zsh
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

安装过程中它会询问是否要把默认shell改成zsh，直接回车即可

![](/assets/img/Pasted image 20251207203617.png)

如图on-my-zsh安装成功，此时默认shell已经变成zsh
![](/assets/img/Pasted image 20251207203728.png)


此时zsh还没有任何插件，需要手动安装

安装自动命令提示插件
```
git clone https://github.com/zsh-users/zsh-autosuggestions \
${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

安装命令语法高亮(命令正确绿色，错误红色)
```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git \
${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

![](/assets/img/Pasted image 20251207205128.png)



在`~/.zshrc`启用插件
```
vim ~/.zshrc
```

找到`plugins=`这一行
```
plugins=(git)
```

改成如下内容:
```
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting
)
```

使配置生效
```
source ~/.zshrc
```

此时zsh就具备了自动命令提示和语法高亮的功能了
