# Домашнее задание к занятию «Организация сети»

> Репозиторий: hw-43\
> Выполнил: Асадбек Асадбеков\
> Дата: октябрь 2025

---

## Задание 1. Yandex Cloud

### Что нужно сделать

1. **Создать пустую VPC**  
   - Выбрать зону.

2. **Публичная подсеть**  
   - Создать subnet `public` — `192.168.10.0/24`  
   - Создать NAT-инстанс с адресом `192.168.10.254`, образ `fd80mrhj8fl2oe87o4e1`  
   - Создать виртуалку с публичным IP и проверить доступ в интернет  

3. **Приватная подсеть**  
   - Создать subnet `private` — `192.168.20.0/24`  
   - Создать route table с маршрутом на NAT-инстанс  
   - Создать VM в приватной сети и проверить доступ в интернет через NAT

---

## Выполненные задачи

### 1. Создание VPC
- Создана VPC сеть **`my-vpc`**
- Зона: **`ru-central1-a`**
- Подсети:
  - `public` → `192.168.10.0/24`
  - `private` → `192.168.20.0/24`

### 2. Публичная подсеть
- Создан NAT-инстанс `192.168.10.254`
- ВМ `public-vm` с публичным IP  
- Проверен доступ в интернет

### 3. Приватная подсеть
- Создана Route Table с маршрутом через NAT  
- Создана ВМ `private-vm` (внутренний IP)  
- Проверен интернет-доступ через NAT

---

## Архитектура решения

```
Интернет
   ↑
[NAT] (192.168.10.254, Pub IP: 158.160.113.142)
   ↑
[Pub Sub] (192.168.10.0/24)
 ├── [public-vm] (192.168.10.28, Pub IP: 51.250.76.37)
 └── [Pub Sub] (192.168.20.0/24)
      └── [private-vm] (192.168.20.24)
```

---

## Файлы конфигурации

<details>
<summary>main.tf</summary>

```hcl
terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.98"
    }
  }
}

provider "yandex" {
  cloud_id                 = var.cloud_id
  folder_id                = var.folder_id
  zone                     = var.zone
  service_account_key_file = var.service_account_key_file
}

resource "yandex_vpc_network" "network" {
  name = "my-vpc"
}

resource "yandex_vpc_subnet" "public" {
  name           = "public"
  zone           = var.zone
  network_id     = yandex_vpc_network.network.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_vpc_route_table" "private-rt" {
  name       = "private-route-table"
  network_id = yandex_vpc_network.network.id

  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = "192.168.10.254"
  }
}

resource "yandex_vpc_subnet" "private" {
  name           = "private"
  zone           = var.zone
  network_id     = yandex_vpc_network.network.id
  v4_cidr_blocks = ["192.168.20.0/24"]
  route_table_id = yandex_vpc_route_table.private-rt.id
}

resource "yandex_compute_instance" "nat-instance" {
  name        = "nat-instance"
  platform_id = "standard-v2"
  zone        = var.zone

  resources {
    cores  = 2
    memory = 2 
  }

  boot_disk {
    initialize_params {
      image_id = "fd80u2bttn91akcs5sgi"  
      size     = 15
    }
  }

  network_interface {
    subnet_id  = yandex_vpc_subnet.public.id
    ip_address = "192.168.10.254" 
    nat        = true              
  }

  metadata = {
    user-data = "#cloud-config\nusers:\n  - name: ubuntu\n    groups: sudo\n    shell: /bin/bash\n    sudo: ['ALL=(ALL) NOPASSWD:ALL']\n    ssh_authorized_keys:\n      - ${file("~/.ssh/id_rsa.pub")}"
  }
}

resource "yandex_compute_instance" "public-vm" {
  name        = "public-vm"
  platform_id = "standard-v2"
  zone        = var.zone

  resources {
    cores  = 2
    memory = 2  
  }

  boot_disk {
    initialize_params {
      image_id = "fd80u2bttn91akcs5sgi"  
      size     = 15
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    nat       = true
  }

  metadata = {
    user-data = "#cloud-config\nusers:\n  - name: ubuntu\n    groups: sudo\n    shell: /bin/bash\n    sudo: ['ALL=(ALL) NOPASSWD:ALL']\n    ssh_authorized_keys:\n      - ${file("~/.ssh/id_rsa.pub")}"
  }
}

resource "yandex_compute_instance" "private-vm" {
  name        = "private-vm"
  platform_id = "standard-v2"
  zone        = var.zone

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd80u2bttn91akcs5sgi"  
      size     = 15
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.private.id
  }

  metadata = {
    user-data = "#cloud-config\nusers:\n  - name: ubuntu\n    groups: sudo\n    shell: /bin/bash\n    sudo: ['ALL=(ALL) NOPASSWD:ALL']\n    ssh_authorized_keys:\n      - ${file("~/.ssh/id_rsa.pub")}"
  }
}
```
</details>

