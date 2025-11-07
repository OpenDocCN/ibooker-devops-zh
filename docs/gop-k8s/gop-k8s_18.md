# 附录 C. 配置 GPG 密钥

GPG，或 GNU 隐私守护者，是一种公钥加密实现。GPG 允许在各方之间安全地传输信息，并可用于验证消息来源的真实性。以下是为设置 GPG 密钥的步骤：

1.  首先，我们需要安装 GPG 命令行工具。无论您的操作系统是什么，这个过程可能需要一些时间。macOS 用户可能使用 `brew` 软件包管理器通过以下命令安装 GPG：

    ```
    brew install gpg
    ```

1.  下一步是生成一个将用于签名和验证提交的 GPG 密钥。使用以下命令生成密钥。在提示时，按 Enter 键接受默认密钥设置。在输入用户身份信息时，请确保使用您 GitHub 账户的已验证电子邮件：

    ```
    gpg --full-generate-key
    ```

1.  查找生成的密钥 ID，并使用该 ID 访问 GPG 密钥主体：

    ```
    gpg --list-secret-keys --keyid-format LONG
    gpg --armor --export <ID>
    ```

    在本例中，GPG 密钥 ID 为 3AA5C34371567BD2：

    ```
    gpg --list-secret-keys --keyid-format LONG
    /Users/hubot/.gnupg/secring.gpg
    ------------------------------------
    sec   4096R/3AA5C34371567BD2 2016-03-10 [expires: 2017-03-10]
    uid                          Hubot 
    ssb   4096R/42B317FD4BA89E7A 2016-03-10
    ```

1.  使用 `gpg --export` 的输出，并按照 [`mng.bz/0mlx`](http://mng.bz/0mlx) 中描述的步骤将密钥添加到您的 GitHub 账户。

1.  配置 Git 使用生成的 GPG 密钥：

    ```
    git config --global user.signingkey <ID>
    ```
