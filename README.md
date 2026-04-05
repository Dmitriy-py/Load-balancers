# Домашнее задание к занятию «Вычислительные мощности. Балансировщики нагрузки»

## ` Дмитрий Климов `

## Задание 1. Yandex Cloud

### Что нужно сделать

---

1. Создать бакет Object Storage и разместить в нём файл с картинкой:
   * Создать бакет в Object Storage с произвольным именем (например, имя_студента_дата).
   * Положить в бакет файл с картинкой.
   * Сделать файл доступным из интернета.
2. Создать группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на            картинку из бакета:
   * Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать image_id =                          fd827b91d99psvq5fjit.
   * Для создания стартовой веб-страницы рекомендуется использовать раздел user_data в meta_data.
   * Разместить в стартовой веб-странице шаблонной ВМ ссылку на картинку из бакета.
   * Настроить проверку состояния ВМ.
3. Подключить группу к сетевому балансировщику:
   * Создать сетевой балансировщик.
   * Проверить работоспособность, удалив одну или несколько ВМ.
4. (дополнительно)* Создать Application Load Balancer с использованием Instance group и проверкой состояния.

---

## Ответ:

## Описание

### Цель задания — развернуть отказоустойчивую инфраструктуру в Yandex Cloud с использованием Terraform, которая включает:

---

  * Бакет Object Storage с публично доступной картинкой.
  * Группу виртуальных машин с шаблоном LAMP.
  * Сетевой балансировщик (NLB), распределяющий трафик между ВМ.
  * Настроенный Health Check для ВМ.
  * Динамическое отображение имени сервера и ссылки на картинку на веб-странице.

---

### Структура проекта

```bash
.
├── instance_group.tf
├── load_balancer.tf
├── main.tf
├── s3.tf
├── terraform.tfvars
├── variables.tf
├── image.png
└── README.md
```

---

### 1. Файл `variables.tf`

```Hcl
variable "yc_token" {
  type        = string
}

variable "yc_cloud_id" {
  type        = string
}

variable "yc_folder_id" {
  type        = string
}

variable "yc_zone" {
  type        = string
  default     = "ru-central1-a"
}

variable "bucket_name" {
  type        = string
  default     = "netology-bucket-student-unique-id"
}

variable "lamp_image_id" {
  type        = string
  default     = "fd827b91d99psvq5fjit"
}
```

---

### 2. Файл `main.tf`

```Hcl
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  token     = var.yc_token
  cloud_id  = var.yc_cloud_id
  folder_id = var.yc_folder_id
  zone      = var.yc_zone
}

resource "yandex_vpc_network" "net" {
  name = "netology-network"
}

resource "yandex_vpc_subnet" "subnet" {
  name           = "public-subnet"
  zone           = var.yc_zone
  network_id     = yandex_vpc_network.net.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_iam_service_account" "sa" {
  name = "ig-sa"
}

resource "yandex_resourcemanager_folder_iam_member" "sa_editor" {
  folder_id = var.yc_folder_id
  role      = "editor"
  member    = "serviceAccount:${yandex_iam_service_account.sa.id}"
}

resource "yandex_resourcemanager_folder_iam_member" "sa_storage" {
  folder_id = var.yc_folder_id
  role      = "storage.admin"
  member    = "serviceAccount:${yandex_iam_service_account.sa.id}"
}

resource "yandex_iam_service_account_static_access_key" "sa_key" {
  service_account_id = yandex_iam_service_account.sa.id
}
```

---

### 3. Файл `s3.tf`
```Hcl
resource "yandex_storage_bucket" "img_bucket" {
  access_key = yandex_iam_service_account_static_access_key.sa_key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa_key.secret_key
  bucket     = var.bucket_name

  depends_on = [yandex_resourcemanager_folder_iam_member.sa_storage]

  anonymous_access_flags {
    read = true
    list = false
  }
}

resource "yandex_storage_object" "picture" {
  access_key = yandex_iam_service_account_static_access_key.sa_key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa_key.secret_key
  bucket     = yandex_storage_bucket.img_bucket.id
  key        = "image.png"
  source     = "image.png"
  acl        = "public-read"
}
```

---

### 4. Файл `instance_group.tf`

