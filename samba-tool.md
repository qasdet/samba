### Расширенные техники тест-дизайна для samba-tool с подробными консольными примерами

#### 1. **Классы эквивалентности + Граничные значения (комбинированные)**
```bash
# Тестирование создания пользователя (длина имени)
samba-tool user create a --given-name=A --surname=B  # min-length (1)
samba-tool user create $(python -c "print('a'*255)") --given-name=Long --surname=User  # max-length (255)
samba-tool user create $(python -c "print('a'*256)") --given-name=TooLong --surname=User  # invalid (256)

# Тестирование парольной политики
samba-tool domain passwordsettings set --min-pwd-length=1  # min boundary
samba-tool domain passwordsettings set --min-pwd-length=127  # max boundary
samba-tool domain passwordsettings set --min-pwd-length=128  # invalid
```

#### 2. **Таблицы решений (комбинации флагов)**
```bash
# Комбинации для создания пользователя:
samba-tool user create test1 --use-username-as-cn --random-password --must-change-at-next-login
samba-tool user create test2 --use-username-as-cn --random-password --no-must-change-at-next-login
samba-tool user create test3 --random-password --must-change-at-next-login --userou="OU=Special"
samba-tool user create test4 --company="TestCorp" --department="QA" --mail-address="test@domain.local"
```

#### 3. **Диаграмма состояний (жизненный цикл объекта)**
```bash
# Полный цикл пользователя:
samba-tool user create state_user --random-password  # State: CREATED
samba-tool user disable state_user  # State: DISABLED
samba-tool user enable state_user  # State: ENABLED
samba-tool user setpassword state_user --newpassword="NewPass123!"  # State: PASSWORD_CHANGED
samba-tool user lock state_user  # State: LOCKED
samba-tool user unlock state_user  # State: UNLOCKED
samba-tool user delete state_user  # State: DELETED

# Проверка состояний после каждого шага:
samba-tool user show state_user | grep "User Account Control"
```

#### 4. **Попарное тестирование (pairwise) для сложных команд**
```bash
# Комбинации для samba-tool domain join:
samba-tool domain join example.com DC -Uadmin --dns-backend=SAMBA_INTERNAL --option="idmap_ldb:use rfc2307" 
samba-tool domain join example.com MEMBER -Uadmin --dns-backend=BIND9_DLZ --option="idmap_rid:use rfc2307"
samba-tool domain join example.com DC -Uadmin --dns-backend=NONE --option="idmap_ad:use rfc2307"
```

#### 5. **Отрицательное тестирование (ошибки и исключения)**
```bash
# Некорректные входные данные:
samba-tool user create "" --surname="Empty"  # пустое имя
samba-tool group add "invalid group name"  # запрещенные символы
samba-tool computer create "NotComputer$"  # отсутствует $ в конце

# Операции с несуществующими объектами:
samba-tool user delete non_existing_user
samba-tool group addmembers admins ghost_user

# Конфликт имен:
samba-tool user create duplicate --random-password
samba-tool user create duplicate --random-password  # повторное создание
```

#### 6. **Тестирование на основе моделей (DNS записи)**
```bash
# Модель жизненного цикла DNS-записи:
samba-tool dns add localhost example.com test A 192.168.1.100  # CREATE
samba-tool dns update localhost example.com test A 192.168.1.100 192.168.1.200  # UPDATE
samba-tool dns query localhost example.com test A  # VERIFY
samba-tool dns delete localhost example.com test A 192.168.1.200  # DELETE

# Проверка согласованности:
samba-tool dns query localhost example.com @ ALL
```

#### 7. **Конфигурационное тестирование (разные окружения)**
```bash
# С разными backends DNS:
samba-tool domain provision --realm=EXAMPLE.COM --domain=EXAMPLE --dns-backend=SAMBA_INTERNAL
samba-tool domain provision --realm=EXAMPLE.COM --domain=EXAMPLE --dns-backend=BIND9_DLZ

# С разными режимами аутентификации:
samba-tool user create auth_user --random-password --use-username-as-cn
samba-tool user create auth_user2 --random-password --use-username-as-cn --kerberos
```

