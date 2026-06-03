---
title: 低成本快速搞定专属域名邮箱：从 0 到 CLI 收发只要 15 分钟
date: 2025-11-15 23:36:22
tags: [技术分享]
---

> 失踪人口回归 👋
> 如果你只想「有个能用的 `@自己域名` 邮箱」，却懒得租服务器、折腾 Postfix、跟 25 端口斗智斗勇、花费你宝贵的💰，那这篇 15 分钟速通教程就是写给你的（当然，一点💰都不花是不可能的，你至少需要买一个域名=)

---

## 1. 最终效果

- 任何前缀都能即时生成： `helloworld@your.com` / `fknudt@your.com` / `ilovethu@your.com`  
- 在终端里使用命令收发邮件，无需专门打开各种邮件客户端。

整套方案 **只花域名注册费用**，**0 服务器 0 运维**，完全白嫖 Resend 免费额度。

---

## 2. 准备工作（2 min）

| 资源 | 用途 | 费用 |
|---|---|---|
| 一枚域名 | 做邮箱后缀 | 约 ￥30+ / 年（阿里云 Namecheap Cloudflare 均可） |
| Resend 账号 | 托管邮服 | 免费 100 封 / 天 |

![Resend文档](/images/resendofficial.png)

---

## 3. 3 步完成域名绑定（5 min）

1. 登录 [Resend](https://resend.com) → **Domains** → **Add Domain**  
   输入 `your.com` → 得到一组 DNS 记录（TXT + MX + DKIM）。  

2. 去域名注册商把记录全部照抄粘贴，保存。(由于笔者为免费用户并且已经配置域名无法再次配置，因此未能提供截图但是步骤非常简单）。
   - 登录你的域名注册商账号，找到DNS管理页面。
   - 添加Resend提供的TXT记录，用于验证域名所有权。
   - 添加Resend提供的MX记录，用于接收邮件。
   - 根据Resend的要求，配置完整所有必要的DNS记录。
   - 保存更改后，等待DNS记录生效，通常需要几分钟到24小时不等（这是resend官方的说法，笔者实测只需要几分钟不到）。
   ⚠️ 只需改 **DNS 解析**，不用转 nameserver，1 min 搞定。  

3. 回到 Resend 点 **Verify**，状态变绿 ✅ 即生效（实测 30s-3min，并不会像官方说的有可能要24小时）。

---

## 4. 无限别名 + 转发（立刻能用）

Resend 对「 catch-all 」免费！  
意思是你 **不用提前新建账号**，任何 `@your.com` 的信都会飞进你设定的地址，只需要在 Resend 官网查收即可。

---

## 5. 终端发邮件（可选，8 min 部署）

网页收发邮件显然太麻烦，我用**rust**写了个可爱的 CLI 客户端 [**rusend**](https://github.com/CGH0S7/rusend)，基于 Resend 官方 API，支持发送与查收邮件非常方便。（好吧其实是Gemini小姐帮忙写的@@）

### ①-1 下载预构建可执行文件

在[Github Releases](https://github.com/CGH0S7/rusend/releases/tag/stable)里下载对应系统的预编译包，解压放到 `$PATH` 目录下即可。

### ①-2 源码构建

下载并编译rusend，准备好rust工具链，具体编译命令如下：

  ```bash
  git clone https://github.com/CGH0S7/rusend.git
  cd rusend
  cargo build --release
  ```

将`rusend`二进制文件移动到系统的`$PATH`目录下，例如`/usr/local/bin`。

### ② 补全 & API Key配置

使用下面的命令进行命令行补全配置：

  ```bash
  echo "source <(rusend completions bash)" >> ~/.bashrc
  source ~/.bashrc
  ```

  ```zsh
  echo "source <(rusend completions zsh)" >> ~/.zshrc
  source ~/.zshrc
    ```

  ```fish
  rusend completions fish > ~/.config/fish/completions/rusend.fish
  source ~/.config/fish/config.fish
   ```

在Resend控制面板中，找到`API Keys`选项，为你配置的域名生成一个新的API Key。然后在终端中运行以下命令进行配置：

  ```bash
  rusend config --key re_xxxxxxxxx
  ```

### ③ 使用 rusend 收发邮件

详细功能请查看[Readme](https://github.com/CGH0S7/rusend)文档：

  ![rusend文档](/images/rusendhelp1.png)
  ![rusend文档](/images/rusendhelp2.png)
  ![rusend软件帮助](/images/rusendhelp3.png)

---

## 6. 成本 & 限额总结

| 项目 | 免费额度 |
|---|---|
| Resend | 100 封 / 天 |
| 域名 | 约 ￥30+ / 年 |
| rusend | 开源 |

非主力邮箱场景下完全够用，不建议用于接收各种广告或更新推送。

若想要「匿名转发」隐藏真实地址，可以再套一层 [AnonAddy](https://addy.io) 即可，本文不展开。

---

## 7. 结语

以上就是「低成本快速拥有个人专属域名邮箱」的全部流程。

零服务器、零运维、全平台通用，甚至能在手机 Termux 里发邮。

如果能帮到你，去 [rusend](https://github.com/CGH0S7/rusend) 点个 ⭐ 当稿费吧～😉