```Hcl
resource "yandex_compute_instance_group" "lamp_group" {
  name               = "lamp-group"
  folder_id          = var.yc_folder_id
  service_account_id = yandex_iam_service_account.sa.id
  
  depends_on = [
    yandex_resourcemanager_folder_iam_member.sa_editor,
    yandex_storage_object.picture
  ]

  instance_template {
    platform_id = "standard-v1"
    resources {
      memory = 2
      cores  = 2
    }

    boot_disk {
      initialize_params {
        image_id = var.lamp_image_id
      }
    }

    network_interface {
      network_id = yandex_vpc_network.net.id
      subnet_ids = [yandex_vpc_subnet.subnet.id]
      nat        = true
    }

    metadata = {
      ssh-keys  = "ubuntu:${file("~/.ssh/id_rsa.pub")}" 
      user-data = <<-EOF
#cloud-config
write_files:
  - content: |
      <html>
        <head><title>Netology Task</title></head>
        <body>
          <h1>Hello! I am server: ###HOSTNAME###</h1>
          <img src="" width="600">
        </body>
      </html>
    path: /var/www/html/index.html
runcmd:
  - sed -i "s|###HOSTNAME###|$$(hostname)|g" /var/www/html/index.html
EOF
    }
  }

  scale_policy {
    fixed_scale {
      size = 3
    }
  }

  allocation_policy {
    zones = [var.yc_zone]
  }

  deploy_policy {
    max_unavailable = 1
    max_creating    = 3
    max_expansion   = 1
    max_deleting    = 1
  }

  health_check {
    interval            = 2
    timeout             = 1
    unhealthy_threshold = 2
    healthy_threshold   = 2
    http_options {
      path = "/"
      port = 80
    }
  }

  load_balancer {
    target_group_name = "lamp-tg"
  }
}
```

---

### 5. Файл `load_balancer.tf`
```Hcl
resource "yandex_lb_network_load_balancer" "nlb" {
  name = "lamp-nlb" # Имя балансировщика

  listener {
    name = "http-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_compute_instance_group.lamp_group.load_balancer[0].target_group_id
    healthcheck {
      name = "http-check"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

output "load_balancer_public_ip" {
  value = [for s in yandex_lb_network_load_balancer.nlb.listener: s.external_address_spec[*].address]
}
```

---

### 6. Файл `terraform.tfvars`
```Hcl
yc_token     = "token" 
yc_cloud_id  = "cloud_id" 
yc_folder_id = "folder_id" 
bucket_name  = "netology-myname-bucket-20231027"
```

---

### 7. Файл `.gitignore`
```gitignore
*.tfstate
*.tfstate.*
*.tfvars
.terraform/
.terraform.lock.hcl
```

---

<img width="1920" height="1080" alt="Снимок экрана (3250)" src="https://github.com/user-attachments/assets/98152f72-5374-4ee8-b894-59908fbf3bde" />

<img width="1920" height="1080" alt="Снимок экрана (3264)" src="https://github.com/user-attachments/assets/c969be6d-7b10-4b59-9fce-d09cae95df43" />

<img width="1920" height="1080" alt="Снимок экрана (3258)" src="https://github.com/user-attachments/assets/6c420d00-9584-4ac8-bb22-ec5b4e4e9ffb" />

<img width="1920" height="1080" alt="Снимок экрана (3259)" src="https://github.com/user-attachments/assets/c66df6bc-9181-4564-9b61-deee8cbcafe8" />

<img width="1920" height="1080" alt="Снимок экрана (3260)" src="https://github.com/user-attachments/assets/93689972-a15c-4e90-a650-67f3b8767155" />

<img width="1920" height="1080" alt="Снимок экрана (3261)" src="https://github.com/user-attachments/assets/ab058941-b974-4f75-aea5-b514b9033e20" />

<img width="1920" height="1080" alt="Снимок экрана (3262)" src="https://github.com/user-attachments/assets/bba1b610-6da8-4ff8-8ccd-7a65bdbe46e9" />

<img width="1920" height="1080" alt="Снимок экрана (3263)" src="https://github.com/user-attachments/assets/ed0defbd-a519-4ad0-a4ee-d5b75c5b002b" />

<img width="1920" height="1080" alt="Снимок экрана (3257)" src="https://github.com/user-attachments/assets/635ef020-60fa-4348-8bee-044b26b17532" />















