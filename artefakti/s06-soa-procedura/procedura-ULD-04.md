# Процедура ULD-04 – SSH само с ключ

**Фирма:** HotelChain Sofia AD  
**Автор:** Виктория Ганчева  
**Версия и дата:** 0.2, 04.11.2026

| Поле | Стойност |
|---|---|
| Реализира контрол | ULD-04 – SSH на `srv` само с ключ, без root и без парола |
| Annex A | 8.5 (сигурна автентикация), свързан с 8.2 |
| R-ID | R-05 (A-10) – риск от отгатване на парола; процедурата защитава **SSH до A-12**, но не замества MFA за акаунтите в резервационната система |
| Изпълнява се от | Системен администратор |
| Честота | При първоначална настройка на `srv` и след промяна в SSH настройките |
| Състояние | Процедурата е написана; реалното изпълнение трябва да се потвърди в лабораторията |

## 1. Предусловия

1. Стартирани са `gw` и `srv`, като `srv` е с проверен IP адрес. В сценария той е `10.60.0.10`; **ако в реалната лаборатория адресът е друг, смени го във всички команди предварително**.
2. На `srv` работи `ssh`, а потребителят `student` има `sudo`.
3. От `gw` **вече има успешен вход с SSH ключ** към `student` на `srv`. Провери го с командата по-долу. Ако не работи, **спри**, без да забраняваш паролите.
4. Направи snapshot на `srv` във VirtualBox с име **`pre-h06`**.
5. Отвори **втора SSH сесия към `srv`** и я остави отворена до края. Тя е нужна за връщане на настройките, ако изгубиш достъп.
6. Доказателствата се събират първо в `~/h06/`, после се качват в `artefakti/s06-soa-procedura/`.

**Проверка от `gw`, преди промяната:**

```bash
ssh -o BatchMode=yes -o PreferredAuthentications=publickey student@10.60.0.10 whoami
```

**Очакван резултат:** `student`, без поискана парола, код 0. При отказ – спри.

## 2. Стъпки на `srv`

### Стъпка 1 – запиши настройките преди промяната

```bash
mkdir -p ~/h06 && cd ~/h06
sudo /usr/sbin/sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) ' > predi.txt
cat predi.txt
```

**Очакван резултат:** `predi.txt` съдържа четирите текущи настройки. При грешка – спри.

### Стъпка 2 – провери мястото за новия файл

```bash
sudo test -d /etc/ssh/sshd_config.d
sudo test ! -e /etc/ssh/sshd_config.d/10-uld04.conf
```

**Очакван резултат:** и двете команди приключват с код 0. Ако папката липсва или файлът вече съществува – спри; не презаписвай настройки.

### Стъпка 3 – добави настройките

```bash
printf '%s\n' 'PermitRootLogin no' 'PasswordAuthentication no' 'KbdInteractiveAuthentication no' 'PubkeyAuthentication yes' | sudo tee /etc/ssh/sshd_config.d/10-uld04.conf >/dev/null
```

**Очакван резултат:** файлът `10-uld04.conf` е създаден. Двете стари SSH сесии остават отворени.

### Стъпка 4 – провери синтаксиса и стойностите

```bash
sudo /usr/sbin/sshd -t
sudo /usr/sbin/sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
```

**Очакван резултат:** първата команда няма грешки, а стойностите са `permitrootlogin no`, `passwordauthentication no`, `kbdinteractiveauthentication no`, `pubkeyauthentication yes`. Ако е различно – не презареждай услугата, а изпълни връщането назад.

### Стъпка 5 – презареди SSH

```bash
sudo systemctl reload ssh
sudo systemctl is-active ssh
```

**Очакван резултат:** `active`. Не затваряй старите сесии, докато не минат тестовете.

### Стъпка 6 – запиши настройките след промяната

```bash
cd ~/h06
sudo /usr/sbin/sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) ' > sled.txt
cat sled.txt
```

**Очакван резултат:** `sled.txt` показва `no`, `no`, `no`, `yes` за четирите настройки.

## 3. Проверка от `gw`

**Положителен тест – входът с ключ трябва да работи (нова връзка):**

```bash
ssh -o BatchMode=yes -o PreferredAuthentications=publickey student@10.60.0.10 whoami
```

**Очакван резултат:** `student`, код 0.

**Отрицателен тест 1 – входът с парола трябва да е забранен:**

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o KbdInteractiveAuthentication=no -o NumberOfPasswordPrompts=1 student@10.60.0.10 true > otkaz.txt 2>&1
rc=$?; echo "password_rc=$rc" >> otkaz.txt
```

**Очакван резултат:** отказ, обикновено `Permission denied`, и код 255. **Ако SSH предложи да въведеш парола, тестът е неуспешен – не я въвеждай и върни настройката.**

**Отрицателен тест 2 – директен root вход трябва да е забранен:**

```bash
ssh -o BatchMode=yes -o PreferredAuthentications=publickey root@10.60.0.10 whoami >> otkaz.txt 2>&1
rc=$?; echo "root_rc=$rc" >> otkaz.txt
```

**Очакван резултат:** отказ и ненулев код; отделно в `sled.txt` трябва да има `permitrootlogin no`. Само отказът на root не доказва настройката, защото може да няма разрешен ключ.

## 4. При отклонение

При грешка на която и да е стъпка **спри**. Ако се загуби входът с ключ, някоя настройка не е `no/no/no/yes` или забраненият вход успее, върни старото състояние от втората SSH сесия. Уведоми Ръководителя ИТ до 15 минути и запиши случилото се до края на следващия работен ден.

## 5. Връщане назад (rollback)

**Изпълни от втората отворена сесия на `srv` или от конзолата на VirtualBox:**

```bash
sudo rm /etc/ssh/sshd_config.d/10-uld04.conf
sudo /usr/sbin/sshd -t && sudo systemctl reload ssh
sudo systemctl is-active ssh
```

**Очакван резултат:** `active` и възстановени предишни настройки. Ако SSH не може да се възстанови от сесията, върни snapshot **`pre-h06`** във VirtualBox. Не изтривай други `.conf` файлове.

## 6. Запис и доказателства

След тестовете на `srv` запази и последните редове от журнала:

```bash
sudo journalctl -u ssh --since '-30 minutes' --no-pager | tail -n 30 > ~/h06/ssh-journal.txt
```

Файлът `otkaz.txt` се получава на `gw`. След като входът с ключ е проверен, го копирай към `srv`:

```bash
scp otkaz.txt student@10.60.0.10:~/h06/otkaz.txt
```

Качи **само след реално изпълнение** `predi.txt`, `sled.txt`, `otkaz.txt`, `ssh-journal.txt`, тази процедура и кратък README с дата, изпълнител и резултат в `artefakti/s06-soa-procedura/`. Пази доказателствата до края на курса (най-малко 90 дни). Не качвай частни ключове или пароли.

## 7. Проверка от съученик

| Поле | Резултат |
|---|---|
| Изпълнил, дата | За попълване след проверка |
| Засякъл на стъпка | За попълване – номер или „не“ |
| Неясно | За попълване след проверка |
| Отрицателният тест даде | За попълване с реалния резултат |
| Поправено след проверката | За попълване след проверка |

**Предаване:** след тестовете добави файловете и изпълни `git add artefakti/s06-soa-procedura && git commit -m "h06: soa i procedura" && git push` от основната папка на Git хранилището.