<details>
<summary>variables.tf</summary>

```hcl
variable "cloud_id" {
  description = "Yandex Cloud ID"
  type        = string
}

variable "folder_id" {
  description = "Yandex Cloud Folder ID"
  type        = string
}

variable "zone" {
  description = "Yandex Cloud Zone"
  type        = string
  default     = "ru-central1-a"
}

variable "service_account_key_file" {
  description = "Path to service account key file"
  type        = string
  default     = "key.json"
}
```
</details>

<details>
<summary>terraform.tfvars</summary>

```hcl
cloud_id  = "b1gsj7sfde79kl5qkpbl"
folder_id = "b1gm0hnoge59gnkmh3dl"
zone      = "ru-central1-a"
```
</details>

---
### 5. Проблема с образом

Возникла проблема с образом:

```hcl
image_id = "fd80mrhj8fl2oe87o4e1"
```
*Этот образ не работает с SSH, поэтому не удавалось подключиться к созданным инстансам. Вместо него был использован рабочий и актуальный образ*:

```hcl
image_id = "fd80u2bttn91akcs5sgi"
# ubuntu-20-04-lts-v20251006 (самый свежий)
```

После смены образа все инстансы стали доступны по SSH. 

## Команды для развертывания

<details>
<summary>Показать команды</summary>

```bash
terraform init
terraform plan
terraform apply
```
</details>

---

## Проверка работы

### Public VM (прямой доступ в интернет)
<details>
<summary>Показать</summary>

```bash
ssh ubuntu@51.250.76.37
ping ya.ru
```
</details>

### Private VM (доступ через NAT)
<details>
<summary>Показать</summary>

```bash
ssh -J ubuntu@51.250.76.37 ubuntu@192.168.20.24
ping ya.ru
curl ifconfig.me   # 51.250.90.182 (IP NAT-инстанса)
```
</details>

---

## Результат

- Создана сеть с публичной и приватной подсетями  
- NAT-инстанс обеспечивает интернет-доступ для приватной сети  
- Проверен доступ к интернету из обеих подсетей  
- Архитектура полностью соответствует требованиям задания

---

## Скриншоты

| № | Описание | Изображение |
|---|-----------|-------------|
| 1 | Развертывание инфраструктуры через Terraform | ![Terraform Apply](https://github.com/asad-bekov/hw-43/blob/main/img/1.PNG) |
| 2 | Тестирование подключения публичного инстанса | ![Public Instance Test](https://github.com/asad-bekov/hw-43/blob/main/img/2.PNG) |
| 3 | Конфигурация NAT Gateway | ![NAT Configuration](https://github.com/asad-bekov/hw-43/blob/main/img/3.PNG) |
| 4 | Доступ к приватному инстансу через Bastion | ![Bastion Access](https://github.com/asad-bekov/hw-43/blob/main/img/4.PNG) |
| 5 | Проверка сети на приватном инстансе | ![Private Instance Network](https://github.com/asad-bekov/hw-43/blob/main/img/5.PNG) |

---
