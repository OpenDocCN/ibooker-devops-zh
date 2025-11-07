# 附录 E. 安装 GPG

GNU 隐私守护者 (GPG) 是一种开放标准的专有优秀隐私 (PGP) 加密方案的实现，通常用于加密和解密电子邮件和文件系统内容。对称密钥加密和公钥加密的结合提供了一种快速且安全的信息交换方法。用户必须首先使用 GPG 工具生成一个公私钥对，以启用信息的加密和解密。私钥在加密过程中使用，而公钥在解密时使用。公钥可以与任何需要解密加密信息的人共享，也可以托管在互联网密钥服务器上，以简化更广泛受众解密加密内容的方式。通过使用 GPG 工具——特别是 `gpg` 命令行界面——可以简化公私钥对的创建以及信息的加密和解密。

## E.1 获取 GPG 工具

GPG 工具以及 `gpg` 命令行界面可以安装在大多数主要操作系统上，并且可以通过直接下载或从包管理器（如 `apt`、`dnf` 或 `brew`）获取。

在基于 Debian 的 Linux 操作系统上，请在终端中执行以下命令：

```
apt install gnupg
```

在基于 RPM 的操作系统上，例如 Fedora 或 Red Hat Enterprise Linux，请在终端中执行以下命令：

```
dnf install gnupg
```

在基于 OSX 的操作系统上，请在终端中执行以下命令：

```
brew install gpg
```

通过使用 `--version` 标志来检查工具的版本，以确认 `gpg` 命令行界面是否成功安装。

列表 E.1 显示 GPG 版本

```
gpg (GnuPG) 2.2.20
libgcrypt 1.8.5
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later
➥<https://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Home: /root/.gnupg       ①
Supported algorithms:
Pubkey: RSA, ELG, DSA, ECDH, ECDSA, EDDSA
Cipher: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256, TWOFISH,
        CAMELLIA128, CAMELLIA192, CAMELLIA256
Hash: SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
Compression: Uncompressed, ZIP, ZLIB, BZIP2
```

① 包含 GPG 文件的目录

如果显示类似于列表 E.1 的响应，则表示 GPG 工具已成功安装。

## E.2 生成公私钥对

在能够加密内容之前，必须生成密钥对。生成密钥对的过程在第三章中也有讨论，但此处将再次讨论以确保完整性。

可以使用 `gpg` 命令行界面通过以下标志之一创建 GPG 密钥：

+   `--quick-generate-key`——在用户提供 `USER-ID` 以及过期和算法详情的同时生成密钥对

+   `--generate-key`——在提示真实姓名和电子邮件详情的同时生成密钥对

+   `--full-generate-key`——所有可能的密钥对生成选项的对话框

您选择的选项取决于您的个人需求。`--generate-key` 和 `--full-generate-key` 选项支持批处理模式，这允许通过非交互式方法创建密钥对。

使用 `--generate-key` 标志创建一个新的 GPG 密钥对：

```
gpg --generate-key
```

一旦执行此命令，就会在您的家目录中的 `.gnupg` 文件夹内创建一个 GPG 文件的主目录。此位置可以通过指定 `GNUPGHOME` 环境变量来更改。

在提示时提供您的姓名和电子邮件地址。然后您将被要求确认这些详情。按 `O` 确认详情。

接下来，系统会提示您提供一个密码短语来保护您的密钥。第三章的练习建议不要创建密码短语，以简化整合所使用的每个安全工具。然而，在为其他用途创建 GPG 密钥对时，建议提供密码短语。一旦提供了密码短语，将生成一个新的密钥对，并显示与密钥相关的详细信息，如下所示列表。

列表 E.2 显示 GPG 版本

```
pub   rsa2048 2020-12-31 [SC] [expires: 2022-12-31]
      53696D1AB6954C043FCBA478A23998F0CBF2A552
uid           [ultimate] John Doe <jdoe@example.com>
sub   rsa2048 2020-12-31 [E] [expires: 2022-12-31]
```

您可以通过提供的值确认名称和电子邮件是否已正确添加，包括算法、密钥大小和过期时间。现在密钥对已生成，可以使用 GPG 工具集加密消息。
