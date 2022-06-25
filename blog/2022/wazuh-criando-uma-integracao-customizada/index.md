[English version](https://eugenio-chaves.github.io/blog/2022/creating-a-custom-wazuh-integration)

### O Motivo

No futuro, eu quero monitorar meu ambiente e minha rede doméstica com Wazuh integrado com Suricata e tudo isso funcionando em uma Raspberry pi 4. Também quero uma forma fácil para visualizar os alertas que o Wazuh irá gerar.

Eu poderia usar as integrações prontas do Wazuh, como a do [Slack](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-integrate-slack.html), por exemplo. Decidi usar o Discord porque é um aplicativo que uso praticamente o dia todo e um exemplo perfeito para demonstrar como criar uma integração customizada do Wazuh com apps externos.

É possível criar uma integração customizada para enviar alertas para qualquer lugar que tenha um webhook ou alguma forma de receber dados via POST requests. Como Microsoft Teams, por exemplo.

### O Ambiente

Eu estou assumindo que você já tem o Wazuh instalado e funcionando. No meu ambiente atual, estou com uma instalação "All in One" na minha VM kali, você pode seguir com qualquer distribuição Linux que quiser. Se você estiver usando o Wazuh distribuído em clusters, irá precisar replicar toda esta configuração nas managers onde você queira que a integração funcione.

### Criando o Webhook

A primeira coisa a se fazer é criar um webhook no chat do **seu** servidor de Discord
```markdown
- Abra as configurações de seu servidor
- Na aba "Integrações", clique em "webhooks" para gerar um
- Salve o link por enquanto
```

### Criando a Integração

Entre na pasta de integração do Wazuh. Você deve ver os scripts de integrações padrões
```bash
cd /var/ossec/integrations 
```
![](/docs/assets/images/01.png)

Como você pode ver, existem dois scritps do slack, um em bash e outro em python, a razão para isso é porque o script em bash vai funcionar como um launcher para o script em python, que é o core da integração.

Copie os dois e passe o nome para **custom-discord**

- Todas as integrações customizadas do Wazuh precisam que o nome inicie com **custom-**
```bash
cp slack custom-discord
cp slack.py custom-discord.py
```

Os scritps do **Slack** foram feitos pelo time do Wazuh, para integrar com um canal de Slack via webhook, podemos modificar o script em python para integrar com o Discord. Para isso, a única modificação que será preciso fazer é na função **generate_msg()** dentro do script **slack.py**.

Antes de começar, acredito que seria interessante demonstrar como uma integração é chamada e como você pode controlar que tipo de alertar iram iniciar ela.

Abra o arquivo de configuração da manager do Wazuh **ossec.conf**
```markdown
vim /var/ossec/etc/ossec.conf
```

Esse bloco de xml é o que ativa a integração, você vai precisar colocar esse bloco no arquivo de configuração da manager do Wazuh, **ossec.conf**. Você pode colocar em qualquer posição dentro do arquivo, só tome cuidado para não inserir no meio de outro bloco.
```xml
  <integration>
    <name>custom-discord</name>
    <hook_url>https://discord.com/api/webhooks/hook</hook_url>
    <level>7</level>
    <alert_format>json</alert_format>
  </integration>
``` 
![](/docs/assets/images/02.png)

- A Tag **name** na linha 354 é onde você define o nome do arquivo que irá executar o script em python.
- A Tag **hook** é onde você deve inserir seu webhook.
- Na linha 356 você pode controlar qual condição irá chamar a integração, no meu caso, qualquer alerta do Wazuh onde o nível seja maior o igual a 07.

Você pode usar as seguintes condições para triggerar a integração:
```xml
<group>suricata,sysmon</group> Only the rules of the group suricata and sysmon will trigger the integration.
<level>12</level> Only rules greate or equal to 12 will trigger.
<rule_id>1299,1300</rule_id> Only this rules will trigger.
```

### Customizando o Script

Após ativar a integração, o próximo passo seria customizar o script custom-discord.py

Abra o script com seu editor de texto favorito, na linha **76** você deve substituir a função **generate_msg()** do script com a abaixo;

Essa função irá pegar um alerta do Wazuh com argumento e como os alertas estão chegando em formato json, tudo que precisa ser feito é preencher os valores das chaves dentro do dicionário da função para o python transformar em json e enviar para o webhook do Discord.

Quando eu construí esse payload, usei o repositório do [Birdie0](https://github.com/Birdie0) para me ajudar a entender como customizar o que envio para o webhook do Discord.

Eu recomendo que você cheque o repositório do Birdie [birdie0.github.io/discord-webhooks-guide/index.html](https://birdie0.github.io/discord-webhooks-guide/index.html) e tente achar se tem algum formato diferente que você gostaria de usar em seu alerta.

Uma boa ferramenta para ajudar com isto é o **Postman** https://www.postman.com/
```python
def generate_msg(alert):
	#save the rule level
    level = alert['rule']['level']
    #compare rules level to set colors of the alert
    if (level <= 4):
    	#green
        color = "3731970"
    elif (level >= 5 and level <= 12):
        #yellow
        color = "15919874"
    else:
        #red
        color = "15870466"

    if 'agentless' in alert:
        agent_ = 'agentless'
    else:
        agent_ = alert['agent']['name']
    #data that the webhook will receive and use to display the alert in discord chat
    payload = json.dumps({
      "embeds": [
        {
          "title": "Wazuh Alert - Rule {}".format(alert['rule']['id']),
          "color": "{}".format(color),
          "description": "{}".format(alert['rule']['description']),
          "fields": [
            {
              "name": "Agent",
              "value": "{}".format(agent_),
              "inline": True
            },
            {
              "name": "Location",
              "value": "{}".format(alert['location']),
              "inline": True
            },
            {
            "name": "Rule Level",
            "value": "{}".format(alert['rule']['level']),
            "inline": True
            }
          ]
        }
      ]
    })

    return payload
```
This is the alert format:
![](/docs/assets/images/03.png)

I like to change the **debug()** function too, so I can control more freely when to save debug logs.

The default path for integrations log is in.
```
/var/ossec/logs/integrations.log
```
You can control if you want logs or not with the variable **deb**
```python
def debug(msg):
        # debug log
        deb = True
        if deb == True:
            msg = "{0}: {1}\n".format(now, msg)
            print(msg)
            f = open(log_file, "a")
            f.write(msg)
            f.close()
```
You will need to chance the scripts permissions and owners:
```bash
chmod 750 custom-*
chown root:wazuh custom-*
```
You will need the requests library, install it if you don't have it.
```
pip3 install requests
```

### Creating A Custom Rule

Everything should be ready now, but before we restart the Wazuh manager to activate the integration, I like to create a custom rule that I can manually trigger to test the integration.

Create a file in /var/log named test.log
```bash
touch /var/log/test.log
```
Open the ossec.conf file again and navigate to the bottom, you should see a lot of **localfile** blocks, they indicate a path to a file that Wazuh will collect his logs from.

Insert this block:
```xml
  <localfile>
    <location>/var/log/test.log</location>
    <log_format>syslog</log_format>
  </localfile>
```
Next, create a custom rule, you will need to open the file designed for that **local_rules.xml**
```bash
vim /var/ossec/etc/rules/local_rules.xml
```
Insert this rule:
```xml
  <rule id="119999" level="12">
    <regex>^test$</regex>
    <description>Test rule to configure integration</description>
  </rule>
```
Now every time that you will echo the word **test** in **/var/log/test.log** the rule should trigger.
To test that we can use the Binary that is exclusively for that, **wazuh-logtest**

Run the binary and type the word test, you should see your rule getting initiated.
```markdown
/var/ossec/bin/wazuh-logtest
```
![](/docs/assets/images/04.png)

Restart the manager and trigger the alert, you should receive the alert on your Discord channel.
```bash
/var/ossec/bin/wazuh-control restart
echo -e "test" /var/log/test.log
```
The alert in Discord:

![](/docs/assets/images/05.png)

### Debugging

If you encounter problems, the files to look at are:
- /var/ossec/logs/ossec.log
- /var/ossec/logs/integration.log

You can make the [integrator](https://documentation.wazuh.com/current/user-manual/reference/daemons/wazuh-integratord.html) daemon more verbose, for that execute with the flag **-d** or **-dd**
```bash
/var/ossec/bin/wazuh-integratord  -d
```
After that, the logs in ossec.log should be more verbose.

### Conclusion

First I would like to offer my thanks to Alexandre Borges [@ale_sp_brazil](https://twitter.com/ale_sp_brazil) for incentivizing me to start writing this blog. You should definitely check his blog [exploitreversing.com](https://exploitreversing.com/)

When I first needed to create a custom integration, I did not find much material talking about it, I hope that this can help someone.

[Top](https://eugenio-chaves.github.io/blog/2022/wazuh-criando-uma-integracao-customizada)