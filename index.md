## Why? // Por que?

By default, Wazuh send his alerts via e-mail, and has some ready to go integrations, [Slack](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-integrate-slack.html) is one of them.
In my case, i use Wazuh to monitor my home network and discord is an app that i'm using all day, it's the perfect place for me see the alerts. But it is possible to create a custom-integration to send alerts to pratically any place that has a webhook or any other form of receiving data via POST requests. Like Teams for example :).

Por default, o Wazuh envia seus alertas via e-mail e atualmente, na versão 4.3.0, tem integrações prontas, uma delas é o [Slack](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-integrate-slack.html).
No meu caso, eu uso o Wazuh para monitorar meu ambiente pessoal e como o discord é uma aplicação que uso praticamente o dia inteiro, nada mais justo do que enviar os alertas para ele. Mas é possivel criar uma integração para enviar alertas para qualquer lugar/serviço que tenha um webhook, como o Teams por exemplo.

### Creating a webhook // Criando um webhook

First thing to do is create a discord webhhok
```markdown
- Open your server settings // Abra as configurações de seu servidor
- On "Integrations", click on "webhooks" and generate one. // Na aba "Integrations", clica no "webhooks" para gerar um
- Save the webhook link for now // Guarde o link do webhook por enquanto

```

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/egn-egn/egn-egn.github.io/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
