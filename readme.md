# Terraform


## **AWS-user.conf**
AWS-user.conf Tem por finalidade a configuração e padronização de usuário que terão acesso aos diferentes recursos do projeto. 

### **Exemplo de uso:**
```
#cloud-config
users:
  - default
  - name: accenture
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDjfSiSsd7w6zCaaCnyV4MylQHAa8AszPOnJ5M9...*
    groups: sudo
    shell: /bin/bash
```

### *Referência do argumento*
* `name` - O nome de login do usuário

* `sudo` - *ALL=(ALL) NOPASSWD:ALL* (permite acesso irrestrito ao sudo de um usuário).

* `SSH_authorized_keys` - (adicionar chaves ao arquivo de chaves autorizadas do usuário).

   * *O restante da ssh-key foi retirado por questões de segurança.
---
## **Datasource.tf**
Fontes de dados para acessar as propriedades dos grupo de recursos do Azure.

### *Data source dos Resources Groups*

### **Exemplo de uso:**
```
data "azurerm_resource_group" "rg-5g" {
  name = "rg-lab5g-${var.resource_env}brsouth"
}


data "azurerm_resource_group" "rg-hub" {
  name = "rg-hub-brsouth"
}

data "azurerm_resource_group" "rg-monitor" {
  name = "rg-tools-brsouth"
}
```
### *Referência do argumento*
* `name` - (obrigatório) Específica o nome do grupo de recursos.

### **Datasource da chave ssh a ser usada nas VMs Azure e VMs AWS**

### **Exemplo de uso**
```
data "azurerm_ssh_public_key" "chave_ssh" {
  name                = "key-lab5g-brsouth"
  resource_group_name = "rg-management-brsouth"
}

data "aws_key_pair" "chave_ssh_aws" {
  key_name          = "key-lab5g-brsouth"
}
```
---

## **fw-rules.tf**
O **fw-rules.tf** será quem criará uma coleção de regras **(D)NAT** em um *Firewall* ao Azure para services-access(serviços de acesso) conforme a necessidade de uso, para *streaming*, *api-core*, *api-gnodeb-1*, etc.
### **Exemplo de uso**
```
#Cria collection com regra de DNAT para o dashboard

resource "random_integer" "ri" {
  min = 001
  max = 999
}

resource "azurerm_firewall_nat_rule_collection" "services-access" {
  name                = "public-access"
  azure_firewall_name = data.azurerm_firewall.hubfw.name
  resource_group_name = data.azurerm_resource_group.rg-hub.name
  priority            = 1000
  action              = "Dnat"

  rule {
    name = "dashboard"

    source_addresses = [
      "*",
    ]

    destination_ports = [
      "3000"
    ]

    destination_addresses = [
      data.azurerm_public_ip.dashboard-pip.ip_address
    ]

    translated_port = 3000

    translated_address = azurerm_linux_virtual_machine.vm_monitor[0].private_ip_address

    protocols = [
      "TCP",
    ]
  }
}
rule {
    name = "webui"

    source_addresses = [
      "*",
    ]

    destination_ports = [
      "3001"
    ]

    destination_addresses = [
      data.azurerm_public_ip.dashboard-pip.ip_address
    ]

    translated_port = 3000

    translated_address = azurerm_linux_virtual_machine.vm_core[0].private_ip_address

    protocols = [
      "TCP",
    ]
  }

  rule {
    name = "streaming"

    source_addresses = [
      "*",
    ]

    destination_ports = [
      "8554"
    ]

    destination_addresses = [
      data.azurerm_public_ip.dashboard-pip.ip_address
    ]

    translated_port = 8554

    translated_address = azurerm_linux_virtual_machine.vm_upf[0].private_ip_address

    protocols = [
      "TCP",
    ]
  }
...
```

### *Referência do argumento*

