# Zsh Setup Guide for Ubuntu Server

This guide will help you install **Zsh**, set it as your default shell, install **Oh My Zsh**, add useful plugins, and configure the **Powerlevel10k** theme.

---

## 1. Install Zsh

```bash
sudo apt update
sudo apt install -y zsh
```

Check installation:

```bash
zsh --version
```

---

## 2. Make Zsh the default shell

```bash
chsh -s $(which zsh)
```

You will need to **log out and log back in** for this change to take effect.

---

## 3. Install Oh My Zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

---

## 4. Install Plugins

### zsh-autosuggestions

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

### zsh-syntax-highlighting

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### fast-syntax-highlighting

```bash
git clone https://github.com/zdharma-continuum/fast-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/fast-syntax-highlighting
```

### zsh-autocomplete

```bash
git clone https://github.com/marlonrichert/zsh-autocomplete.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autocomplete
```

---

## 5. Enable Plugins

Edit `~/.zshrc` and set the plugins line to:

```bash
plugins=(
  git
  docker-compose
  zsh-autosuggestions
  zsh-syntax-highlighting
  fast-syntax-highlighting
  zsh-autocomplete
)
```

Save and reload:

```bash
source ~/.zshrc
```

---

## 6. Install Powerlevel10k Theme

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
```

Edit `~/.zshrc`:

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

Reload:

```bash
source ~/.zshrc
```

---

## 7. Configure Powerlevel10k

Run:

```bash
p10k configure
```

Follow the interactive setup wizard to customize your prompt.

---

## 8. Optional: Font for Icons

If you want icons and symbols to display correctly, install a Nerd Font (locally, if connecting via SSH, install the font on your **client machine**):

1. Download from [Nerd Fonts](https://www.nerdfonts.com/font-downloads)
2. Install it on your local terminal
3. Set your terminal to use the installed Nerd Font.

---

✅ Done! You now have a fully customized Zsh setup with Powerlevel10k and helpful plugins.
