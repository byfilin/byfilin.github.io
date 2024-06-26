---
title: Про side-channel атаки
date: 2024-06-20 12:00:00 -500
categories: [basics]
tags: [cyber, side-channel]
image:
  path: /assets/img/articles/side-channel-main.jpeg
  alt: timing table
---

Вони зараз доволі часто зʼявляються у медіа. Наприклад вразливість процесорів M1 що дозволяє теоретично розшифрувати диск, чи те як аналітик виявив бекдор у XZ нашумівшому, або Spectre 2018 року.

Це тип атак які засновані на дослідженні процесів (в широкому сенсі) що виникають під час роботи алгоритму, та допомагають виявити як саме він працює. Зазвичай метою атаки є криптографічні ключи, але атака ними не обмежена.

Сигналами може бути будь що: таймінг, енергоспоживання, електромагнітні спайки, звук тд
Наприклад відновлення вашого пароля по звуку клавіш що ви натискаєте на клавіатурі (існує реальний PoC).

Хоча це все здається доволі низкорівневою атакою: процесор, енергія, але давайте розглянемо більш цікавий класичний приклад такої атаки яка стосується вебу. А саме response timing enumeration

Уявімо що в нас є auth механізм який при логіні перевіряє чи існує наданий користувач, якщо так - хешуємо наданий пароль та перевіряємо із тим що в нас в базі. Нам треба дізнатись чи існує користувач за допомогою форми логіну. 
Сам механізм буде вигляди так:

![img-description](/assets/img/articles/side-channel-1.jpeg)
_Auth scheme_

Враховуючи назву та свій досвід у розробці ви можете подумати: ок, і де тут вразливість? різниця у часі між обома варіантами мінімальна, а якщо ще зважити похибку яку надає мережевий канал то взагалі...
І будете праві, тож давайте поглянемо на наступну картинку власне атаки:

![img-description](/assets/img/articles/side-channel-2.jpeg)
_Attack scheme_

Зважаючи на складність та максимальну довжину пароля в цьому сценарії час на його хеш та перевірку порівняно із не правильним логіном вже буде більш значним та помітним. Звісно таке потребує перевірки враховуючи кількість факторів що впливають на результат. 

Також варіанти захисту ви і самі вже мабуть накидали в голові кілька. Але це все ще гарний приклад не дуже очевидної атаки що може викрити додаткову інформацію. Далі будуть посилання на те що згадував у треді, кому цікаво велком, всім іншим - гарного дня

Лабка із веб вразливостю щоб помацати руками
[Portswigger Lab](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)


Розбір вразливості процессорів М1
[Стаття на Arstechnica](https://arstechnica.com/security/2024/03/hackers-can-extract-secret-encryption-keys-from-apples-mac-chips/)