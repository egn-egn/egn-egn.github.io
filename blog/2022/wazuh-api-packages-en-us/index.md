[Versão em Português](https://eugenio-chaves.github.io/blog/2022/wazuh-api-packages)

## Wazuh API - Packages

### Packages Module

Wazuh already has a RESTful API in the Manager, serving on port 55000. Besides the internal communication with the Dashboard, one of the main benefits of the API is to collect information about multiple agents all at once, which is not yet possible for most Wazuh modules through the dashboard.

The "Packages" module allows you to see which packages are installed on the agent's operating system.

![](/docs/assets/images/PacotesInterface.png)

The problem with this interface would be collecting package's information from all agents on the environment, you would have to query agent by agent. The API provides a method to do this collection all at once. To do this, I will be using a python script, after the collection, the script will consolidate the data from all the agents in a csv file.


### How The Script Will Work

The API has multiples [endpoints](https://documentation.wazuh.com/current/user-manual/api/reference.html)

I will follow the following process for collecting [Packages](https://documentation.wazuh.com/current/user-manual/api/reference.html#operation/api.controllers.syscollector_controller.get_packages_info) information with the script:

1. Send a request to the endpoint **/agents** asking for agent information and save the response in a variable. This information will include important data such as the agent's name and id.
2. The response will be returned in json format. (the script will manipulate the API response to collect the agent ID)
3. Create a loop to go through all the IDs returned by the API.
4. For each returned ID, make a new call, but this time to the desired endpoint, in this case the **/packages** passing the agent ID, this way I will be requesting package information from all agents
5. Manipulate the output to have the information on the screen or save it in a file.


### Script

The script is in Python, but this process can be done with any other language.
    
```python
#!/usr/bin/env python3
import json
from base64 import b64encode
from os.path import join
import csv
import requests  #pip3 install requests
import urllib3


# Configurating the manager variables
protocol = 'https'
host = '192.168.100.38'
port = '55000'
user = '' #API user
password = '' #API password
output_filename = 'packages.csv'

# Auth Variables
base_url = f'{protocol}://{host}:{port}'
login_url = f'{base_url}/security/user/authenticate'
basic_auth = f'{user}:{password}'.encode()
headers = {'Authorization': f'Basic {b64encode(basic_auth).decode()}'}

# Disable https warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)


# Function to send requests
def get_response(url, headers, verify=False):
    """Get API result"""
    request_result = requests.get(url, headers=headers, verify=verify)

    if request_result.status_code == 200:
        return json.loads(request_result.content.decode())
    else:
        raise Exception(f'Error during request: {request_result.json()}')

# Function to create a csv file
def write_csv(data):
    try:
        with open(join(output_filename), 'w', encoding="utf-8", newline='') as outfile:
            #Here we can declare the csv headers
            writer = csv.DictWriter(outfile, fieldnames=['agent_name', 'scan_time', 'version','vendor','format','name','architecture','section','description','install_time','size'])
            writer.writeheader()
            for row in data:
                writer.writerow(row)
        print(f'\n "{join(output_filename)}".')
    except Exception as e:
        print(f'Found the following error during file creation {join(output_filename)}: {e}. ')
        response = input('Want to print the Report? [y/n]: ')
        if response == 'y':
            print(data)
            pass
        else:
            print('Exiting...')


def main():
    result = [] #variable to temporarily save the API response
    headers['Authorization'] = f'Bearer {get_response(login_url, headers)["data"]["token"]}'

    # Request
    agents = get_response(base_url + '/agents?wait_for_complete=true&select=name&select=status&limit=100000', headers)
    if agents['data']['total_affected_items'] == 0:
        print(f'No agents found: \n{agents}')
        exit(0)

    #Loop in all agents
    for agent_data in agents['data']['affected_items']:
        if agent_data['status'] == 'never_connected':
            print(f'Agent "{agent_data["name"]}" is with the status "never_connected".'
                  f'Skipping...')
            continue

        #Calling the Packages endpoint
        try:
            packages = get_response(
                base_url + f'/syscollector/{agent_data["id"]}/packages?limit=100000',
                headers
            )
        except Exception:
            print(f'Error collecting data for the following agent {agent_data["name"]} ({agent_data["id"]}). '
                f'Skipping...')
            continue

        for package in packages['data']['affected_items']:
            if agent_data['status'] == 'never_connected':
                print(f'Agent "{agent_data["name"]} is with the status "never_connected".')
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
                    print(f'Error during the parsing for the following "{agent_data["name"]}": {e}. Skipping...')
    else:
        pass
    write_csv(result)

if __name__ == '__main__':
    main()
```

### Conclusion

As mentioned before, this same method can be used for several Wazuh API endpoints, and is very useful for collecting vulnerability information as well.

[Top](https://eugenio-chaves.github.io/blog/2022/wazuh-api-packages)