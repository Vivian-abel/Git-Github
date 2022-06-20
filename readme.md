# Terraform


## **AWS-user.conf**
AWS-user.conf Tem por finalidade a configuração de usuários que vão........

### **Exemplo de uso:**
```
#cloud-config
users:
  - default
  - name: accenture
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDjfSiSsd7w6zCaaCnyV4MylQHAa8AszPOnJ5M9n6W6HRkydm+cdpke1ZtPbHuX0ERy8K1C2qhw3yrSMtgVdt/YJzMnCLoFgiQCrYsmsuFq37bOBb7NNaGN3A28Lhjix483BhXFKw23MuHLan6vpYUx8OQ0A5kdhYZFjoagzgKmpMekgkdSckxGoWQM3Zw3/EJEKT445w3mrI7phrWO8avTenOm4CoXGw3nYPYrd6l7SFcDg5Fbz7rvFYSlKjBbUwNvKW339/Hs+RYd714VIws1hz4BLpJMxsDzXA7Mh1dWYn/241uAq6wMpTnN+CmxNYOmy3kkWZ6Tw16PN/BVjTaPFq/kz2D8GIp6aljiNv6odWQuzWoqoBMiGIuxytTTgAUPGAwv8XEOrphW9zk6mpmr9yR4Vxx6k8bc6ylyUUCgOgDgc4hBeUx00cJ5OUloMwX90dUX0UHBs9KqzYPzNnPnDWJly6L4bQejMkEPrBEOwfm7LvTz1oU+bz95S+uQEfE= generated-by-azure
    groups: sudo
    shell: /bin/bash
```
### *Referência do argumento*
`name` - O nome de login do usuário

`sudo` - ALL=(ALL) NOPASSWD:ALL (permite acesso irrestrito ao sudo de um usuário).

`SSH_authorized_keys` - (adicionar chaves ao arquivo de chaves autorizadas do usuário).
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
`name` - (obrigatório) Específica o nome do grupo de recursos.

## **Datasource da chave ssh a ser usada nas VMs Azure e VMs AWS**

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






  
