[English version](https://eugenio-chaves.github.io/blog/2022/creating-a-custom-wazuh-integration)

### O Motivo

No futuro, eu quero monitorar meu ambiente e minha rede doméstica com Wazuh integrado com Suricata e tudo isso funcionando em uma Raspberry pi 4. Também quero uma forma fácil para visualizar os alertas que o Wazuh irá gerar.

Eu poderia usar as integrações prontas do Wazuh, como a do [Slack](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-integrate-slack.html), por exemplo. Decidi usar o Discord porque é um aplicativo que uso praticamente o dia todo e um exemplo perfeito para demonstrar como criar uma integração customizada do Wazuh com apps externos.

É possível criar uma integração customizada para enviar alertas para qualquer lugar que tenha um webhook ou alguma forma de receber dados via POST requests. Como Microsoft Teams, por exemplo.

### O Ambiente

Eu estou assumindo que você já tem o Wazuh instalado e funcionando. No meu ambiente atual, estou com uma instalação "All in One" na minha VM kali, você pode seguir com qualquer distribuição Linux que quiser. Se você estiver usando o Wazuh distribuído em clusters, irá precisar replicar toda esta configuração nas managers onde você queira que a integração funcione.

### Criando o Webhook

Á primeira coisa a se fazer é criar um webhook no chat do **seu** servidor de Discord
```markdown
- Abra as configurações de seu servidor
- Na aba "Integrações", clica em "webhooks" e gere um
- Salve o link por enquanto
```

### Criando a Integração

Entre na pasta de integração do Wazuh. Você deve ver os scripts de integrações padrões do Wazuh 
```bash
cd /var/ossec/integrations 
```
![](/docs/assets/images/01.png)

Como você pode ver, existem dois scritps do slack, um em bash e outro em python, a razão para isso é porque o script em bash vai funcionar como um launcher para o script em python, que é o core da integração.

Copie os dois e passe o nome para **custom-discord**

- todas as integrações customizadas do Wazuh precisam que o nome inicie com **custom-**
```bash
cp slack custom-discord
cp slack.py custom-discord.py
```

The **Slack's** scripts were made by the Wazuh team to integrate with Slack via webhook, we can modify them to work with Discord webhook instead. Practically, the only modification needed for that to work is going to be the in the **slack.py** **generate_msg()** function.

Before we talk code, I think it's good to understand how an integration is triggered and how you can control what types of alerts will trigger it.

Open the configuration file of the Wazuh manager, **ossec.conf** 
```markdown
vim /var/ossec/etc/ossec.conf
```

This block of xml is what activates the integration, you will need to insert this on the configuration file of the Wazuh manager, **ossec.conf**. You can insert in any place you like, just be careful to not put in the middle of another block.
```xml
  <integration>
    <name>custom-discord</name>
    <hook_url>https://discord.com/api/webhooks/hook</hook_url>
    <level>7</level>
    <alert_format>json</alert_format>
  </integration>
``` 
![](/docs/assets/images/02.png)

- The tag **name** in the line 354 is where we define the name of the script that will launch the python script.
- The tag **hook** is where you will need to place your webhook URL.
- In the line 356 I can control what will trigger the integration, in my case is any Wazuh alert that is equal or greater than 07.

You can use the following options to trigger the alert:
```xml
<group>suricata,sysmon</group> Only the rules of the group suricata and sysmon will trigger the integration.
<level>12</level> Only rules greate or equal to 12 will trigger.
<rule_id>1299,1300</rule_id> Only this rules will trigger.
```

### Customizing The Script

After activating the integration, the next step would be customizing the custom-discord.py script.
open the script with your favorite text editor, in the line **76** you can replace the **generate_msg()** function with the bellow function.

This function will take an Wazuh alert as an argument and because the alerts are coming in json format we just need to fill the values to send to Discord. When I build this function, I used [Birdie0](https://github.com/Birdie0) repository to help me understand how discord webhooks worked.

If you want to modify the contents of the alert, you will need to modify the fields in the **payload**

I recommend that you check out Birdie repository [birdie0.github.io/discord-webhooks-guide/index.html](https://birdie0.github.io/discord-webhooks-guide/index.html) and explore to see if you will like something different for your alert.

A good tool to help you with that is **Postman** https://www.postman.com/

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