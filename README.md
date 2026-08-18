[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Ansible](https://img.shields.io/badge/Ansible-2.9+-blue.svg)](https://www.ansible.com/)
[![1C](https://img.shields.io/badge/1C-8.3.27-green.svg)](https://1c.ru/)
[![Windows](https://img.shields.io/badge/Windows-WinRM-blue)](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html)
[![WSL2](https://img.shields.io/badge/WSL2-Ubuntu-orange)](https://learn.microsoft.com/ru-ru/windows/wsl/)

Ansible playbooks для автоматизации установки и обновления платформы 1С:Предприятие на Windows-хостах через WSL2.

## 📋 Содержание
- [Возможности](#-возможности)
- [Установка WSL2](#-Установка-WSL2)
- [Настройка Ansible](#-Установка-ansible)
- [Настройка WinRM](#-Настройка-WinRM)
- [Создание пользователя для Ansible](#Создание-пользователя-для-Ansible)
- [Настройка окружения](#-Настройка-окружения)

## 🚀 Возможности

- Полностью автоматизированная установка 1С на Windows хосты
- Работа через WSL2 (Windows Subsystem for Linux)
- Поддержка WinRM для подключения к Windows
- Копирование и распаковка архивов на целевых машинах
- Установка с тихими параметрами (/S /quiet /norestart)
- Проверка успешности установки через WMI и реестр
- Очистка временных файлов после установки

## 🔧 Установка WSL2

Откройте **PowerShell от имени Администратора** и выполните:
```
wsl --install
```
Важно! После установки перезагрузите компьютер
 
## 🔧 Установка Ansible

Обновление пакетов
```
sudo apt update
```
Установка Ansible
```
sudo apt install ansible -y
```
Проверка установки
```
ansible --version
```
## 🚀 Настройка WinRM

Установка pip и pywinrm
```
sudo apt install python3-pip -y
pip3 install pywinrm
```

Или через системные пакеты
```
sudo apt install python3-winrm -y
```
Этап 1: Подготовка
Запускаем PowerShell от имени Администратора. Сбросим настройки, чтобы избежать конфликтов со старыми конфигами.
Включение WinRM
```
Enable-PSRemoting -Force
```
# Создание пользователя для Ansible
```
СОЗДАЕМ НОВОГО ПОЛЬЗОВАТЕЛЯ
Write-Host "`nСоздаем пользователя AnsibleUser..." -ForegroundColor Cyan
$Password = Read-Host "Введите пароль для AnsibleUser" -AsSecureString
New-LocalUser -Name "AnsibleUser" -Password $Password -Description "Ansible automation user" -AccountNeverExpires -PasswordNeverExpires

ДОБАВЛЯЕМ В ГРУППЫ (Критически важно!)
Write-Host "Добавляем пользователя в группы..." -ForegroundColor Cyan

Группа администраторов (нужна для управления системой)
Add-LocalGroupMember -Group "Администраторы" -Member "AnsibleUser" -ErrorAction SilentlyContinue
# Группа удаленного управления (Remote Management Users)
Add-LocalGroupMember -Group "Пользователи удаленного управления" -Member "AnsibleUser" -ErrorAction SilentlyContinue

Проверяем, что пользователь добавлен
Write-Host "`nПроверка членства в группах:" -ForegroundColor Yellow
Get-LocalGroupMember -Group "Администраторы" | Where-Object {$_.Name -like "*AnsibleUser*"}
Get-LocalGroupMember -Group "Пользователи удаленного управления" | Where-Object {$_.Name -like "*AnsibleUser*"}
```
Этап 2: Настройка брандмауэра и прав доступа
```
НАСТРОЙКА БРАНДМАУЭРА
Write-Host "`nНастраиваем брандмауэр..." -ForegroundColor Cyan

Закрываем старый HTTP порт (5985) для надежности
Remove-NetFirewallRule -Name "WinRM HTTP" -ErrorAction SilentlyContinue

Открываем новый HTTPS порт (5986)
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Name "WinRM HTTPS" -Profile Any -LocalPort 5986 -Protocol TCP -Action Allow -ErrorAction SilentlyContinue

Проверяем, что правило создалось и включено
Get-NetFirewallRule -DisplayName "WinRM HTTPS" | Format-Table Name, Enabled, Direction, Action

НАСТРОЙКА АУТЕНТИФИКАЦИИ (Важно!)
Write-Host "`nНастраиваем параметры аутентификации WinRM..." -ForegroundColor Cyan

Запрещаем незашифрованные соединения (только HTTPS)
winrm set winrm/config/service '@{AllowUnencrypted="false"}'

Включаем Basic аутентификацию (часто используется Ansible)
winrm set winrm/config/service/auth '@{Basic="true"}'

Включаем Negotiate/NTLM (для совместимости)
winrm set winrm/config/service/auth '@{Negotiate="true"}'
winrm set winrm/config/service/auth '@{Kerberos="true"}'

ПЕРЕЗАПУСКАЕМ СЛУЖБУ ДЛЯ ПРИМЕНЕНИЯ ВСЕХ НАСТРОЕК
Restart-Service WinRM
Write-Host "Служба WinRM перезапущена." -ForegroundColor Green
```
Этап 3: Создание сертификата и HTTPS-слушателя
```
СОЗДАЕМ СЕРТИФИКАТ
Write-Host "Создаем самоподписанный сертификат..." -ForegroundColor Cyan
$cert = New-SelfSignedCertificate -DnsName $env:COMPUTERNAME -CertStoreLocation "Cert:\LocalMachine\My" -KeyLength 2048 -KeyAlgorithm "RSA" -HashAlgorithm "SHA256"

Сохраняем отпечаток (Thumbprint) в переменную, он нам пригодится
$thumbprint = $cert.Thumbprint

Write-Host "Сертификат создан:" -ForegroundColor Green
Write-Host "  Thumbprint (отпечаток): $thumbprint" -ForegroundColor White
Write-Host "  Subject: $($cert.Subject)"
Write-Host "  Истекает: $($cert.NotAfter)"

СОЗДАЕМ HTTPS СЛУШАТЕЛЬ
Write-Host "`nСоздаем HTTPS слушатель на порту 5986..." -ForegroundColor Cyan
winrm create winrm/config/listener?Address=*+Transport=HTTPS "@{Hostname=`"$env:COMPUTERNAME`"; CertificateThumbprint=`"$thumbprint`"}"

ПРОВЕРЯЕМ, ЧТО СЛУШАТЕЛЬ ПОЯВИЛСЯ
Write-Host "`nСписок активных слушателей WinRM:" -ForegroundColor Yellow
winrm enumerate winrm/config/listener
```
Полезные ссылки!
```
https://docs.ansible.com/projects/ansible/latest/os_guide/windows_winrm.html
```
```
https://gist.github.com/nikhilsingnurkar/9776116d44446a3f5da64d71cfafe57f#file-configureremotingforansible-ps1
```
## 📁 Настройка окружения

🎯 Создайте локальный `host.ini` на основе `host.example.ini`. Файл с реальными адресами исключён из Git, а пароль вводится интерактивно.
```ini
[windows_depo_new_pc]
windows_pc1 ansible_host=192.0.2.10
windows_pc2 ansible_host=192.0.2.11

[windows_depo_new_pc:vars]
ansible_user = AnsibleUser
ansible_connection = winrm
ansible_winrm_transport = basic
ansible_winrm_port = 5986
ansible_winrm_scheme = https
ansible_winrm_server_cert_validation = ignore
```
🔧 Настройка install_1c.yml

```
- name: Install 1C from archive
  hosts: windows_depo_new_pc
  gather_facts: true

  vars:
    # Путь к архиву 1С на вашем Linux-компьютере
    archive_path: "/home/admin/files/setuptc64_8_3_27_1859.zip"

    # Временная папка на Windows для распаковки
    temp_dir: "C:\\Temp\\1с_fixed_install"

  tasks:
    - name: Create temp directory on Windows
      ansible.windows.win_file:
        path: "{{ temp_dir }}"
        state: directory

    - name: Copy archive to Windows
      ansible.windows.win_copy:
        src: "{{ archive_path }}"
        dest: "{{ temp_dir }}\\setuptc64_8_3_27_1859.zip"
      when: archive_path is defined

    - name: Extract archive using PowerShell
      ansible.windows.win_shell: |
        # Создаем папку для распаковки
        $extractPath = "{{ temp_dir }}\\extracted"
        New-Item -ItemType Directory -Path $extractPath -Force

        # Распаковываем архив
        Add-Type -AssemblyName System.IO.Compression.FileSystem
        [System.IO.Compression.ZipFile]::ExtractToDirectory("{{ temp_dir }}\\setuptc64_8_3_27_1859.zip", $extractPath)

        Write-Output "Archive extracted to: $extractPath"
      register: extract_result

    - name: Show extract result
      debug:
        var: extract_result.stdout

    - name: Find setup files in extracted directory
      ansible.windows.win_find:
        paths: "{{ temp_dir }}\\extracted"
        patterns: "*.exe"
        file_type: file
        recurse: yes
      register: found_setup_files

    - name: Display found setup files
      debug:
        var: found_setup_files.files

    - name: Install 1C from found setup file
      block:
        - name: Run setup file
          ansible.windows.win_package:
            path: "{{ found_setup_files.files[0].path }}"
            arguments: "/S /quiet /norestart"
            state: present
          register: install_result

        - name: Show installation result
          debug:
            var: install_result
      when: found_setup_files.files | length > 0

    - name: Alternative installation method
      ansible.windows.win_shell: |
        $setupFile = "{{ found_setup_files.files[0].path }}"
        if (Test-Path $setupFile) {
            Write-Output "Installing from: $setupFile"
            $process = Start-Process -FilePath $setupFile -ArgumentList "/S", "/quiet", "/norestart" -PassThru -Wait
            Write-Output "Exit code: $($process.ExitCode)"
        } else {
            Write-Error "Setup file not found: $setupFile"
        }
      when: found_setup_files.files | length > 0
      register: alt_install_result

    - name: Verify 1C installation
      ansible.windows.win_shell: |
        # Проверяем установленные программы
        $1cProducts = Get-WmiObject -Class Win32_Product | Where-Object {$_.Name -like "*1C*"}
        if ($1cProducts) {
            $1cProducts | Format-List Name, Version
        } else {
            Write-Output "1C products not found in installed programs"

            # Альтернативная проверка через реестр
            $registryPaths = @(
                "HKLM:\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Uninstall\\*",
                "HKLM:\\SOFTWARE\\WOW6432Node\\Microsoft\\Windows\\CurrentVersion\\Uninstall\\*"
            )
            $installedSoftware = Get-ItemProperty $registryPaths -ErrorAction SilentlyContinue |
                                Where-Object {$_.DisplayName -like "*1C*"}
            if ($installedSoftware) {
                $installedSoftware | Select-Object DisplayName, DisplayVersion
            }
        }
      register: verify_installation

    - name: Show installation verification
      debug:
        var: verify_installation.stdout_lines

    - name: Clean up temp files (optional)
      ansible.windows.win_file:
        path: "{{ temp_dir }}"
        state: absent
      ignore_errors: yes
```

Ping
```
ansible -i host.ini windows_depo_new_pc -m win_ping --ask-pass
```