* `name` - *public-access* (Obrigatório específica o nome da coleção de Regras NAT que deve ser exclusiva no Firewall).
* `Azure_firewall _name` - *data.azurerm_firewall.hubfw.name* ( Obrigatório específica o nome do Firewall no qual a coleção de regras deve ser criada).
* `Resource_group_name` - *data.azurerm_resource_group.rg-hub.name* ( Obrigatório específica o nome do Grupo de Recursos no qual o Firewall existe).
* `Priority` - *1000* ( Obrigatório específica a prioridade da coleção de regras, os valores possíveis estão entre 100 - 65000).
* `Action` - *Dnat* ( Obrigatório específica a ação que a regra aplicará ao tráfego correspondente. Os valores possíveis são DNAT e SNAT).

---
## **main.tf**
O **main.tf** irá ser o responsável em *criar* e *configurar* as instancias referente a "estrutura" do 5G. Isto é, irá criar as instancias do Core (core, gnodeb, ue, upf e monitor) e obter metadados para gerar inventário destas. 

### **Exemplo de uso**
>O código para criação do core 

```
locals {
  instance_core_outputs = [
    for i, instance in var.instances_core :
    {
      "instance_name" : instance.instance_name
      "instance_private_ip" : azurerm_network_interface.nic_core[i].ip_configuration[0].private_ip_address
      #"instance_dns" : "acclab5g.${data.azurerm_resource_group.rg-5g.location}.cloudapp.azure.com"
      "instance_dns" : azurerm_network_interface.nic_core[i].ip_configuration[0].private_ip_address
      "instance_role" : instance.instance_role
      "ue_data" : instance.ue_data
    }
  ]

  core_instances = [for instance in local.instance_core_outputs : instance if instance.instance_role == "core"]

}
```

Logo após, iremos concatena os dados em uma lista contendo todos os outputs e gerar o inventário no Ansible.
### **Exemplos de uso**
>Concatenação dos dos dados
```
locals {
  instance_outputs = [
    concat(local.instance_core_outputs, local.instance_gnodeb_outputs, local.instance_ue_outputs, local.instance_upf_outputs, local.instance_monitor_outputs)
  ]
}
```

>Geração do inventário do ansible
```
resource "local_file" "ansible_inventory" {
  content = templatefile("templates/ansible_inventory.tmpl",
    {
      core_instances    = local.core_instances
      upf_instances     = local.upf_instances
      gnb_instances     = local.gnb_instances
      ue_instances      = local.ue_instances
      upf_instances     = local.upf_instances
      monitor_instances = local.monitor_instances
    }
  )
  filename = "../ansible/hosts.ini"
}

resource "local_file" "host_vars_ues" {
  count = length(local.ue_instances)
  content = templatefile("templates/host_vars.tmpl",
    {
      ue_list = [local.ue_instances[count.index]]
    }
  )
  filename = "../ansible/host_vars/${local.ue_instances[count.index].instance_name}.yaml"
}

resource "local_file" "host_vars_upfs" {
  count = length(local.upf_instances)
  content = templatefile("templates/upf_host_vars.tmpl",
    {
      upf_list = [local.upf_instances[count.index]]
    }
  )
  filename = "../ansible/host_vars/${local.upf_instances[count.index].instance_name}.yaml"
}

resource "local_file" "host_vars_cores" {
  count = length(local.core_instances)
  content = templatefile("templates/core_host_vars.tmpl",
    {
      ue_list  = local.ue_instances
      upf_list = local.upf_instances
    }
  )
  filename = "../ansible/host_vars/${local.core_instances[count.index].instance_name}.yaml"
}
```

---

## **Outputs.tf**




## **Provider.tf**
O **provider.tf** irá ser o responsável em *criar* e *configurar* as instancias da Azure e AWS.
### **Exemplo de uso**
```
provider "azurerm" {
  features {}
}

provider "aws" {
  region = "us-east-1"
}

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "2.99.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "4.17.1"
    }
  }
  backend "azurerm" {
    resource_group_name  = "rg-lab5g-dev-brsouth"
    storage_account_name = "stlab5gdevbrsouth"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"

  }
}

data "azurerm_client_config" "current" {}
```





  