#### 8. **Тестирование производительности (объемные операции)**
```bash
# Массовое создание объектов:
for i in {1..500}; do samba-tool user create user$i --random-password; done

# Замер времени операций:
time samba-tool user list > /dev/null
time samba-tool group list members "Domain Admins"
```

#### 9. **Исследовательское тестирование (неочевидные сценарии)**
```bash
# Восстановление после сбоя:
samba-tool drs showrepl  # проверка репликации
samba-tool dbcheck --cross-ncs --fix --yes  # восстановление БД

# Тестирование доверительных отношений:
samba-tool trust create TRUSTED_DOMAIN --type=external --direction=incoming
samba-tool trust validate TRUSTED_DOMAIN

# Миграция данных:
samba-tool user import users.csv
samba-tool group import groups.csv
```

#### 10. **Тестирование безопасности**
```bash
# Проверка чувствительных данных:
samba-tool user getpassword testuser  # без прав администратора
samba-tool domain exportkeytab --principal=administrator keytab.file

# Проверка привилегий:
sudo -u nobody samba-tool domain info
sudo -u nobody samba-tool user list
```

#### 11. **Тестирование совместимости**
```bash
# Взаимодействие с Windows tools:
ldapsearch -x -H ldap://dc.example.com -D "administrator@example.com" -W -b "dc=example,dc=com"

# Проверка Kerberos:
kinit administrator@EXAMPLE.COM
klist
```

#### 12. **Тестирование обновлений и миграции**
```bash
# Миграция с предыдущей версии:
samba-tool domain backup online --target=backup.tar
samba-tool domain backup restore backup.tar --newservername=NEWDC

# Обновление схемы:
samba-tool domain schemaupgrade --ldf-file=schemaExtension.ldf
```

### Рекомендации по выполнению тестов:
1. **Автоматизация проверок**:
```bash
samba-tool user create verify_user --random-password
if samba-tool user show verify_user | grep -q "User Account Control"; then
    echo "Test PASSED"
else
    echo "Test FAILED"
fi
```

2. **Параметризованное тестирование** (через циклы):
```bash
for OU in "Sales" "HR" "IT"; do
    samba-tool ou create "OU=$OU,DC=example,DC=com"
    samba-tool user create "user_$OU" --userou="OU=$OU,DC=example,DC=com"
done
```

3. **Валидация через смежные инструменты**:
```bash
samba-tool user create mixed_case_user --given-name=John --surname=Smith
ldbsearch -H /var/lib/samba/private/sam.ldb \
  -b "cn=users,dc=example,dc=com" \
  "(cn=mixed_case_user)" distinguishedName
```

4. **Тестирование отката изменений**:
```bash
# Создание снапшота
sudo btrfs subvolume snapshot /var/lib/samba /var/lib/samba_backup

# Выполнение теста
samba-tool domain passwordsettings set --history-length=10

# Восстановление
sudo rm -rf /var/lib/samba
sudo mv /var/lib/samba_backup /var/lib/samba
sudo systemctl restart samba
```

### Ключевые области для глубокого тестирования:
1. **Репликация**:
```bash
samba-tool drs replicate DC2 DC1 DC=example,DC=com --full-sync
samba-tool drs showrepl
```

2. **Групповые политики**:
```bash
samba-tool gpo create "Test Policy"
samba-tool gpo listall
samba-tool gpo apply "Test Policy" "OU=TestOU,DC=example,DC=com"
```

3. **DNS управление**:
```bash
samba-tool dns zonecreate example.com newzone
samba-tool dns add example.com newzone test A 192.168.1.1
samba-tool dns query example.com newzone test A
```

4. **Службы времени**:
```bash
samba-tool ntp set time 0x1234567890abcdef
samba-tool ntp get time
```

5. **Управление сайтами**:
```bash
samba-tool sites create SiteA
samba-tool sites subnet create 192.168.1.0/24 SiteA
samba-tool sites addsitekdc SiteA kdc1.example.com
```

Данные примеры покрывают основные аспекты тестирования samba-tool, фокусируясь на практическом применении техник тест-дизайна непосредственно в консольных командах.