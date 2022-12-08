[Versão em Inglês](https://eugenio-chaves.github.io/blog/2022/)

## Wazuh API

### Beneficios da API

O Wazuh vem com uma API RESTful instalada em sua controladora, servindo na porta 55000. Além da comunicação interna com a Dashboard, um dos principais beneficios da API é coletar informações sobre multiplos agentes de uma forma conjunta, o que ainda não é possivel para a maiorias dos modulos do Wazuh através de sua dashboard.

O módulo "Packages" permite que você veja quais pacotes estão instalados no sistema operacional do agente.

![](/docs/assets/images/PacotesInterface.png)

O problema dessa interface seria a coleta de todos os agentes do ambiente, seria preciso consultar agente por agente. A API proporciona um método para fazer esta coleta de uma vez só. Para fazer isso, estarei usando um script em python, após a coleta, o script irá consolidar os dados de todos os agents em um arquivo csv.


### Como Funciona

A API proporciona diversos [endpoints](https://documentation.wazuh.com/current/user-manual/api/reference.html)

irei seguir o seguinte processo para a coleta com o script:

1. Mandar uma requisição para o endpoint **/agents** solicitando informações dos agentes e salvar a resposta em uma variável. Nesta informação estará inclusos dados importantes como o nome e id do agente.
2. A resposta será retornada no formato json, desta forma sera preciso manipular a resposta da API para podermos coletar o ID de agente retornado.
3. Criar um loop para passar por todos os IDs retornados pela API.
4. Para cada ID retornado, fazer uma nova chamada, mas desta vez para o endpoint desejado, neste caso o **/packages** passando o ID do agente, desta forma eu estarei solicitando informaçoes de pacotes de todos os agentes
5. Manipular a saida para ter a informação na tela ou salvar em um documento.


### Script de Coleta

O script está em Python, mas este processo pode ser feito com qualquer linguagem.
    
```python
#!/usr/bin/env python3
import json
from base64 import b64encode
from os.path import join
import csv
import requests  #pip3 install requests caso não tenha esta lib instalada
import urllib3


# Configuração da Manager do Wazuh
protocol = 'https'
host = '192.168.100.38'
port = '55000'
user = '' #usuário API
password = '' # senha API
output_filename = 'packages.csv'

# Variáveis
base_url = f'{protocol}://{host}:{port}'
login_url = f'{base_url}/security/user/authenticate'
basic_auth = f'{user}:{password}'.encode()
headers = {'Authorization': f'Basic {b64encode(basic_auth).decode()}'}

# Desabilitar avisos de conexão https insegura (para casos de certificados auto-assinados)
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)


# Fazer chamadas para os endpoints
def get_response(url, headers, verify=False):
    """Get API result"""
    request_result = requests.get(url, headers=headers, verify=verify)

    if request_result.status_code == 200:
        return json.loads(request_result.content.decode())
    else:
        raise Exception(f'Erro durante a requisição: {request_result.json()}')

# Gerar o documento csv
def write_csv(data):
    try:
        with open(join(output_filename), 'w', encoding="utf-8", newline='') as outfile:
            #Declarar cabeçalhos de csv
            writer = csv.DictWriter(outfile, fieldnames=['agent_name', 'scan_time', 'version','vendor','format','name','architecture','section','description','install_time','size'])
            writer.writeheader()
            for row in data:
                writer.writerow(row)
        print(f'\nReporte criado "{join(output_filename)}".')
    except Exception as e:
        print(f'Seguinte erro encontrado na criação do csv {join(output_filename)}: {e}. ')
        response = input('Deseja printar o reporte? [y/n]: ')
        if response == 'y':
            print(data)
            pass
        else:
            print('Saindo...')


def main():
    result = [] #salvar dados coletados da API
    headers['Authorization'] = f'Bearer {get_response(login_url, headers)["data"]["token"]}'

    # Request
    agents = get_response(base_url + '/agents?wait_for_complete=true&select=name&select=status&limit=100000', headers)
    if agents['data']['total_affected_items'] == 0:
        print(f'Nenhum agente encontrado: \n{agents}')
        exit(0)

    #Loop em todos os agentes
    for agent_data in agents['data']['affected_items']:
        if agent_data['status'] == 'never_connected':
            print(f'Agente "{agent_data["name"]}" esta com o status "never_connected" não sendo possível coletar seus pacotes.'
                  f'Skipping...')
            continue

        #Realizar a chamada no endpoint packages
        try:
            packages = get_response(
                base_url + f'/syscollector/{agent_data["id"]}/packages?limit=100000',
                headers
            )
        except Exception:
            print(f'Não foi possível coletar informações do agente {agent_data["name"]} ({agent_data["id"]}). '
                f'Skipping...')
            continue

        for package in packages['data']['affected_items']:
            if agent_data['status'] == 'never_connected':
                print(f'Agente "{agent_data["name"]} esta com o status "never_connected" não sendo possível coletar informações de seus pacotes')
                continue
   
            try:
                result.extend([{'agent_name': agent_data.get('name', 'unknown'),
                                'scan_time': package['scan'].get('time', 'unknown'),
                                'version': package.get('version', 'unknown'),
                                'vendor': package.get('vendor', 'unknown'),
                                'format': package.get('format', 'unknown'),
                                'name': package.get('name', 'unknown'),
                                'architecture': package.get('architecture', 'unknown'),
                                'section': package.get('section', 'unknown'),
                                'description': package.get('description', 'unknown'),
                                'install_time': package.get('install_time', 'unknown'),
                                'size': package.get('size', 'unknown')}                        
                            for package in packages['data']['affected_items']])
            except Exception as e:
                    print(f'Erro encontrado durante o parsing do agente "{agent_data["name"]}": {e}. Skipping...')
    else:
        pass
    write_csv(result)

if __name__ == '__main__':
    main()
```

### Conclusão

Como mencionado antes, esse mesmo método pode ser usado para varios endpoints da API do Wazuh, é muito util para coletar o de vulnerabilidades também.

[Top](https://eugenio-chaves.github.io/blog/2022/wazuh-api-packages)
```