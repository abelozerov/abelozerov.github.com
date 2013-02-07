---
layout: post
title: "Как я переходил с Blogger на Octopress"
date: 2012-08-26 22:02
comments: true
categories: octopress
---

{% img /images/2012/08/26/octopress.png Octopress %}

Сам по себе переход занял у меня ровно один день с периодическим отвлечением на развлечения. Для начала я решил создать виртуальную машину с Ubuntu и делать все на ней, т.к. когда увидел в инструкции по установке RVM под Windows "Установите Cygwin", понял, что быстрее будет поставить систему с нуля, чем пытаться все правильно заставить работать под Винду. Также виртуальная машина мне скоро пригодится в экспериментах с Python'ом, так что время в любом случае было потрачено не зря.
<!-- more -->

Все мероприятие можно разделить на следующие этапы

1. Установка environment'а для Octopress и самого Octopress.
Тут все просто, выполняем шаги, следуя иструкциям:
[http://octopress.org/docs/setup/](http://octopress.org/docs/setup/)
Пришлось немного повозиться с зависимостями, но гуглилось все очень быстро, так что проблем не возникло.

2. Настройка Octopress, написание hello world поста и генерация блога.
[http://octopress.org/docs/configuring/](http://octopress.org/docs/configuring/), 
[http://octopress.org/docs/blogging/](http://octopress.org/docs/blogging/)

3. Импорт записей из Blogger. Воспользовался готовым скриптом [https://gist.github.com/2928871](https://gist.github.com/2928871)

4. Использование GitHub в качестве хостинга, заливка на него, настройка DNS.
[http://octopress.org/docs/deploying/github/](http://octopress.org/docs/deploying/github/), 
[https://help.github.com/articles/setting-up-a-custom-domain-with-pages](https://help.github.com/articles/setting-up-a-custom-domain-wi-pages)

5. Настройка редиректов, для людей, пришедших с поисковых систем.
Т.к. постов у меня было мало, направил старый домен blog.abelozerov.com на свой сервер с IIS7 и сделал там Rewrite Map с явным указанием, какая страница на какую должна переходить. Выкладываю получившийся web.config, может кому пригодится. Обратите внимание на строку с ключом "/feeds/posts/default" - это редирект для RSS-ленты, сделано для того чтобы старые подписчики продолжали получать обновления по RSS. 

{% include_code 2012/08/26/web.config lang:xml %}
